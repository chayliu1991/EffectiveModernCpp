# 用using别名声明替代typedef

[using](https://en.cppreference.com/w/cpp/language/type_alias) 别名声明比 [typedef](https://en.cppreference.com/w/cpp/language/typedef) 可读性更好，尤其是对于函数指针类型：

```
typedef void (*F)(int);
using F = void (*)(int);
```

C++11 还引入了 [别名模板](https://en.cppreference.com/w/cpp/language/type_alias)，它只能使用 [using](https://en.cppreference.com/w/cpp/language/type_alias) 别名声明：

```
template<typename T>
using X = std::vector<T>; //@ X<int>等价于std::vector<int>

//@ C++11之前的做法是在模板内部typedef
template<typename T>
struct Y { //@ Y<int>::type等价于std::vector<int>
	typedef std::vector<T> type;
};

//@ 在其他类模板中使用这两个别名的方式
template<typename T>
class A {
	X<T> x;
	typename Y<T>::type y;
};
```

C++11 引入了 [type traits](https://en.cppreference.com/w/cpp/header/type_traits)，为了方便使用，C++14 为每个 [type traits](https://en.cppreference.com/w/cpp/header/type_traits) 都定义了 [别名模板](https://en.cppreference.com/w/cpp/language/type_alias) ：

```
//@ std::remove_reference的实现
template<typename T>
struct remove_reference {
	using type = T;
};

template<typename T>
struct remove_reference<T&> {
	using type = T;
};

template<typename T>
struct remove_reference<T&&> {
	using type = T;
};

//@ std::remove_reference_t的实现
template<typename T>
using remove_reference_t = typename remove_reference<T>::type;
```

为了简化生成值的 [type traits](https://en.cppreference.com/w/cpp/header/type_traits)，C++14 还引入了 [变量模板](https://en.cppreference.com/w/cpp/language/variable_template)：

```
//@ std::is_same的实现
template<typename T, typename U>
struct is_same {
	static constexpr bool value = false;
};

//@ std::is_same_v的实现
template<typename T>
constexpr bool is_same_v = is_same<T, U>::value;
```

