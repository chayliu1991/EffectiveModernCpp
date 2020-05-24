# 用enum class替代enum

一般在大括号中声明的名称，只在大括号的作用域内可见，但这对  `enum` 成员例外。`enum` 成员属于` enum` 所在的作用域，因此作用域内不能出现同名实例。

```
enum X { a, b, c };
int a = 1; //@ 错误：a已在作用域内声明过
```

C++11 引入了限定作用域的枚举类型，用 `enum class` 关键字表示：

```
enum class X { a, b, c };
int a = 1; //@ OK
X x = X::a; //@ OK
X y = b; //@ 错误
```

`enum class` 的另一个优势是不会进行隐式转换：

```
enum X { a, b, c };
X x = a;
if (x < 3.14) ... //@ 不应该将枚举与浮点数进行比较，但这里合法

enum class Y { a, b, c };
Y y = Y::a;
if (x < 3.14) ... //@ 报错：不允许比较
//@ 但enum class允许强制转换为其他类型
if (static_cast<double>(x) < 3.14) ... //@ OK
```

C++11 之前的 `enum` 不允许前置声明，而 C++11 的 `enum` 和 `enum class` 都可以前置声明：

 ```
enum Color; //@ C++11之前错误
enum class X; //@ OK
 ```

C++11 之前不能前置声明 `enum` 的原因是编译器为了节省内存，要在 `enum`  被使用前选择一个足够容纳成员取值的最小整型作为底层类型。

```
enum X { a, b, c }; //@ 编译器选择底层类型为char
enum Status { //@ 编译器选择比char更大的底层类型
    good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    indeterminate = 0xFFFFFFFF
};
```

不能前置声明的一个弊端是，由于编译依赖关系，在 `enum` 中仅仅添加一个成员可能就要重新编译整个系统。如果在头文件中包含前置声明，修改 `enum class` 的定义时就不需要重新编译整个系统，如果 `enum class` 的修改不影响函数的行为，则函数的实现也不需要重新编译。

C++11 支持前置声明的原因很简单，底层类型是已知的，用 [std::underlying_type](https://en.cppreference.com/w/cpp/types/underlying_type) 即可获取。也可以指定枚举的底层类型，如果不指定，`enum class` 默认为 ` int`，`enum ` 则不存在默认类型。

```
enum class X : std::uint32_t;
//@ 也可以在定义中指定
enum class Y : std::uint32_t { a, b, c };
```

C++11 中使用 `enum` 更方便的场景只有一种，即希望用到 `enum` 的隐式转换时：

```
enum X { name, age, number };
auto t = std::make_tuple("downdemo", 6, "13312345678");
auto x = std::get<name>(t); //@ get的模板参数类型是std::size_t，name可隐式转换为std::size_t
```

如果用 `enum class`，则需要强制转换：

```
enum class X { name, age, number };
auto t = std::make_tuple("downdemo", 6, "13312345678");
auto x = std::get<static_cast<std::size_t>(X::name)>(t);
```

可以用一个函数来封装转换的过程，但也不会简化多少：

```
template<typename E>
constexpr auto f(E e) noexcept
{
	return static_cast<std::underlying_type_t<E>>(e);
}

auto x = std::get<f(X::name)>(t);
```

















