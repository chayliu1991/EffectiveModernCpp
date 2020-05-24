# 用override标记被重写的虚函数

虚函数的重写很容易出错，因为要在派生类中重写虚函数，必须满足一系列要求

- 基类中必须有此虚函数
- 基类和派生类的函数名相同（析构函数除外）
- 函数参数类型相同
- `const` 属性相同
- 函数返回值和异常说明相同

C++11 多出一条要求：引用修饰符相同。引用修饰符的作用是，指定成员函数仅在对象为左值（成员函数标记为 `&`）或右值（成员函数标记为 `&&`）时可用：

```
class A {
public:
    void f()& { std::cout << 1 << "\n"; } //@ *this是左值时才使用
    void f()&& { std::cout << 2 << "\n"; } //@ *this是右值时才使用
};

A makeA() { return A{}; }

A a;
a.f(); //@ 1
makeA().f(); //@ 2
```

对于这么多的要求难以面面俱到，比如下面代码没有任何重写但可以通过编译：

```
class A {
public:
    virtual void f1() const;
    virtual void f2(int x);
    virtual void f3() &;
    void f4() const;
};

class B : public A {
public:
    virtual void f1();
    virtual void f2(unsigned int x);
    virtual void f3() &&;
    void f4() const;
};
```

为了保证正确性，C++11 提供了 [override](https://en.cppreference.com/w/cpp/language/override) 来标记要重写的虚函数，如果未重写就不能通过编译：

```
class A {
public:
    virtual void f1() const;
    virtual void f2(int x);
    virtual void f3() &;
    virtual void f4() const;
};

class B : public A {
public:
    virtual void f1() const override;
    virtual void f2(int x) override;
    virtual void f3() & override;
    void f4() const override;
};
```

[override](https://en.cppreference.com/w/cpp/language/override) 是一个 `contextual keyword`，只在特殊语境中保留，[override](https://en.cppreference.com/w/cpp/language/override) 只有出现在成员函数声明末尾才有保留意义，因此如果以前的遗留代码用到了 [override](https://en.cppreference.com/w/cpp/language/override) 作为名字，不用改名就可以升到 C++11。

```
class A {
public:
	void override(); //@ 在C++98和C++11中都合法
};
```

C++11 还提供了另一个 `contextual keyword`：[final](https://en.cppreference.com/w/cpp/language/final) 可以用来指定虚函数禁止被重写：

```
class A {
public:
    virtual void f() final;
    void g() final; //@ 错误：final只能用于指定虚函数
};

class B : public A {
public:
    virtual void f() override; //@ 错误：f不可重写
};
```

[final ](https://en.cppreference.com/w/cpp/language/final) 还可以用于指定某个类禁止被继承：

```
class A final {};
class B : public A {}; //@ 错误：A禁止被继承
```



