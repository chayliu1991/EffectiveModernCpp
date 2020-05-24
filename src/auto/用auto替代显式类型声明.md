# 用auto替代显式类型声明

`auto` 声明的变量必须初始化，因此使用 `auto` 可以避免忘记初始化的问题。

```
int a; //@ 潜在的未初始化风险
auto b; //@ 错误：必须初始化
```

对于名称非常长的类型，如迭代器相关的类型，用 `auto` 声明可以大大简化工作：

```
template<typename It>
void f(It b, It e)
{
    while (b != e)
    {
        auto currentValue = *b;
        //@ typename std::iterator_traits<It>::value_type currentValue = *b;
        ...
    }
}
```

`lambda` 生成的闭包类型是编译期内部的匿名类型，无法得知，使用 `auto` 推断就没有这个问题：

```
auto f = [](auto& x, auto& y) { return x < y; };
```

如果不使用 `auto`，可以改用[std::function](https://en.cppreference.com/w/cpp/utility/functional/function)：

```
//@ std::function的模板参数中不能使用auto
std::function<bool(int&, int&)> f = [](auto& x, auto& y) { return x < y; };
```

除了明显的语法冗长和不能利用 `auto` 参数的缺点，[std::function](https://en.cppreference.com/w/cpp/utility/functional/function) 与 `auto` 的最大区别在于，`auto` 和闭包类型一致，内存量和闭包相同，而 [std::function](https://en.cppreference.com/w/cpp/utility/functional/function) 是类模板，它的实例有一个固定大小，这个大小不一定能容纳闭包，于是会分配堆上的内存以存储闭包，导致比 `auto` 变量占用更多内存。此外，编译器一般会限制内联，[std::function ](https://en.cppreference.com/w/cpp/utility/functional/function)调用闭包会比 `auto`慢。

`auto` 可以避免简写类型存在的潜在问题。比如如下代码有潜在隐患：

```
std::vector<int> v;
unsigned sz = v.size(); //@ v.size()类型实际为std::vector<int>::size_type
//@ 在32位机器上std::vector<int>::size_type与unsigned尺寸相同
//@ 但在64位机器上，std::vector<int>::size_type是64位，而unsigned是32位
```

如下代码也有潜在问题：

```
std::unordered_map<std::string, int> m; //@ m的元素类型实际是std::pair<const std::string, int>
for (const std::pair<std::string, int>& p : m) ... //@ 类型不一致，仍要转换，期间要构造大量临时对象
```

如果显式类型声明能让代码更清晰或有其他好处就不用强行 `auto`，此外 IDE 的类型提示也能缓解不能直接看出对象类型的问题。