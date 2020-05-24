# 对右值引用使用std::move，对转发引用使用std::forward

右值引用只会绑定到可移动对象上，因此应该使用 [std::move](https://en.cppreference.com/w/cpp/utility/move)。转发引用用右值初始化时才是右值引用，因此应当使用 [std::forward](https://en.cppreference.com/w/cpp/utility/forward) 。

```
class A {
public:
    A(A&& rhs) : s(std::move(rhs.s)), p(std::move(rhs.p)) {}

    template<typename T>
    void f(T&& x)
    {
        s = std::forward<T>(x);
    }
private:
    std::string s;
    std::shared_ptr<int> p;
};
```

如果希望只有在移动构造函数保证不抛异常时才能转为右值，则可以用 [std::move_if_noexcept](https://en.cppreference.com/w/cpp/utility/move_if_noexcept) 替代 [std::move](https://en.cppreference.com/w/cpp/utility/move)。

```
class A {
public:
	A() = default;
	A(const A&) { std::cout << 1 << "\n"; }
	A(A&&) { std::cout << 2 << "\n"; }
};

class B {
public:
	B() {}
	B(const B&) noexcept { std::cout << 3 << "\n"; }
	B(B&&) noexcept { std::cout << 4 << "\n"; }
};

int main()
{
	A a;
	A a2 = std::move_if_noexcept(a); //@ 1
	B b;
	B b2 = std::move_if_noexcept(b); //@ 4
}
```

如果返回对象传入时是右值引用或转发引用，在返回时要用 [std::move](https://en.cppreference.com/w/cpp/utility/move) 或 [std::forward](https://en.cppreference.com/w/cpp/utility/forward) 转换。返回类型不需要声明为引用，按值传递即可。

```
A f(A&& a)
{
    doSomething(a);
    return std::move(a);
}


template<typename T>
A g(T&& x)
{
    doSomething(x);
    return std::forward<T>(x);
}
```

返回局部变量时，不需要使用 [std::move](https://en.cppreference.com/w/cpp/utility/move) 来优化。

```
A makeA()
{
    A a;
    return std::move(a); //@ 画蛇添足
}
```

局部变量会直接创建在为返回值分配的内存上，从而避免拷贝，这是 C++ 标准诞生时就有的 [RVO（return value optimization）](https://en.cppreference.com/w/cpp/language/copy_elision)。RVO 的要求十分严谨，它要求局部对象类型与返回值类型相同，且返回的就是局部对象本身，而使用了[std::move](https://en.cppreference.com/w/cpp/utility/move) 反而不满足 RVO 的要求。此外 RVO 只是种优化，编译器可以选择不采用，但标准规定，即使编译器不省略拷贝，返回对象也会被作为右值处理，所以 [std::move](https://www.ibm.com/developerworks/community/blogs/5894415f-be62-4bc0-81c5-3956e82276f3/entry/RVO_V_S_std_move?lang=en) 是多余的。

```
A makeA()
{
    return A{};
}

auto x = makeA(); //@ 只需要调用一次A的默认构造函数
```



























