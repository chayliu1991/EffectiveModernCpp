# 用noexcept标记不抛异常的函数

C++98 中，必须指出一个函数可能抛出的所有异常类型，如果函数有所改动则 [exception specification](https://en.cppreference.com/w/cpp/language/except_spec) 也要修改，而这可能破坏代码，因为调用者可能依赖于原本的 [exception specification](https://en.cppreference.com/w/cpp/language/except_spec)，所以 C++98 中的 [exception specification](https://en.cppreference.com/w/cpp/language/except_spec) 被认为不值得使用。

C++11 中达成了一个共识，真正需要关心的是函数会不会抛出异常。一个函数要么可能抛出异常，要么绝对不抛异常，这种 maybe-or-never 形成了 C++11 [exception specification](https://en.cppreference.com/w/cpp/language/except_spec) 的基础，C++98 的 [exception specification](https://en.cppreference.com/w/cpp/language/except_spec) 在 C++17 移除。

函数是否要加上 `noexcept` 声明与接口设计相关，调用者可以查询函数的 `noexcept` 状态，查询结果将影响代码的异常安全性和执行效率。因此函数是否要声明为 `noexcept` 就和成员函数是否要声明为 `const` 一样重要，如果一个函数不抛异常却不为其声明 `noexcept`，这就是接口规范缺陷。

`noexcept` 的一个额外优点是，它可以让编译器生成更好的目标代码。为了理解原因只需要考虑 C++98 和 C++11 表达函数不抛异常的区别。

```
int f(int x) throw(); //@ C++98
int f(int x) noexcept; //@ C++11
```

如果一个异常在运行期逃出函数，则 [exception specification](https://en.cppreference.com/w/cpp/language/except_spec) 被违反。在 C++98 中，调用栈会展开到函数调用者，执行一些无关的动作后中止程序。C++11 的一个微小区别是是，在程序中止前只是可能展开栈。这一点微小的区别将对代码生成造成巨大的影响。

`noexcept` 声明的函数中，如果异常传出函数，优化器不需要保持栈在运行期的展开状态，也不需要在异常逃出时，保证其中所有的对象按构造顺序的逆序析构。而声明为 `throw()` 的函数就没有这样的优化灵活性。总结起来就是：

```
RetType function(params) noexcept; //@ most optimizable
RetType function(params) throw(); //@ less optimizable
RetType function(params); //@ less optimizable
```

这个理由已经足够支持给任何已知不会抛异常的函数加上 noexcept，比如移动操作就是典型的不抛异常函数。

[std::vector::push_back](https://en.cppreference.com/w/cpp/container/vector/push_back) 在容器空间不够容纳元素时，会扩展新的内存块，再把元素转移到新的内存块。C++98 的做法是逐个拷贝，然后析构旧内存的对象，这使得 [push_back](https://en.cppreference.com/w/cpp/container/vector/push_back) 提供强异常安全保证：如果拷贝元素的过程中抛出异常，则[std::vector](https://en.cppreference.com/w/cpp/container/vector) 保持原样，因为旧内存元素还未被析构。

[std::vector::push_back](https://en.cppreference.com/w/cpp/container/vector/push_back) 在 C++11 中的优化是把拷贝替换成移动，但为了不违反强异常安全保证，只有确保元素的移动操作不抛异常时才会用移动替代拷贝。

`swap` 函数是需要 `noexcept` 声明的另一个例子，不过标准库的 `swap` 用 [noexcept](https://en.cppreference.com/w/cpp/language/noexcept) 操作符的结果决定。

```
//@ 数组的swap
template <class T, size_t N>
void swap(T(&a)[N], T(&b)[N]) noexcept(noexcept(swap(*a, *b))); //@ 由元素类型决定noexcept结果
//@ 比如元素类型是class A，如果swap(A, A)不抛异常则该数组的swap也不抛异常

// std::pair的swap
template <class T1, class T2>
struct pair {
    …
        void swap(pair& p) noexcept(noexcept(swap(first, p.first)) &&
            noexcept(swap(second, p.second)));
    …
};
```

虽然 `noexcept` 有优化的好处，但将函数声明为 `noexcept` 的前提是，保证函数长期具有 noexcept 性质，如果之后随意移除 noexcept 声明，就有破坏客户代码的风险。

大多数函数是异常中立的，它们本身不抛异常，但它们调用的函数可能抛异常，这样它们就允许抛出的异常传到调用栈的更深一层，因此异常中立函数天生永远不具备 `noexcept` 性质。

如果为了强行加上 `noexcept` 而修改实现就是本末倒置，比如调用一个会抛异常的函数是最简单的实现，为了不抛异常而环环相扣地来隐藏这点（比如捕获所有异常，将其替换成状态码或特殊返回值），大大增加了理解和维护的难度，并且这些复杂性的时间成本可能超过 `noexcept` 带来的优化。

对某些函数来说，`noexcept` 性质十分重要，内存释放函数和所有的析构函数都隐式 `noexcept`，这样就不必加 `noexcept `声明。析构函数唯一未隐式 `noexcept`  的情况是，类中有数据成员的类型显式将析构函数声明 `noexcept(false)`。但这样的析构函数很少见，标准库中一个也没有。

有些库的接口设计者会把函数区分为 `wide contract` 和 `narrow contract`：

- `wide contract`  函数没有前置条件，不用关心程序状态，对传入的实参没有限制，一定不会有未定义行为，如果知道不会抛异常就可以加上 `noexcept`。
- `narrow contract` 函数有前置条件，如果条件被违反则结果未定义。但函数没有义务校验这个前置条件，它断言前置条件一定满足（调用者负责保证断言成立），因此加上 `noexcept` 声明也是合理的。

```
//@ 假设前置条件是s.length(//@ <= 32
void f(const std::string& s)//@ noexcept;
```

但如果想在违反前置条件时抛出异常，由于函数的 `noexcept` 声明，异常就会导致程序中止，因此一般只为 `wide contract` 函数声明 `noexcept`

在 `noexcept` 函数中调用可能抛异常的函数时，编译器不会帮忙给出警告：

```
void start();
void finish();
void f() noexcept
{
    start();
    … //@ do the actual work
    finish();
}
```

带 `noexcept` 声明的函数调用了不带 `noexcept` 声明的函数，这看起来自相矛盾，但也许被调用的函数在文档中写明了不会抛异常，也许它们来自 C 语言的库，也许来自还没来得及根据 C++11 标准做修订的 C++98 库。































