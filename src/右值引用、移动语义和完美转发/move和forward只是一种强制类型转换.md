# std::move和std::forward只是一种强制类型转换

[std::move ](https://en.cppreference.com/w/cpp/utility/move) 不完全符合标准的实现如下：

```
template<typename T>
decltype(auto) move(T&& x)
{
    using ReturnType = remove_reference_t<T>&&;
    return static_cast<ReturnType>(x);
}
```

[std::move ](https://en.cppreference.com/w/cpp/utility/move) 会保留 `cv` 限定符：

```
void f(int&&)
{
	std::cout << 1 << "\n";
}

void f(const int&)
{
	std::cout << 2 << "\n";
}
const int i = 1;
f(std::move(i)); //@ 2
```

这可能导致的一个问题是，传入右值却执行拷贝操作：

```
class A {
public:
    explicit A(const std::string x)
        : s(std::move(x)) {} //@ 转为const std::string&&，调用std::string(const std::string&)
private:
    std::string s;
};
```

因此如果希望移动 [std::move](https://en.cppreference.com/w/cpp/utility/move) 生成的值，传给 [std::move](https://en.cppreference.com/w/cpp/utility/move) 的就不要是 `const`。

```
class A {
public:
    explicit A(std::string x)
    : s(std::move(x)) {} //@ 转为std::string&&，调用std::string(std::string&&)
private:
    std::string s;
};
```

C++11 之前的转发很简单：

```
void f(int&) { std::cout << 1 << "\n"; }
void f(const int&) { std::cout << 2 << "\n"; }

//@ 用多个重载转发给对应版本比较繁琐
void g(int& x)
{
	f(x);
}

void g(const int& x)
{
	f(x);
}

//@ 同样的功能可以用一个模板替代
template<typename T>
void h(T& x)
{
	f(x);
}

int main()
{
	int a = 1;
	const int b = 1;

	g(a); h(a); //@ 11
	g(b); h(b); //@ 22
	g(1); //@ 2
	h(1); //@ 错误
}
```

C++11 引入了右值引用，但原有的模板无法转发右值。如果使用 [std::move](https://en.cppreference.com/w/cpp/utility/move) 则无法转发左值，因此为了方便引入了 [std::forward](https://en.cppreference.com/w/cpp/utility/forward) 。

```
void f(int&) { std::cout << 1 << "\n"; }
void f(const int&) { std::cout << 2 << "\n"; }
void f(int&&) { std::cout << 3 << "\n"; }

//@ 用多个重载转发给对应版本比较繁琐
void g(int& x)
{
	f(x);
}

void g(const int& x)
{
	f(x);
}

void g(int&& x)
{
	f(std::move(x));
}

//@ 同样可以用一个模板来替代上述功能
template<typename T>
void h(T&& x)
{
	f(std::forward<T>(x)); //@ 注意std::forward的模板参数是T
}

int main()
{
	int a = 1;
	const int b = 1;

	g(a); h(a); //@ 11
	g(b); h(b); //@ 22
	g(std::move(a)); h(std::move(a)); //@ 33
	g(1); h(1); //@ 33
}
```

看起来完全可以用 [std::forward](https://en.cppreference.com/w/cpp/utility/forward) 取代 [std::move](https://en.cppreference.com/w/cpp/utility/move)，但[std::move](https://en.cppreference.com/w/cpp/utility/move) 的优势在于清晰简单。

```
h(std::forward<int>(a)); //@ 3
h(std::move(a)); //@ 3
```

结合可变参数模板，完美转发可以转发任意数量的实参：

```
template<typename... Ts>
void f(Ts&&... args)
{
    g(std::forward<Ts>(args)...); //@ 把任意数量的实参转发给g
}
```









