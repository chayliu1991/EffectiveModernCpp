# 用std::cbegin和std::cend获取const_iterator

需要迭代器但不修改值时就应该使用 `const_iterator`，获取和使用 `const_iterator` 十分简单：

```
std::vector<int> v{ 2, 3 };
auto it = std::find(std::cbegin(v), std::cend(v), 2); //@ C++14
v.insert(it, 1);
```

上述功能很容易扩展成模板：

```
template<typename C, typename T>
void f(C& c, const T& x, const T& y)
{
    auto it = std::find(std::cbegin(c), std::cend(c), x);
    c.insert(it, y);
}
```

C++11 没有 [std::cbegin](https://en.cppreference.com/w/cpp/iterator/begin) 和 [std::cend](https://en.cppreference.com/w/cpp/iterator/end)，手动实现即可：

```
template<class C>
auto cbegin(const C & c)->decltype(std::begin(c))
{
	return std::begin(c); //@ c是const所以返回const_iterator
}
```































