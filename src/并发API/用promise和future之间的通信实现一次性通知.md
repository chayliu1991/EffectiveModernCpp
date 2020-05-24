# 用std::promise和std::future之间的通信实现一次性通知

让一个任务通知另一个异步任务发生了特定事件，一种实现方法是使用条件变量：

```
std::condition_variable cv;
std::mutex m;
bool flag(false);
std::string s("hello");

void f()
{
    std::unique_lock<std::mutex> l(m);
    cv.wait(l, [] { return flag; }); //@ lambda返回false则阻塞，并在收到通知后重新检测
    std::cout << s; //@ 若返回true则继续执行
}

int main()
{
    std::thread t(f);
    {
        std::lock_guard<std::mutex> l(m);
        s += " world";
        flag = true;
        cv.notify_one(); //@ 发出通知
    }
    t.join();
}
```

另一种方法是用 [std::promise::set_value](https://en.cppreference.com/w/cpp/thread/promise/set_value) 通知 [std::future::wait](https://en.cppreference.com/w/cpp/thread/future/wait)：

```
std::promise<void> p;

void f()
{
    p.get_future().wait(); //@ 阻塞至p.set_value
    std::cout << 1;
}

int main()
{
    std::thread t(f);
    p.set_value(); //@ 解除阻塞
    t.join();
}
```

这种方法非常简单，但也有缺点，[std::promise](https://en.cppreference.com/w/cpp/thread/promise) 和 [std::future](https://en.cppreference.com/w/cpp/thread/future) 之间的 shared state 是动态分配的，存在堆上的分配和回收成本。更重要的是，[std::promise](https://en.cppreference.com/w/cpp/thread/promise) 只能设置一次，因此它和 [std::future](https://en.cppreference.com/w/cpp/thread/future) 的之间的通信只能使用一次，而条件变量可以重复通知。因此这种方法一般用来创建暂停状态的 [std::thread](https://en.cppreference.com/w/cpp/thread/thread)：

```
std::promise<void> p;

void f()
{
    std::cout << 1;
}

int main()
{
    std::thread t([] { p.get_future().wait(); f(); });
    p.set_value();
    t.join();
}
```

此时可能会想到使用 RAII 版的 [std::thread](https://en.cppreference.com/w/cpp/thread/thread)，但这并不安全：

```
int main()
{
    A t(
        std::thread([] { p.get_future().wait(); f(); }),
        A::DtorAction::join
    );
    ... //@ 如果此处抛异常，则set_value不会被调用，wait将永远不返回
    //@ 而RAII会在析构时调用join，join将一直等待线程完成，但wait使线程永不完成
    //@ 因此如果此处抛出异常，析构函数永远不会完成，程序将失去效应
    p.set_value();
}
```

[std::condition_variable::notify_all](https://en.cppreference.com/w/cpp/thread/condition_variable/notify_all) 可以一次通知多个任务，这也可以通过 [std::promise](https://en.cppreference.com/w/cpp/thread/promise) 和 [std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future) 之间的通信实现：

```
std::promise<void> p;

void f(int x)
{
    std::cout << x;
}

int main()
{
    std::vector<std::thread> v;
    auto sf = p.get_future().share();
    for(int i = 0; i < 10; ++i) v.emplace_back([sf, i] { sf.wait(); f(i); });
    p.set_value();
    for(auto& x : v) x.join();
}
```







