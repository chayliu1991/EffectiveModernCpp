# std::atomic提供原子操作，volatile禁止优化内存

Java 中的 `volatile` 变量提供了同步机制，C++ 的 `volatile` 变量和并发没有任何关系。

[std::atomic ](https://en.cppreference.com/w/cpp/atomic/atomic) 是原子类型，提供了原子操作：

```
std::atomic<int> i(0);

void f()
{
    ++i; //@ 原子自增
    ++i; //@ 原子自增
}

void g()
{
    std::cout << i;
}

int main()
{
    std::thread t1(f);
    std::thread t2(g); //@ 结果只能是0或1或2
    t1.join();
    t2.join();
}
```

`volatile` 变量是普通的非原子类型，则不保证原子操作：

```
volatile int i(0);

void f()
{
    ++i; //@ 读改写操作，非原子操作
    ++i; //@ 读改写操作，非原子操作
}

void g()
{
    std::cout << i;
}

int main()
{
    std::thread t1(f);
    std::thread t2(g); //@ 存在数据竞争，值未定义
    t1.join();
    t2.join();
}
```

编译器或底层硬件对于不相关的赋值会重新排序以提高代码运行速度，[std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic) 可以限制重排序以保证顺序一致性：

```
std::atomic<bool> a(false);
int x = 0;

void f()
{
    x = 1; //@ 一定在a赋值为true之前执行
    a = true;
}

void g()
{
    if(a) std::cout << x;
}

int main()
{
    std::thread t1(f);
    std::thread t2(g);
    t1.join();
    t2.join(); //@ 不打印，或打印1
}
```

`volatile` 不会限制代码的重新排序：

```
volatile bool a(false);
int x = 0;

void f()
{
    x = 1; //@ 可能被重排在a赋值为true之后
    a = true;
}

void g()
{
    if(a) std::cout << x;
}

int main()
{
    std::thread t1(f);
    std::thread t2(g);
    t1.join();
    t2.join(); //@ 不打印，或打印0或1
}
```

`volatile` 的用处是告诉编译器正在处理的是特殊内存，不要对此内存上的操作进行优化。所谓优化指的是，如果把一个值写到内存某个位置，值会保留在那里，直到被覆盖，因此冗余的赋值就能被消除：

```
int x = 42;
int y = x;
y = x; //@ 冗余的初始化

//@ 优化为
int x = 42;
int y = x;
```

如果把一个值写到内存某个位置但从不读取，然后再次写入该位置，则第一次的写入可被消除：

```
int x;
x = 10;
x = 20;

//@ 优化为
int x;
x = 20；
```

结合上述两者：

```
int x = 42;
int y = x;
y = x;
x = 10;
x = 20;

//@ 优化为
int x = 42;
int y = x;
x = 20;
```

原子类型的读写也是可优化的：

```
std::atomic<int> y(x.load());
y.store(x.load());

//@ 优化为
register = x.load(); //@ 将x读入寄存器
std::atomic<int> y(register); //@ 用寄存器值初始化y
y.store(register); //@ 将寄存器值存入y
```

这种冗余的代码不会直接被写出来，但往往会隐含在大量代码之中。这种优化只在常规内存中合法，特殊内存则不适用。一般主存就是常规内存，特殊内存一般用于 memory-mapped I/O ，即与外部设备（如外部传感器、显示器、打印机、网络端口）通信。这个需求的原因在于，看似冗余的操作可能是有实际作用的：

```
int currentTemperature; //@ 传感器中记录当前温度的变量
currentTemperature = 25; //@ 更新当前温度，这条语句不应该被消除
currentTemperature = 26;
```













