# decltype

`decltype` 会推断出直觉预期的类型：

```
const int i = 0; //@ decltype(i)为const int

struct Point {
    int x, y; //@ decltype(Point::x)和decltype(Point::y)为int
};

A a; //@ decltype(a)为A
bool f(const A& x); //@ decltype(x)为const A&，decltype(f)为bool(const A&)
if (f(a)) … //@ decltype(f(a))为bool

int a[] {1, 2, 3}; //@ decltype(a)为int[3]
```

`decltype` 一般用来声明与参数类型相关的返回类型。比如下面模板的参数是容器和索引，而返回类型取决于元素类型：

```
template<typename Container, typename Index>
auto f(Container& c, Index i) -> decltype(c[i])
{ //@ auto不做任何事，只是表示使用类型推断，推断使用的是decltype
    return c[i];
}
```

C++14 允许省略尾置返回类型，只留下 `auto`：

```
template<typename Container, typename Index>
auto f(Container& c, Index i)
{
    return c[i];
}
```

但直接使用会发现问题：

```
std::vector<int> v;
…
f(v, 5) = 10; //@ 返回v[5]然后赋值为10，但不能通过编译
```

`operator[]` 返回元素引用，类型为 `int&`，但 `auto` 推断为 `int`，因此上面的操作相当于给一个整型值赋值，显然是错误的。

为了得到期望的返回类型，需要对返回类型使用 `decltype` 的推断机制，C++14 允许将返回类型声明为` decltype(auto) `来实现这点。

```
template<typename Container, typename Index>
decltype(auto) f(Container& c, Index i)
{
    return c[i];
}
```

`decltype(auto)` 也可以作为变量声明类型：

```
int i = 1;
const int& j = i;
decltype(auto) x = j; //@ const int& x = j;
```

但还有一些问题，容器传的是 `non-const` 左值引用，这就无法接受右值：

```
std::vector<int> makeV(); //@ 工厂函数
auto i = f(makeV(), 5);
```

为了同时匹配左值和右值而又不想重载，只需要模板参数写为转发引用：

```
template<typename Container, typename Index>
decltype(auto) f(Container&& c, Index i)
{
    return std::forward<Container>(c)[i]; //@ 传入的实参是右值时，std::forward将c转为右值
}

//@ C++11版本
template<typename Container, typename Index>
auto f(Container&& c, Index i) -> decltype(std::forward<Container>(c)[i])
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

### decltype的特殊情况

如果表达式是解引用，`decltype` 会推断为引用类型：

```
int* p; //@ decltype(*p)是int&
```

赋值表达式会产生引用，类型为赋值表达式中左值的引用类型：

```
int a = 0;
int b = 1;
decltype(a=1) c = b; //@ int&
c = 3;
std::cout << a << b << c; //@ 033
```

如果表达式加上一层或多层括号，编译器会将其看作表达式，变量是一种可以作为赋值语句左值的特殊表达式，因此也得到引用类型。`decltype((variable))`  结果永远是引用，`declytpe(variable)` 只有当变量本身是引用时才是引用。

```
int i; //@ decltype((i))是int&
```

在返回类型为 `decltype(auto)` 时，这可能导致返回局部变量的引用：

```
decltype(auto) f1()
{
    int x = 0;
    …
    return x; //@ decltype(x)是int，因此返回int
}

decltype(auto) f2()
{
    int x = 0;
    …
    return (x); //@ decltype((x))是int&，因此返回了局部变量的引用
}
```











