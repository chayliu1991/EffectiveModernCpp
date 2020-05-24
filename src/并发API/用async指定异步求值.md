# 用std::launch::async指定异步求值

[std::async](https://en.cppreference.com/w/cpp/thread/async) 有两种标准启动策略：

- [std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)：函数必须异步运行，即运行在不同的线程上。
- [std::launch::deferred](https://en.cppreference.com/w/cpp/thread/launch)：函数只在对返回的 [std::future](https://en.cppreference.com/w/cpp/thread/future) 调用[get](https://en.cppreference.com/w/cpp/thread/future/get)或[wait](https://en.cppreference.com/w/cpp/thread/future/wait)时运行。即执行会推迟，调用 [get](https://en.cppreference.com/w/cpp/thread/future/get) 或 [wait](https://en.cppreference.com/w/cpp/thread/future/wait) 时函数会同步运行，调用方会阻塞至函数运行结束。

[std::async](https://en.cppreference.com/w/cpp/thread/async) 的默认启动策略不是二者之一，而是对二者求或的结果。

```
auto ft1 = std::async(f); //@ 意义同下
auto ft2 = std::async(std::launch::async | std::launch::deferred, f);
```

默认启动策略允许异步或同步运行函数，这种灵活性使得 [std::async](https://en.cppreference.com/w/cpp/thread/async) 和标准库的线程管理组件能负责线程的创建和销毁、负载均衡以及避免 oversubscription。

但默认启动策略存在一些潜在问题，比如给定线程 `t` 执行如下语句：

```
auto ft = std::async(f);
```

潜在的问题有：

- 无法预知 `f` 和 `t` 是否会并发运行，因为 `f` 可能被调度为推迟运行。
- 无法预知运行 `f` 的线程是否不同于对 `ft` 调用 [get](https://en.cppreference.com/w/cpp/thread/future/get) 或 [wait](https://en.cppreference.com/w/cpp/thread/future/wait) 的线程，如果调用 [get](https://en.cppreference.com/w/cpp/thread/future/get) 或 [wait](https://en.cppreference.com/w/cpp/thread/future/wait) 的线程是 `t`，就说明无法预知 `f` 是否会运行在与 `t` 不同的某线程上。
- 甚至很可能无法预知 `f` 是否会运行，因为无法保证在程序的每条路径上，`ft` 的 [get](https://en.cppreference.com/w/cpp/thread/future/get) 或 [wait](https://en.cppreference.com/w/cpp/thread/future/wait) 会被调用。

默认启动策略在调度上的灵活性会在使用 [thread_local](https://en.cppreference.com/w/cpp/keyword/thread_local) 变量时导致混淆，这意味着如果 `f` 读写此 thread-local

storage（TLS）时，无法预知哪个线程的局部变量将被访问。

```
auto ft = std::async(f); //@ f的TLS可能和一个独立线程相关，但也可能与对ft调用get或wait的线程相关
```

它也会影响使用 timeout 的 wait-based 循环，因为对返回的 [std::future](https://en.cppreference.com/w/cpp/thread/future) 调用 [wait_for](https://en.cppreference.com/w/cpp/thread/future/wait_for) 或 [wait_until](https://en.cppreference.com/w/cpp/thread/future/wait_until) 会产生[std::future_status::deferred](https://en.cppreference.com/w/cpp/thread/future_status) 值。这意味着以下循环看似最终会终止，但实际可能永远运行：

```
using namespace std::literals;

void f()
{
    std::this_thread::sleep_for(1s);
}

auto ft = std::async(f);
while (ft.wait_for(100ms) != std::future_status::ready)
{ //@ 循环至f运行完成，但这可能永远不会发生
    …
}
```

如果选用了 [std::launch::async](https://en.cppreference.com/w/cpp/thread/launch) 启动策略，`f` 和调用 [std::async ](https://en.cppreference.com/w/cpp/thread/async) 的线程并发执行，则没有问题。但如果 `f `被推迟执行，则 [wait_for](https://en.cppreference.com/w/cpp/thread/future/wait_for) 总会返回 [std::future_status::deferred](https://en.cppreference.com/w/cpp/thread/future_status)，于是循环永远不会终止。

这类 bug 在开发和单元测试时很容易被忽略，只有在运行负载很重时才会被发现。解决方法很简单，检查返回的[std::future](https://en.cppreference.com/w/cpp/thread/future)，确定任务是否被推迟。但没有直接检查是否推迟的方法，替代的手法是，先调用一个 timeout-based 函数，比如 [wait_for](https://en.cppreference.com/w/cpp/thread/future/wait_for)，这并不表示想等待任何事，而只是为了查看返回值是否为 [std::future_status::deferred](https://en.cppreference.com/w/cpp/thread/future_status)。

```
auto ft = std::async(f);
if (ft.wait_for(0s) == std::future_status::deferred) //@ 任务被推迟
{
    … //@ 使用ft的wait或get异步调用f
}
else //@ 任务未被推迟
{
    while (ft.wait_for(100ms) != std::future_status::ready)
    {
        … //@ 任务未被推迟也未就绪，则做并发工作直至结束
    }
    … //@ ft准备就绪
}
```

综上，[std::async](https://en.cppreference.com/w/cpp/thread/async) 使用默认启动策略创建要满足以下所有条件：

- 任务不需要与对返回值调用 [get](https://en.cppreference.com/w/cpp/thread/future/get) 或 [wait](https://en.cppreference.com/w/cpp/thread/future/wait) 的线程并发执行
- 读写哪个线程的 thread_local 变量没有影响
- 要么保证对返回值调用 [get](https://en.cppreference.com/w/cpp/thread/future/get) 或 [wait](https://en.cppreference.com/w/cpp/thread/future/wait)，要么接受任务可能永远不执行
- 使用 [wait_for](https://en.cppreference.com/w/cpp/thread/future/wait_for) 或 [wait_until](https://en.cppreference.com/w/cpp/thread/future/wait_until) 的代码要考虑任务被推迟的可能

只要一点不满足，就可能意味着想确保异步执行任务，这只需要指定启动策略为 [std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)。

```
auto ft = std::async(std::launch::async, f); //@ 异步执行f
```

默认使用 [std::launch::async](https://en.cppreference.com/w/cpp/thread/launch) 启动策略的 [std::async](https://en.cppreference.com/w/cpp/thread/async) 将会是一个很方便的工具，实现如下：

```
template<typename F, typename... Ts>
inline
auto // std::future<std::invoke_result_t<F, Ts...>>
reallyAsync(F&& f, Ts&&... args)
{
    return std::async(std::launch::async,
        std::forward<F>(f),
        std::forward<Ts>(args)...);
}
```

这个函数的用法和 [std::async](https://en.cppreference.com/w/cpp/thread/async) 一样：

```
auto ft = reallyAsync(f); //@ 异步运行f，如果std::async抛出异常则reallyAsync也抛出异常
```



