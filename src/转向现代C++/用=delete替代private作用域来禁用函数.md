# 用=delete替代private作用域来禁用函数

C++11 之前禁用拷贝的方式是将拷贝构造函数和拷贝赋值运算符声明在 `private` 作用域中：

```
class A {
private:
    A(const A&); // 不需要定义
    A& operator=(const A&);
};
```

C++11 中可以直接将要删除的函数用 `=delete` 声明，习惯上会声明在 `public` 作用域中，这样在使用删除的函数时，会先检查访问权再检查删除状态，出错时能得到更明确的诊断信息。

```
class A {
public:
    A(const A&) = delete;
    A& operator(const A&) = delete;
};
```

`private` 作用域中的函数还可以被成员和友元调用，而 `=delete` 是真正禁用了函数，无法通过任何方法调用。

任何函数都可以用 `=delete` 声明，比如函数不想接受某种类型的参数，就可以删除对应类型的重载。

```
void f(int);
void f(double) = delete; //@ 拒绝double和float类型参数
f(3.14); //@ 错误
```

`=delete` 还可以禁止模板对某个类型的实例化：

```
template<typename T>
void f(T x) {}

template<>
void f<int>(int) = delete;

f(1); //@ 错误：使用已删除的函数
template<typename T>
void processPointer(T * ptr);
```

类内的函数模板也可以用这种方式禁用：

```
class A {
public:
template<typename T>
	void f(T x) {}
};

template<>
void A::f<int>(int) = delete;
```

当然，写在 `private` 作用域也可以起到禁用的效果：

```
class A {
public:
    template<typename T>
    void f(T x) {}
private:
    template<>
    void f<int>(int);
};
```

但把模板和特化置于不同的作用域不太合逻辑，与其效仿 `=delete` 的效果，不如直接用 `=delete`。

