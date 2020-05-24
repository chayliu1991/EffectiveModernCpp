# 用std::shared_ptr管理所有权可共享的资源

[std::shared_ptr ](https://en.cppreference.com/w/cpp/memory/shared_ptr) 内部有一个引用计数，用来存储资源被共享的次数。因为内部多了一个指向引用计数的指针，所以 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 的尺寸是原始指针的两倍。

```
int* p = new int(42);
auto q = std::make_shared<int>(42);
std::cout << sizeof(p) <<" "<< sizeof(q); //@ 8 16
```

[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 保证线程安全，因此引用计数的递增和递减是原子操作，原子操作一般比非原子操作慢。

[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 默认析构方式和 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 一样，也是 delete 内部的原始指针，同样可以自定义删除器，不过不需要在模板参数中指明删除器类型。

```
class A {};
auto f = [](A* p) { delete p; };

std::unique_ptr<A, decltype(f)> p(new A, f);
std::shared_ptr<A> q(new A, f);
```

模板参数不使用删除器的设计在使用上带来了一些弹性。

```
std::shared_ptr<A> p(new A, f);
std::shared_ptr<A> q(new A, g);
//@ 使用不同的删除器但具有相同的类型，因此可以放进同一容器
std::vector<std::shared_ptr<A>> v{ p, q };
```

删除器不影响 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 的尺寸，因为删除器不是 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 的一部分，而是位于堆上或自定义分配器的内存位置。[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 有一个 `control block`，它包含了引用计数的指针和自定义删除器的拷贝，以及一些其他数据（比如弱引用计数）。

![](../img/shared_ptr.png)



[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 内部实现如下：

```
template<typename T>
struct sp_element {
    using type = T;
};

template<typename T>
struct sp_element<T[]> {
    using type = T;
};

template<typename T, std::size_t N>
struct sp_element<T[N]> {
    using type = T;
};

template<typename T>
class shared_ptr {
    using elem_type = typename sp_element<T>::type;
    elem_type* px; //@ 内部指针
    shared_count pn; //@ 引用计数
    template<typename U> friend class shared_ptr;
    template<typename U> friend class weak_ptr;
};

class shared_count {
    sp_counted_base* pi;
    int shared_count_id;
    friend class weak_count;
};

class weak_count {
    sp_counted_base* pi;
};

class sp_counted_base {
    int use_count; //@ 引用计数
    int weak_count; //@ 弱引用计数
};

template<typename T>
class sp_counted_impl_p: public sp_counted_base {
    T* px; //@ 删除器
};
```

`control block` 在创建第一个 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 时确定，因此 `control block` 的创建发生在如下时机：

- 调用 [std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared) 时：调用时生成一个新对象，此时显然不会有关于该对象的 `control block`。
- 从 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 构造 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 时：因为 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 没有 `control block`。
- 用原始指针构造 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 时。

这意味着用同一个原始指针构造多个 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)，将创建多个 control block，即有多个引用指针，当引用指针变为零时就会出现多次析构的错误。

```
int main()
{
    {
        int* i = new int(42);
        std::shared_ptr<int> p(i);
        std::shared_ptr<int> q(i);
    } //@ 错误
}
```

使用 [std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared) 就不会有这个问题。

```
auto p = std::make_shared<int>(42);
```

但 [std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared) 不支持自定义删除器，这时应该直接传递 new 的结果。

```
auto f = [](int*) {};
std::shared_ptr<int> p(new int(42), f);
```

用类的 `this` 指针构造 [std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared) 时，`*this` 的所有权不会被共享。

```
class A {
public:
    std::shared_ptr<A> f() { return std::shared_ptr<A>(this); }
};
auto p = std::make_shared<A>();
auto q = p->f();
std::cout << p.use_count() << q.use_count(); //@ 11
```

为了解决这个问题，需要继承 [std::enable_shared_from_this](https://en.cppreference.com/w/cpp/memory/enable_shared_from_this)，通过其提供的 [shared_from_this](https://en.cppreference.com/w/cpp/memory/enable_shared_from_this/shared_from_this) 获取 `*this` 的所有权。

```
class A : public std::enable_shared_from_this<A> {
public:
    std::shared_ptr<A> f() { return shared_from_this(); }
};

auto p = std::make_shared<A>();
auto q = p->f();
std::cout << p.use_count() << q.use_count(); //@ 22
```

[shared_from_this ](https://en.cppreference.com/w/cpp/memory/enable_shared_from_this/shared_from_this) 的原理是为 `*this` 的 `control block` 创建一个新的 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)，因此 `*this` 必须有一个已关联的 `control block`，即有一个指向 ` *this`  的 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)，否则行为未定义，抛出 [std::bad_weak_ptr](https://en.cppreference.com/w/cpp/memory/bad_weak_ptr) 异常。

```
class A : public std::enable_shared_from_this<A> {
public:
    std::shared_ptr<A> f() { return shared_from_this(); }
};

auto p = new A;
auto q = p->f(); //@ 抛出std::bad_weak_ptr异常
```

为了只允许创建用 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 指向的对象，可以将构造函数放进 private 作用域，并提供一个返回[std::shared_ptr ](https://en.cppreference.com/w/cpp/memory/shared_ptr)对象的工厂函数：

```
class A : public std::enable_shared_from_this<A> {
public:
    static std::shared_ptr<A> create() { return std::shared_ptr<A>(new A); }
    std::shared_ptr<A> f() { return shared_from_this(); }
private:
    A() = default;
};

auto p = A::create(); //@ 构造函数为private，auto p = new A将报错
auto q = p->f(); //@ OK
```































