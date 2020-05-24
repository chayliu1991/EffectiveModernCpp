# 用std::make_unique（std::make_shared）创建std::unique_ptr（std::shared_ptr）

C++14 提供了 [std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)，C++11 可以手动实现一个基础功能版：

```
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
	return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```

这个基础函数不支持数组和自定义删除器，但这些不难实现。从这个基础函数可以看出，`make` 函数把实参完美转发给构造函数并返回构造出的智能指针。除了 [std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared) 和 [std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)，还有一个 `make` 函数是[std::allocate_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/allocate_shared)，它的行为和 [std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared) 一样，只不过第一个实参是分配器对象。

优先使用 `make` 函数的一个明显原因就是只需要写一次类型：

 ```
auto p = std::make_unique<int>(42);
std::unique_ptr<int> q(new int(42);
 ```

另一个原因与异常安全相关:

```
void f(std::shared_ptr<A> p, int n) {}
int g() { return 1; }
f(std::shared_ptr<A>(new A), g()); //@ 潜在的内存泄露隐患
//@ g可能运行于new A还未返回给std::shared_ptr的构造函数时
//@ 此时如果g抛出异常，则new A就发生了内存泄漏
f(std::make_shared<A>(), g()); //@ 不会发生内存泄漏，且只需要一次内存分配
```

`make` 函数有两个限制，一是它无法定义删除器：

```
auto f = [] (A* p) { delete p; };
std::unique_ptr<A, decltype(f)> p(new A, f);
std::shared_ptr<A> q(new A, f);
```

使用自定义删除器，但又想避免内存泄漏，解决方法是单独用一条语句来创建 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)：

```
auto d = [] (A* p) { delete p; };
std::shared_ptr<A> p(new A, d); //@ 如果发生异常，删除器将析构new创建的对象
f(std::move(p), g());
```

`make` 函数的第二个限制是，`make` 函数中的完美转发使用的是小括号初始化，在持有 [std::vector](https://en.cppreference.com/w/cpp/container/vector) 类型时，设置初始化值不如大括号初始化方便。一个不算直接的解决方法是，先构造一个 [std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list) 再传入。

```
auto p = std::make_unique<std::vector<int>>(3, 6); // vector中是3个6
auto q = std::make_shared<std::vector<int>>(3, 6); // vector中是3个6

auto x = { 1, 2, 3, 4, 5, 6 };
auto p2 = std::make_unique<std::vector<int>>(x);
auto q2 = std::make_shared<std::vector<int>>(x);
```

[std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique) 只存在这两个限制，但 [std::make_shared ](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)和 [std::allocate_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/allocate_shared) 还有两个限制：

- 如果类重载了[operator new](https://en.cppreference.com/w/cpp/memory/new/operator_new) 和 [operator delete](https://en.cppreference.com/w/cpp/memory/new/operator_delete)，其针对的内存尺寸一般为类的尺寸，而 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 还要加上 `control block` 的尺寸，因此 [std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared) 不适用重载了 [operator new](https://en.cppreference.com/w/cpp/memory/new/operator_new) 和 [operator delete](https://en.cppreference.com/w/cpp/memory/new/operator_delete) 的类。
- [std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)使[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)的 `control block` 和管理的对象在同一内存上分配（比用new构造智能指针在尺寸和速度上更优的原因），对象在引用计数为0时被析构，但其占用的内存直到 `control block` 被析构时才被释放，比如 [std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr) 会持续指向 `control block`（为了检查引用计数以检查自身是否失效），`control block` 直到最后一个 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 和 [std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr) 被析构时才释放

假如对象尺寸很大，且最后一个 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 和 [std::weak_ptr ](https://en.cppreference.com/w/cpp/memory/weak_ptr)析构之间的时间间隔不能忽略，就会产生对象析构和内存释放之间的延迟。

```
auto p = std::make_shared<ReallyBigType>();
… //@ 创建指向该对象的多个std::shared_ptr和std::weak_ptr并做一些操作
… //@ 最后一个std::shared_ptr被析构，但std::weak_ptr仍存在
… //@ 此时，大尺寸对象占用内存仍未被回收
… //@ 最后一个std::weak_ptr被析构，control block和对象占用的同一内存块被释放
```

如果 `new` 构造 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)，最后一个[std::shared_ptr ](https://en.cppreference.com/w/cpp/memory/shared_ptr) 被析构时，内存就能立即被释放。

```
std::shared_ptr<ReallyBigType> p(new ReallyBigType);
… //@ 创建指向该对象的多个std::shared_ptr和std::weak_ptr并做一些操作
… //@ 最后一个std::shared_ptr被析构，std::weak_ptr仍存在，但ReallyBigType占用的内存立即被释放
… //@ 此时，仅control block内存处于分配而未回收状态
… //@ 最后一个std::weak_ptr被析构，control block的内存块被释放
```



