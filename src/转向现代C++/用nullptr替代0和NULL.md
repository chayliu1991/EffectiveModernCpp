# 用nullptr替代0和NULL

字面值0本质是 `int` 而非指针，只有在使用指针的语境中发现0才会解释为空指针。

[NULL](https://en.cppreference.com/w/cpp/types/NULL) 的本质是宏，没有规定的实现标准，一般在 C++ 中定义为0，在 C 中定义为 `void*`。

```
//@ VS2017中的定义
#ifndef NULL
#ifdef __cplusplus
#define NULL 0
#else
#define NULL ((void *)0)
#endif
#endif
```

在重载解析时，[NULL](https://en.cppreference.com/w/cpp/types/NULL) 作为参数不会优先匹配指针类型。而 [nullptr](https://en.cppreference.com/w/cpp/language/nullptr) 的类型是 [std::nullptr_t](https://en.cppreference.com/w/cpp/types/nullptr_t)，[std::nullptr_t](https://en.cppreference.com/w/cpp/types/nullptr_t) 可以转换为任何原始指针类型。

```
void f(bool) { std::cout << 1 << "\n"; }
void f(int) { std::cout << 2 << "\n"; }
void f(void*) { std::cout << 3 << "\n"; }
f(0); //@ 2
f(NULL); //@ 2
f(nullptr); //@ 3
```

这点也会影响模板实参推断：

```
template<typename T>
void f() {}

f(0); //@ T推断为int
f(NULL); //@ T推断为int
f(nullptr); //@ T推断为std::nullptr_t
```

使用 [nullptr](https://en.cppreference.com/w/cpp/language/nullptr) 就可以避免推断出非指针类型：

 ```
void f1(std::shared_ptr<int>) {}
void f2(std::unique_ptr<int>) {}
void f3(int*) {}

template<typename F, tpyename T>
void g(F f, T x)
{
	f(x);
}

g(f1, 0); //@ 错误
g(f1, NULL); //@ 错误
g(f1, nullptr); //@ OK

g(f2, 0); //@ 错误
g(f2, NULL); //@ 错误
g(f2, nullptr); //@ OK

g(f3, 0); //@ 错误
g(f3, NULL); //@ 错误
g(f3, nullptr); //@ OK
 ```

使用 [nullptr](https://en.cppreference.com/w/cpp/language/nullptr) 也能使代码意图更清晰：

```
auto res = f();
if (res == nullptr) ... //@ 很容易看出res是指针类型
```



