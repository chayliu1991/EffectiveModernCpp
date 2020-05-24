# 用constexpr表示编译期常量

`constexpr` 用于对象时就是一个加强版的 `const`，表面上看 `constexpr` 表示值是 `const`，且在编译期（严格来说是翻译期，包括编译和链接，如果不是编译器或链接器作者，无需关心这点区别）已知，但用于函数则有不同的意义。

编译期已知的值可能被放进只读内存，这对嵌入式开发是一个很重要的语法特性。

 ```
int i = 42;
constexpr auto j = i; //@ 错误：i的值在编译期未知
std::array<int, i> v1; //@ 错误：同上
constexpr auto n = 10; //@ OK：10是一个编译期常量
std::array<int, n> v2; //@ OK：n的值是在编译期已知
 ```

`constexpr` 函数在调用时若传入的是编译期常量，则产出编译期常量，传入运行期才知道的值，则产出运行期值。`constexpr` 函数可以满足所有需求，因此不必为了有非编译期值的情况而写两个函数。

```
constexpr int pow(int base, int exp) noexcept
{
    … //@ 实现见后
}

constexpr auto n = 5;
std::array<int, pow(3, n)> results; //@ pow(3, n)在编译期计算出结果
```

上面的 `constexpr` 并不表示函数要返回 `const `值，而是表示，如果参数都是编译期常量，则返回结果就可以当编译期常量使用，如果有一个不是编译期常量，返回值就在运行期计算。

```
auto base = 3; //@ 运行期获取值
auto exp = 10; //@ 运行期获取值
auto baseToExp = pow(base, exp); //@ pow在运行期被调用
```

C++11 中，`constexpr` 函数只能包含一条语句，即一条 `return` 语句。有两个应对限制的技巧：用条件运算符 ` ?: ` 替代 `if-else`、用递归替代循环。

```
constexpr int pow(int base, int exp) noexcept
{
    return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
```

C++14 解除了此限制：

```
//@ C++14
constexpr int pow(int base, int exp) noexcept
{
    auto result = 1;
    for (int i = 0; i < exp; ++i) result *= base;
    return result;
}
```

`constexpr` 函数必须传入和返回 [literal type](https://en.cppreference.com/w/cpp/named_req/LiteralType)。`constexpr` 构造函数可以让自定义类型也成为 [literal type](https://en.cppreference.com/w/cpp/named_req/LiteralType) 。

```
class Point {
public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept
    : x(xVal), y(yVal) {}
    constexpr double xValue() const noexcept { return x; }
    constexpr double yValue() const noexcept { return y; }
    void setX(double newX) noexcept { x = newX; } //@ 修改了对象所以不能声明为constexpr
    void setY(double newY) noexcept { y = newY; } //@ 另外C++11中constexpr函数返回类型不能是void
private:
    double x, y;
};

constexpr Point p1(9.4, 27.7); //@ 编译期执行constexpr构造函数
constexpr Point p2(28.8, 5.3); //@ 同上

//@ 通过constexpr Point对象调用xValue和yValue也会在编译期获取值
//@ 于是可以再写出一个新的constexpr函数
constexpr Point midpoint(const Point& p1, const Point& p2) noexcept
{
    return { (p1.xValue() + p2.xValue()) / 2, (p1.yValue() + p2.yValue()) / 2 };
}
constexpr auto mid = midpoint(p1, p2); //@ mid在编译期创建
```

因为 `mid` 是编译期已知值，这就意味着如下表达式可以用于模板形参：

```
mid.xValue()*10
//@ 因为上式是浮点型，浮点型不能用于模板实例化，因此还要如下转换一次
static_cast<int>(mid.xValue()*10)
```

C++14 允许对值进行了修改或无返回值的函数声明为 `constexpr`：

```
//@ C++14
class Point {
public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept
    : x(xVal), y(yVal) {}
    constexpr double xValue() const noexcept { return x; }
    constexpr double yValue() const noexcept { return y; }
    constexpr void setX(double newX) noexcept { x = newX; }
    constexpr void setY(double newY) noexcept { y = newY; }
private:
    double x, y;
};

//@ 于是C++14允许写出下面的代码
constexpr Point reflection(const Point& p) noexcept //@ 返回p关于原点的对称点
{
    Point res;
    res.setX(-p.xValue());
    res.setY(-p.yValue());
    return res;
}

constexpr Point p1(9.4, 27.7);
constexpr Point p2(28.8, 5.3);
constexpr auto mid = midpoint(p1, p2);
constexpr auto reflectedMid = reflection(mid); //@ 值为(-19.1, -16.5)，且在编译期已知
```

使用 `constexpr` 的前提是必须长期保证需要它，因为如果后续要删除 `constexpr` 可能会导致许多错误。

