# 用std::mutex或std::atomic保证const成员函数线程安全

假设有一个表示多项式的类，它包含一个返回根的 `const` 成员函数：

```
class Polynomial {
public:
    std::vector<double> roots() const
    { //@ 实际仍需要修改值，所以将要修改的成员声明为mutable
        if (!rootsAreValid)
        {
            … //@ 计算根
            rootsAreValid = true;
        }
        return rootVals;
    }
private:
    mutable bool rootsAreValid{ false };
    mutable std::vector<double> rootVals{};
};
```

假如此时有两个线程对同一个对象调用成员函数，虽然函数声明为 const，但由于函数内部修改了数据成员，就可能产生数据竞争。最简单的解决方法是引入一个 [std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)：

```
class Polynomial {
public:
    std::vector<double> roots() const
    {
        std::lock_guard<std::mutex> l(m);
        if (!rootsAreValid)
        {
            … //@ 计算根
            rootsAreValid = true;
        }
        return rootVals;
    }
private:
    mutable std::mutex m; //@ std::mutex是move-only类型，因此这个类只能移动不能拷贝
    mutable bool rootsAreValid{ false };
    mutable std::vector<double> rootVals{};
};
```

对一些简单的情况，使用原子变量 [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic) 可能开销更低（取决于机器及 [std::mutex](https://en.cppreference.com/w/cpp/thread/mutex) 的实现）：

```
class Point {
public:
    double distanceFromOrigin() const noexcept
    {
        ++callCount; //@ 计算调用次数
        return std::sqrt((x * x) + (y * y));
    }
private:
    mutable std::atomic<unsigned> callCount{ 0 }; //@ std::atomic也是move-only类型
    double x, y;
};
```

因为 [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic) 的开销比较低，很容易想当然地用多个原子变量来同步：

```
class A {
public:
    int f() const
    {
        if (flag) return res;
        else
        {
            auto x = expensiveComputation1();
            auto y = expensiveComputation2();
            res = x + y;
            flag = true; //@ 设置标记
            return res;
        }
    }
private:
    mutable std::atomic<bool> flag{ false };
    mutable std::atomic<int> res;
};
```

这样做可行，但如果多个线程同时观察到标记值为 `false`，每个线程都要继续进行运算，这个标记反而没起到作用。先设置标记再计算可以消除这个问题，但会引起一个更大的问题：

````
class A {
public:
    int f() const
    {
        if (flag) return res;
        else
        {
            flag = true; //@ 在计算前设置标记值为true
            auto x = expensiveComputation1();
            auto y = expensiveComputation2();
            res = x + y;
            return res;
        }
    }
private:
    mutable std::atomic<bool> flag{ false };
    mutable std::atomic<int> res;
};
````

假如线程1刚设置好标记，线程2此时正好检查到标记值为 `true` 并直接返回数据值，然后线程1接着计算结果，这样线程2的返回值就是错的。

因此如果要同步多个变量或内存区，最好还是使用 [std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)。

```
class A {
public:
    int f() const
    {
        std::lock_guard<std::mutex> l(m);
        if (flag) return res;
        else
        {
            auto x = expensiveComputation1();
            auto y = expensiveComputation2();
            res = x + y;
            flag = true;
            return res;
        }
    }
private:
    mutable std::mutex m;
    mutable bool flag{ false };
    mutable int res;
};
```











