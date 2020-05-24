# 用std::unique_ptr实现pimpl手法必须在.cpp文件中提供析构函数定义

[pimpl手法](https://en.cppreference.com/w/cpp/language/pimpl) 就是把数据成员提取到类中，用指向该类的指针替代原来的数据成员。因为数据成员会影响内存布局，将数据成员用一个指针替代可以减少编译期依赖，保持 ABI 兼容。

比如对如下类：

```
//@ A.h
#include <string>
#include <vector>

class A {
    int i;
    std::string s;
    std::vector<double> v;
};
```

使用 [pimpl手法](https://en.cppreference.com/w/cpp/language/pimpl) 后：

```
//@ A.h
class A {
public:
    A();
    ~A();
private:
    struct X;
    X* x;
};

//@ A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(new X) {}
A::~A() { delete x; }
```

现在使用 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 替代原始指针，不再需要使用析构函数释放指针：

```
//@ A.h
#include <memory>

class A {
public:
    A();
private:
    struct X;
    std::unique_ptr<X> x;
};

//@ A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_unique<X>()) {}
```

但调用上述代码会出错：

```
//@ main.cpp
#include "A.h"

int main()
{
	A a; //@ 错误：A::X是不完整类型
}
```

原因在于 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 析构时会在内部调用默认删除器，默认删除器的 `delete` 语句之前会用 [static_assert](https://en.cppreference.com/w/cpp/language/static_assert) 断言指针指向的不是非完整类型。

```
//@ 删除器的实现
template<class T>
struct default_delete //@ default deleter for unique_ptr
{
    constexpr default_delete() noexcept = default;

    template<class U, enable_if_t<is_convertible_v<U*, T*>, int> = 0>
    default_delete(const default_delete<U>&) noexcept
    { //@ construct from another default_delete
    }

    void operator()(T* p) const noexcept
    {
        static_assert(0 < sizeof(T), "can't delete an incomplete type");
        delete p;
    }
};
```

解决方法就是让析构 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 的代码看见完整类型，即让析构函数的定义位于要析构的类型的定义之后。

```
//@ A.h
#include <memory>

class A {
public:
    A();
    ~A();
private:
    struct X;
    std::unique_ptr<X> x;
};

//@ A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_unique<X>()) {}
A::~A() = default; //@ 必须位于A::X的定义之后
```

- 使用 [pimpl手法](https://en.cppreference.com/w/cpp/language/pimpl) 的类自然应该支持移动操作，但定义析构函数会阻止默认生成移动操作，因此会想到添加默认的移动操作声明。

```
//@ A.h
#include <memory>

class A {
public:
    A();
    ~A();
    A(A&&) = default;
    A& operator=(A&&) = default;
private:
    struct X;
    std::unique_ptr<X> x;
};

//@ A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_unique<X>()) {}
A::~A() = default; //@ 必须位于A::X的定义之后
```

但调用移动操作会出现相同的问题。

```
//@ main.cpp
#include "A.h"

int main()
{
    A a;
    A b(std::move(a)); //@ 错误：使用了未定义类型A::X
    A c = std::move(a); //@ 错误：使用了未定义类型A::X
}
```

原因也一样，移动操作会先析构原有对象，调用删除器时触发断言。解决方法也一样，让移动操作的定义位于要析构的类型的定义之后。

```
//@ A.h
#include <memory>

class A {
public:
    A();
    ~A();
    A(A&&);
    A& operator=(A&&);
private:
    struct X;
    std::unique_ptr<X> x;
};

//@ A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_unique<X>()) {}
A::A(A&&) = default;
A& A::operator=(A&&) = default;
A::~A() = default;
```

编译器不会为 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 这类 `move-only` 类型生成拷贝操作，即使可以生成也只是拷贝指针本身（浅拷贝），因此如果要提供拷贝操作，则需要自己编写。

```
//@ A.h
#include <memory>

class A {
public:
    A();
    ~A();
    A(A&&);
    A& operator=(A&&);
    A(const A&);
    A& operator=(const A&);
private:
    struct X;
    std::unique_ptr<X> x;
};

//@ A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_unique<X>()) {}
A::A(A&&) = default;
A& A::operator=(A&&) = default;
A::~A() = default;
A::A(const A & rhs) : x(std::make_unique<X>(*rhs.x)) {}
A& A::operator=(const A & rhs)
{
    *x = *rhs.x;
    return*this;
}
```

如果使用 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)，则不需要关心上述所有问题

```
//@ A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_shared<X>()) {}
```

实现 [pimpl手法](https://en.cppreference.com/w/cpp/language/pimpl) 时：

- [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)尺寸更小，运行更快一些，但必须在实现文件中指定特殊成员函数，[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 开销大一些，但不需要考虑因为删除器引发的一系列问题。
- 但对于 [pimpl手法](https://en.cppreference.com/w/cpp/language/pimpl) 来说，主类和数据成员类之间是专属所有权的关系，[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 更合适。如果在需要共享所有权的特殊情况下，[std::shared_ptr ](https://en.cppreference.com/w/cpp/memory/shared_ptr) 更合适。







