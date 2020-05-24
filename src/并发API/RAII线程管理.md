# RAII线程管理

每个 [std::thread](https://en.cppreference.com/w/cpp/thread/thread) 对象都处于可合并（joinable）或不可合并（unjoinable）的状态。一个可合并的 [std::thread](https://en.cppreference.com/w/cpp/thread/thread) 对应于一个底层异步运行的线程，若底层线程处于阻塞、等待调度或已运行结束的状态，则此 [std::thread ](https://en.cppreference.com/w/cpp/thread/thread) 可合并，否则不可合并。不可合并的 [std::thread](https://en.cppreference.com/w/cpp/thread/thread) 包括：

- 默认构造的 [std::thread](https://en.cppreference.com/w/cpp/thread/thread)：此时没有要运行的函数，因此没有对应的底层运行线程
- 已移动的 [std::thread](https://en.cppreference.com/w/cpp/thread/thread)：移动操作导致底层线程被转用于另一个[std::thread](https://en.cppreference.com/w/cpp/thread/thread)
- 已 [join](https://en.cppreference.com/w/cpp/thread/thread/join) 或已 [join](https://en.cppreference.com/w/cpp/thread/thread/join) 的 [std::thread](https://en.cppreference.com/w/cpp/thread/thread)

如果可合并的 [std::thread](https://en.cppreference.com/w/cpp/thread/thread) 对象的析构函数被调用，则程序的执行将终止：

```
void f() {}

void g()
{
    std::thread t(f); //@ t.joinable() == true
}

int main()
{
    g(); //@ g运行结束时析构t，导致整个程序终止
    ...
}
```

析构可合并的 [std::thread](https://en.cppreference.com/w/cpp/thread/thread) 时，隐式 [join](https://en.cppreference.com/w/cpp/thread/thread/join) 或隐式 [detach](https://en.cppreference.com/w/cpp/thread/thread/detach) 的带来问题更大。隐式 [join](https://en.cppreference.com/w/cpp/thread/thread/join) 导致 `g` 运行结束时仍要保持等待 `f` 运行结束，这就会导致性能问题，且调试时难以追踪原因。隐式 [detach](https://en.cppreference.com/w/cpp/thread/thread/detach) 导致的调试问题更为致命：

```
void f(int&) {}

void g()
{
    int i = 1;
    std::thread t(f, i);
} //@ 如果隐式detach，局部变量i被销毁，但f仍在使用局部变量的引用
```

完美销毁一个可合并的 [std::thread](https://en.cppreference.com/w/cpp/thread/thread) 十分困难，因此规定销毁将导致终止程序。要避免程序终止，只要让可合并的线程在销毁时变为不可合并状态即可，使用 RAII 手法就能实现这点。

```
class A {
public:
    enum class DtorAction { join, detach };
    A(std::thread&& t, DtorAction a) : action(a), t(std::move(t)) {}
    ~A()
    {
        if (t.joinable())
        {
            if (action == DtorAction::join) t.join();
            else t.detach();
        }
    }
    A(A&&) = default;
    A& operator=(A&&) = default;
    std::thread& get() { return t; }
private:
    DtorAction action;
    std::thread t;
};
void f() {}

void g()
{
    A t(std::thread(f), A::DtorAction::join); //@ 析构前使用join
}

int main()
{
    g(); //@ g运行结束时将内部的std::thread置为join，变为不可合并状态
    //@ 析构不可合并的std::thread不会导致程序终止
    //@ 这种手法带来了隐式join和隐式detach的问题，但可以调试
    ...
}
```























