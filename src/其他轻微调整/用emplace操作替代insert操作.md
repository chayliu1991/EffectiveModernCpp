# 用emplace操作替代insert操作

[std::vector::push_back](https://en.cppreference.com/w/cpp/container/vector/push_back) 对左值和右值的重载为：

```
template<class T, class Allocator = allocator<T>>
class vector {
public:
    void push_back(const T& x);
    void push_back(T&& x);
};
```

直接传入字面值时，会创建一个临时对象。使用 [std::vector::emplace_back](https://en.cppreference.com/w/cpp/container/vector/emplace_back) 则可以直接用传入的实参调用元素的构造函数，而不会创建任何临时对象。

```
class A {
public:
    A(int i) { std::cout << 1; }
    A(const A&) { std::cout << 2; }
    A(A&&) { std::cout << 3; }
    ~A() { std::cout << 4; }
};

std::vector<A> v;
v.push_back(1); //@ 134
//@ 先创建一个临时对象右值A(1)
//@ 随后将临时对象传入push_back，需要一次移动（不支持则拷贝）
//@ 最后析构临时对象

v.emplace_back(1); //@ 1：直接构造
```

所有 `insert` 操作都有对应的 `emplace` 操作：

```
push_back => emplace_back //@ std::list、std::deque、std::vector
push_front => emplace_front //@ std::list、std::deque、std::forward_list
insert_after => emplace_after //@ std::forward_list
insert => emplace //@ 除std::forward_list、std::array外的所有容器
insert => try_emplace//@ std:map、std::unordered_map
emplace_hint //@ 所有关联容器
```

即使 `insert` 函数不需要创建临时对象，也可以用 `emplace` 函数替代，此时两者本质上做的是同样的事。因此 `emplace ` 函数就能做到 `insert` 函数能做的所有事，有时甚至做得更好：

```
std::vector<std::string> v;
std::string s("hi");
//@ 下面两个调用的效果相同
v.push_back(s);
v.emplace_back(s);
```

`emplace` 不一定比 `insert`  快。之前 `emplace` 添加元素到容器末尾，该位置不存在对象，因此新值会使用构造方式。但如果添加值到已有对象占据的位置，则会采用赋值的方式，于是必须创建一个临时对象作为移动的源对象，此时 `emplace` 并不会比 `insert` 高效：

```
std::vector<std::string> v { "hhh", "iii" };
v.emplace(v.begin(), "hi"); //@ 创建一个临时对象后移动赋值
```

对于 [std::set](https://en.cppreference.com/w/cpp/container/set) 和 [std::map](https://en.cppreference.com/w/cpp/container/map)，为了检查值是否已存在，`emplace` 会为新值创建一个 `node`，以便能与容器中已存在的 `node` 进行比较。如果值不存在，则将 `node` 链接到容器中。如果值已存在，`emplace` 就会中止，`node` 会被析构，这意味着构造和析构的成本被浪费了，此时 `emplace` 不如 `insert` 高效。

```
#include <chrono>
#include <functional>
#include <iostream>
#include <set>
#include <iomanip>

class A {
    int a, b, c;
public:
    A(int _a, int _b, int _c) : a(_a), b(_b), c(_c) {}
    bool operator<(const A &other) const
    {
        if (a < other.a)  return true;
        if (a == other.a && b < other.b)  return true;
        return (a == other.a && b == other.b && c < other.c);
    }
};

constexpr int n = 100;

void set_emplace() {
    std::set<A> set;
    for (int i = 0; i < n; ++i)
        for (int j = 0; j < n; ++j)
            for (int k = 0; k < n; ++k) set.emplace(i, j, k);
}

void set_insert() {
    std::set<A> set;
    for (int i = 0; i < n; ++i)
        for (int j = 0; j < n; ++j)
            for (int k = 0; k < n; ++k) set.insert(A(i, j, k));
}

void test(std::function<void()> f)
{
    auto start = std::chrono::system_clock::now();
    f();
    auto stop = std::chrono::system_clock::now();
    std::chrono::duration<double, std::milli> time = stop - start;
    std::cout << std::fixed << std::setprecision(2) << time.count() << " ms" << '\n';
}

int main()
{
    test(set_insert);  //@ 7640.89 ms
    test(set_emplace); //@ 7662.36 ms
    test(set_insert);  //@ 7575.74 ms
    test(set_emplace); //@ 7661.48 ms
    test(set_insert);  //@ 7575.80 ms
    test(set_emplace); //@ 7664.50 ms
}
```

创建临时对象并非总是坏事。假设给一个存储 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 的容器添加一个自定义删除器的 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 对象：

```
std::list<std::shared_ptr<A>> v;

void f(A*);
v.push_back(std::shared_ptr<A>(new A, f));
//@ 或者如下，意义相同
v.push_back({ new A, f });
```

如果使用 [emplace_back](https://en.cppreference.com/w/cpp/container/list/emplace_back) 会禁止创建临时对象。但这里临时对象带来的收益远超其成本。考虑如下可能发生的事件序列：

- 创建一个 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 临时对象
- [push_back](https://en.cppreference.com/w/cpp/container/list/push_back) 以引用方式接受临时对象，分配内存时抛出了内存不足的异常
- 异常传到 [push_back](https://en.cppreference.com/w/cpp/container/list/push_back) 之外，临时对象被析构，于是删除器被调用，A 被释放

即使发生异常，也没有资源泄露。[push_back](https://en.cppreference.com/w/cpp/container/list/push_back) 的调用中，由 `new` 构造的 A 会在临时对象被析构时释放。如果使用的是[emplace_back](https://en.cppreference.com/w/cpp/container/list/emplace_back)，`new` 创建的原始指针被完美转发到 [emplace_back](https://en.cppreference.com/w/cpp/container/list/emplace_back) 分配内存的执行点。如果内存分配失败，抛出内存不足的异常，异常传到 [emplace_back](https://en.cppreference.com/w/cpp/container/list/emplace_back) 外，唯一可以获取堆上对象的原始指针丢失，于是就产生了资源泄漏。

实际上不应该把 `new A` 这样的表达式直接传递给函数，应该单独用一条语句来创建 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 再将对象作为右值传递给函数：

```
std::shared_ptr<A> p(new A, f);
v.push_back(std::move(p));
//@ emplace_back的写法相同，此时两者开销区别不大
v.emplace_back(std::move(p));
```

`emplace` 函数在调用 [explicit](https://en.cppreference.com/w/cpp/language/explicit) 构造函数时存在一个隐患：

```
std::vector<std::regex> v;
v.push_back(nullptr); //@ 编译出错
v.emplace_back(nullptr); //@ 能通过编译，运行时抛出异常，难以发现此问题
```

原因在于 [std::regex](https://en.cppreference.com/w/cpp/regex/basic_regex) 接受 `const char*` 参数的构造函数被声明为 [explicit](https://en.cppreference.com/w/cpp/language/explicit)，用 [nullptr](https://en.cppreference.com/w/cpp/language/nullptr) 赋值要求 [nullptr](https://en.cppreference.com/w/cpp/language/nullptr) 到 [std::regex](https://en.cppreference.com/w/cpp/regex/basic_regex) 的隐式转换，因此不能通过编译：

```
std::regex r = nullptr; //@ 错误
```

而 [emplace_back](https://en.cppreference.com/w/cpp/container/vector/emplace_back) 直接传递实参给构造函数，这个行为在编译器看来等同于：

```
std::regex r(nullptr); //@ 能编译但会引发异常，未定义行为
```











