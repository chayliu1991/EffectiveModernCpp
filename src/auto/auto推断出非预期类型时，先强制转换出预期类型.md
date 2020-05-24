# auto推断出非预期类型时，先强制转换出预期类型

如下代码没有问题：

 ```
bool x = f()[0];
if (x) std::cout << "OK"<<"\n";
 ```

但如果把显式声明改为 `auto` 则会出现非预期行为：

```
std::vector<bool> f()
{
	return std::vector<bool>{ true, false };
}

auto x = f()[0]; //@ 改用auto声明
if (x) std::cout << "OK" << "\n"; //@ 错误：未定义行为
```

原因在于实际上得到的类型不是 `bool`：

```
auto x = f()[0]; //@ x类型为std::vector<bool>::reference
```

[std::vector](https://en.cppreference.com/w/cpp/container/vector_bool) 不是真正的 STL 容器，也不包含 `bool`  类型元素。它是 [std::vector](https://en.cppreference.com/w/cpp/container/vector) 对于 `bool` 类型的特化，为了节省空间，每个元素用一个 `bit`（而非一个 `bool` ）表示，于是 [operator](https://en.cppreference.com/w/cpp/container/vector/operator_at) 返回的应该是单个 `bit` 的引用，但 C++ 中不存在指向单个 `bit` 的指针，因此也不能获取单个 `bit` 的引用。

```
std::vector<bool> v { true, false };
bool* p = &v[0]; //@ 错误
std::vector<bool>::reference* q = &v[0]; //@ 正确
```

因此需要一个行为类似单个 `bit` 并可以被引用的对象，也就是 [std::vector::reference](https://en.cppreference.com/w/cpp/container/vector_bool/reference)，它可以隐式转换为 `bool`。

```
bool x = f()[0];
```

而对于 `auto` 推断则不会进行隐式转换：

```
auto x = f()[0]; //@ std::vector<bool>::reference x = f()[0];
//@ x不一定指向std::vector<bool>的第0个bit，这取决于std::vector<bool>::reference的实现
//@ 一种实现是含有一个指向一个machine word的指针，word持有被引用的bit和这个bit相对word的offset
//@ 于是x持有一个由opeartor[]返回的临时的machine word的指针和bit的offset
//@ 这条语句结束后临时对象被析构，于是x含有一个空悬指针，导致后续的未定义行为
if(x) ... // 相当于int* p; if(p) ...
```

[std::vector::reference ](https://en.cppreference.com/w/cpp/container/vector_bool/reference) 是一个代理类（`proxy class`，模拟或扩展其他类型的类）的例子，比如 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 和 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 是很明显的代理类。还有一些为了提高数值计算效率而使用表达式模板技术开发的类，比如给定一个 `Matrix ` 类和它的对象：

```
Matrix sum = m1 + m2 + m3 + m4;
```

`Matrix` 对象的 `operator+` 返回的是结果的代理而非结果本身，这样可以使得表达式的计算更为高效。

```
auto x = m1 + m2; //@ x可能是Sum<Matrix, Matrix>而不是Matrix对象
```

`auto` 推断出代理类的问题实际很容易解决，事先做一次到预期类型的强制转换即可：

```
auto x = static_cast<bool>(f()[0]);
```



