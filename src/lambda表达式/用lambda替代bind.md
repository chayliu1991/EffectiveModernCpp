# 用lambda替代std::bind
`lambda` 代码更简洁：
```
auto f = [l, r] (const auto& x) { return l <= x && x <= r; };
```

//@ 用std::bind实现相同效果
```
using namespace std::placeholders;
//@ C++14
auto f = std::bind(
    std::logical_and<>(),
    std::bind(std::less_equal<>(), l, _1),
    std::bind(std::less_equal<>(), _1, r));
//@ C++11
auto f = std::bind(
    std::logical_and<bool>(),
    std::bind(std::less_equal<int>(), l, _1),
    std::bind(std::less_equal<int>(), _1, r));
```

`lambda` 可以指定值捕获和引用捕获，而 [std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind) 总会按值拷贝实参，要按引用传递则需要使用 [std::ref](https://en.cppreference.com/w/cpp/utility/functional/ref)。

```
void f(const A&);

using namespace std::placeholders;
A a;
auto g = std::bind(f, std::ref(a), _1);
```
- `lambda` 中可以正常使用重载函数，而 [std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind) 无法区分重载版本，为此必须指定对应的函数指针类型。
- `lambda` 闭包类的 `operator()` 采用的是能被编译器内联的常规的函数调用，而 [std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind) 采用的是一般不会被内联的函数指针调用，这意味着 `lambda` 比 [std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind) 运行得更快。

```
auto g1 = [] { f(1); }; //@ OK
auto g2 = std::bind(f, 1); //@ 错误
auto g3 = std::bind(static_cast<void(*)(int)>(f), 1); //@ OK
```
实参绑定的是 [std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind) 返回的对象，而非内部的函数：
```
void f(std::chrono::steady_clock::time_point t, int i)
{
	std::this_thread::sleep_until(t);
	std::cout << i << "\n";
}

auto g = [](int i)
{
	f(std::chrono::steady_clock::now() + std::chrono::seconds(3), i);
};

g(1); //@ 3秒后打印1

//@ 用std::bind实现相同效果，但存在问题
auto h = std::bind(
	f,
	std::chrono::steady_clock::now() + std::chrono::seconds(3),
	std::placeholders::_1);

h(1); //@ 3秒后打印1，但3秒指的是调用std::bind后的3秒，而非调用f后的3秒
```
上述代码的问题在于，计算时间的表达式作为实参被传递给 [std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)，因此计算发生在调用 [std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind) 的时刻，而非调用其绑定的函数的时刻。解决办法是延迟到调用绑定的函数时再计算表达式值，这可以通过在内部再嵌套一个 [std::bind ](https://en.cppreference.com/w/cpp/utility/functional/bind)来实现。
```
auto h = std::bind(
    f,
    std::bind(std::plus<>(), std::chrono::steady_clock::now(), std::chrono::seconds(3)),
    std::placeholders::_1);

```
C++14 中没有需要使用[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)的理由，C++11 由于特性受限存在两个使用场景：

- 一是模拟 C++11 缺少的移动捕获。
- 二是函数对象的 `operator()` 是模板时，若要将此函数对象作为参数使用，用 [std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind) 绑定才能接受任意类型实参。
```
struct X {
    template<typename T>
    void operator()(const T&) const;
};
X x;
auto f = std::bind(x, _1); //@ f可以接受任意参数类型
```
C++11 的 `lambda` 无法达成上述效果，但 C++14 可以:
```
X a;
auto f = [a](const auto& x) { a(x); };
```
## std::bind用法示例
- 占位符
```
void f(int a, int b, int c) { std::cout << a << b << c << "\n"; }
using namespace std::placeholders;
auto x = std::bind(f, _2, _1, 3);
//@ _n表示f的第n个参数
//@ x(a, b, c)相当于f(b, a, 3);

x(4, 5, 6); //@ 543
```
- 传引用
```
void f(int& n) { ++n; }

int n = 1;
auto x = std::bind(f, n);
x(); //@ n == 1
auto y = std::bind(f, std::ref(n));
y(); //@ n == 2
```
- 传递占位符给其他函数
```
void f(int a, int b) { std::cout << a << b << "\n"; }
int g(int n) { return n + 1; }

using namespace std::placeholders;
auto x = std::bind(f, _1, std::bind(g, _1));

x(1); //@ 12
```
- 绑定成员函数指针
```
struct A {
	void f(int n) { std::cout << n << "\n"; }
};

A a;
auto x = std::bind(&A::f, &a, std::placeholders::_1); //@ &a也可以换成a
x(42); // 42
```
- 绑定数据成员
```
struct A {
	int i = 1;
};

auto x = std::bind(&A::i, std::placeholders::_1);
A a;
std::cout << x(a) << "\n"; //@ 1
std::cout << x(&a) << "\n"; //@ 1
std::cout << x(std::make_unique<A>(a)) << "\n"; //@ 1
std::cout << x(std::make_shared<A>(a)) << "\n"; //@ 1
```
- 生成随机数
```
// 打印20个数字
std::default_random_engine e;
std::uniform_int_distribution<> d(0, 9);
auto x = std::bind(d, e);

for (int i = 0; i < 20; ++i)
	std::cout << x() << "\n";
```
## std::function用法示例
- 存储函数
```
void f(int i) { std::cout << i << "\n"; }
std::function<void(int)> g = f;
g(1); //@ 1
```
- 存储函数对象
```
struct X {
	void operator()(int n) const
	{
		std::cout << n << "\n";
	}
};

std::function<void(int)> f = X();
f(1); //@ 1
```
- 存储 `lambda`
```
std::function<void(int)> f = [](int i) { std::cout << i << "\n"; };
f(1); //@ 1
```
- 存储 `bind` 对象
```
void f(int i) { std::cout << i << "\n"; }
std::function<void(int)> g = std::bind(f, std::placeholders::_1);
g(1); //@ 1
```
- 存储绑定成员函数指针的 `bind` 对象
```
struct A {
	void f(int n) { std::cout << n<<"\n"; }
};

A a;
std::function<void(int)> g = std::bind(&A::f, &a, _1);
g(1);
```
- 存储成员函数
```
struct A {
	void f(int n) { std::cout << n << "\n"; }
};

std::function<void(A&, int)> g = &A::f;
A a;
g(a, 1); //@ 1
```
- 存储 `const` 成员函数
```
struct A {
	void f(int n) const { std::cout << n << "\n"; }
};

std::function<void(const A&, int)> g = &A::f;

A a;
g(a, 1); //@ 1
const A b;
g(b, 1); //@ 1
```
- 存储数据成员
```
struct A {
	int i = 1;
};

std::function<int(const A&)> g = &A::i;
A a;
std::cout << g(a) << "\n"; //@ 1
```












