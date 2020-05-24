# 用std::unique_ptr管理所有权唯一的资源

使用智能指针时一般首选 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)，默认情况下它和原始指针尺寸相同。

[std::unique_ptr ](https://en.cppreference.com/w/cpp/memory/unique_ptr) 对资源拥有唯一所有权，因此它是 move-obly 类型，不允许拷贝。它常用作工厂函数的返回类型，这样工厂函数生成的对象在需要销毁时会被自动析构，而不需要手动析构

```
class A {};

std::unique_ptr<A> makeA()
{
    return std::unique_ptr<A>(new A);
}

auto p = makeA();
```

[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 的析构默认通过 `delete` 内部的原始指针完成，但也可以自定义删除器，删除器需要一个[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 内部指针类型的参数。

```
class A {};

auto f = [](A* p) { std::cout << "destroy\n"; delete p; };

std::unique_ptr<A, decltype(f)> makeA()
{
    std::unique_ptr<A, decltype(f)> p(new A, f);
    return p;
}
```

使用 C++14 的 `auto` 返回类型，可以将删除器的 `lambda` 定义在工厂函数内，封装性更好一些。

```
class A {};

auto makeA()
{
    auto f = [](A* p) { std::cout << "destroy\n"; delete p; };
    std::unique_ptr<A, decltype(f)> p(new A, f);
    return p;
}
```

可以进一步扩展成支持继承体系的工厂函数：

```
class A {
public:
    virtual ~A() {} //@ 删除器对任何对象调用的是基类的析构函数，因此必须声明为虚函数
};
class B : public A {}; //@ 基类的析构函数为虚函数，则派生类的析构函数默认为虚函数
class C : public A {};
class D : public A {};

auto makeA(int i)
{
    auto f = [](A* p) { std::cout << "destroy\n"; delete p; };
    std::unique_ptr<A, decltype(f)> p(nullptr, f);
    if (i == 1) p.reset(new B);
    else if (i == 2) p.reset(new C);
    else p.reset(new D);
    return p;
}
```

默认情况下，[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 和原始指针尺寸相同，如果自定义删除器则 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 会加上删除器的尺寸。一般无状态的函数对象（如无捕获的 `lambda`）不会浪费任何内存，作为删除器可以节约空间。

```
class A {};

auto f = [](A* p) { delete p; };
void g(A* p) { delete p; }
struct X {
    void operator()(A* p) const { delete p; }
};
std::unique_ptr<A> p1(new A);
std::unique_ptr<A, decltype(f)> p2(new A, f);
std::unique_ptr<A, decltype(g)*> p3(new A, g);
std::unique_ptr<A, decltype(X())> p4(new A, X());

//@ 机器为64位
std::cout << sizeof(p1) //@ 8：默认尺寸，即一个原始指针的尺寸
	<< sizeof(p2) //@ 8：无捕获lambda不会浪费尺寸
	<< sizeof(p3) //@ 16：函数指针占一个原始指针尺寸
	<< sizeof(p4); //@ 8：无状态的函数对象。但如果X中存储了状态（如数据成员、虚函数）就会增加尺寸
```

[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 作为返回类型的另一个方便之处是，可以转为 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 。

```
//@ std::make_unique 的返回类型是std::unique_ptr
std::shared_ptr<int> p = std::make_unique<int>(42); //@ OK
```

[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 针对数组提供了一个特化版本，此版本提供 [operator[]](https://en.cppreference.com/w/cpp/memory/unique_ptr/operator_at)，但不提供单元素版本的 `[operator*]`和 `[operator->]`，这样对其指向的对象就不存在二义性。

```
std::unique_ptr<int[]> p(new int[3]{11, 22, 33});
for(int i = 0; i < 3; ++i) std::cout << p[i];
```









