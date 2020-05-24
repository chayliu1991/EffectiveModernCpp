# auto

`auto` 类型推断几乎和模板类型推断一致。

调用模板时，编译器根据 `expr` 推断 `T` 和 `ParamType` 的类型。当变量用 `auto` 声明时，`auto` 就扮演了模板中的 `T` 的角色，变量的类型修饰符则扮演 `ParamType` 的角色。

为了推断变量类型，编译器表现得好比每个声明对应一个模板，模板的调用就相当于对应的初始化表达式。

```
auto x = 1;
const auto cx = x;
const auto& rx = x;

template<typename T> //@ 用来推断x类型的概念上假想的模板
void func_for_x(T x);

func_for_x(1); //@ 假想的调用: param的推断类型就是x的类型

template<typename T> //@ 用来推断cx类型的概念上假想的模板
void func_for_cx(const T x);

func_for_cx(x); //@ 假想的调用: param的推断类型就是cx的类型

template<typename T> //@ 用来推断rx类型的概念上假想的模板
void func_for_rx(const T& x);

func_for_rx(x); //@ 假想的调用: param的推断类型就是rx的类型
```

`auto` 的推断适用模板推断机制的三种情形：`T&`、`T&&` 和 `T`：

```
auto x = 1; //@ int x
const auto cx = x; //@ const int cx
const auto& rx = x; //@ const int& rx
auto&& uref1 = x; //@ int& uref1
auto&& uref2 = cx; //@ const int& uref2
auto&& uref3 = 1; //@ int&& uref3
```

`auto` 对数组和指针的推断也和模板一致：

```
const char name[] = "downdemo"; //@ 数组类型是const char[9]
auto arr1 = name; //@ const char* arr1
auto& arr2 = name; //@ const char (&arr2)[9]

void g(int, double); //@ 函数类型是void(int, double)
auto f1 = g; //@ void (*f1)(int, double)
auto& f2 = g; //@ void (&f2)(int, double)
```

`auto` 推断唯一不同于模板实参推断的情形是 C++11 的初始化列表。下面是同样的赋值功能：

```
//@ C++98
int x1 = 1;
int x2(1);
//@ C++11
int x3 = { 1 };
int x4{ 1 };
```

但换成 `auto` 声明，这些赋值的意义就不一样了：

```
auto x1 = 1; //@ int x1
auto x2(1); //@ int x2
auto x3 = { 1 }; //@ std::initializer_list<int> x3
auto x4{ 1 }; //@ C++11为std::initializer_list<int> x4，C++14为int x4
```

如果初始化列表中元素类型不同，则无法推断：

```
auto x5 = { 1, 2, 3.0 }; //@ 错误：不能为std::initializer_list<T>推断T
```

C++14 禁止对 `auto` 用 [std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list) 直接初始化，而必须用 `=`，除非列表中只有一个元素，这时不会将其视为[std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)。

```
auto x1 = { 1, 2 }; //@ C++14中必须用=，否则报错
auto x2 { 1 }; //@ 允许单元素的直接初始化，不会将其视为initializer_list
```

模板不支持模板参数为 `T` 而 `expr` 为初始化列表的推断，不会将其假设为 [std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)，这就是 `auto` 推断和模板推断唯一的不同之处。

```
auto x = { 1, 2, 3 }; //@ x类型是std::initializer_list<int>

template<typename T> //@ 等价于x声明的模板
void f(T x);

f({ 1, 2, 3 }); //@ 错误：不能推断T的类型
```

不过将模板参数为 [std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list) 则可以推断 `T`。

```
template<typename T>
void f(std::initializer_list<T> initList);

f({ 11, 23, 9 }); //@ T被推断为int，initList类型是std::initializer_list<int>
```

对于 C++11，`auto` 的介绍就到此为止了。

## C++14中的auto

C++14中，`auto` 可以作为函数返回类型，并且 `lambda` 可以将参数声明为 `auto`，这种 `lambda` 称为泛型 `lambda`。

```
auto f() { return 1; }
auto g = [](auto x) { return x; };
```

但此时 `auto` 仍然使用的是模板实参推断的机制，因此不能为 `auto` 返回类型返回一个初始化列表，即使是单元素：

```
auto newInitList() { return { 1 }; } //@ 错误
```

泛型 `lambda` 同理：

```
std::vector<int> v { 2, 4, 6 };
auto resetV = [&v](const auto& newValue) { v = newValue; };
resetV({ 1, 2, 3 }); //@ 错误
```

## C++17中的auto

C++17 中，`auto` 可以作为非类型模板参数：

```
template<auto N>
struct X {
    void f() { std::cout << N; }
};

X<1> x;
x.f(); //@ 1
```













