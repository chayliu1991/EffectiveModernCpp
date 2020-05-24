# 用std::weak_ptr观测std::shared_ptr的内部状态

[std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr) 不能解引用，它不是一种独立的智能指针，而是 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 的一种扩充，它用[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)初始化，共享对象但不改变引用计数，主要作用是观察 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 的内部状态。

```
std::weak_ptr<int> w;

void f(std::weak_ptr<int> w)
{
	if (auto p = w.lock()) std::cout << *p << "\n";
	else std::cout << "can't get value" << "\n";
}

int main()
{
	{
		auto p = std::make_shared<int>(42);
		w = p;
		assert(p.use_count() == 1);
		assert(w.expired() == false);
		f(w); //@ 42
		auto q = w.lock();
		assert(p.use_count() == 2);
		assert(q.use_count() == 2);
	}
	f(w); //@ can't get value
	assert(w.expired() == true);
	assert(w.lock() == nullptr);
}
```

[std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr) 的另一个作用是解决循环引用问题：

```
class B;
class A {
public:
	std::shared_ptr<B> b;
};

class B {
public:
	std::shared_ptr<A> a; //@ std::weak_ptr<A> a;
};

int main()
{
	{
		std::shared_ptr<A> x(new A);
		x->b = std::shared_ptr<B>(new B);
		x->b->a = x;
	} //@ x.use_count由2减为1，不会析构，于是x->b也不会析构，导致两次内存泄漏
	//@ 如果B::a改为std::weak_ptr，则use_count不会为2而保持1，此处就会由1减为0，从而正常析构
}
```

