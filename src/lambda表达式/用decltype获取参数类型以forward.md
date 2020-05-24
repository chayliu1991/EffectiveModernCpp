# 用decltype获取`auto&&`参数类型以std::forward
对于泛型 `lambda` 同样可以使用完美转发：
```
//@ 传入参数是auto，类型未知，std::forward的模板参数应该是什么？
auto f = [](auto&& x) { return g(std::forward< ? ? ? >(x)); };
```
此时可以用 `decltype` 判断传入的实参是左值还是右值
- 如果传递给 `auto&&` 的实参是左值，则 `x` 为左值引用类型，`decltype(x)` 为左值引用类型。
- 如果传递给 `auto&&` 的实参是右值，则 `x` 为右值引用类型，`decltype(x)` 为右值引用类型。
```
auto f = [](auto&& x) { return g(std::forward<decltype(x)>(x)); };
```
转发任意数量的实参：
```
auto f = [](auto&&... args) {
    return g(std::forward<decltype(args)>(args)...);
};
```

