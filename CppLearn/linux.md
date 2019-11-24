___this file is my knowledge about <linux多线程服务端编程>___


1. mutex并不是多线程的办法。mutex只能保证函数一个接一个的执行。考虑以下代码：   
```
Foo::~Foo()
{
    MutexLockGuard lock(mutex_);
    //free internal state (1)
}

//
void Foo::update()
{
    MutexLockGuard lock(mutex_)// (2)
    // make use of internal state
}
```   
此时A/B两个线程都能看到Foo对象x，线程A即将销毁x，而线程B正准备调用`x->update()`。   

尽管线程A在销毁独享之后把指针置为NULL，尽管线程B在调用x的成员函数之前检查了指针x的值，但是还是无法避免一种竞态条件.   
    1. 线程A执行到了析构函数的(1)处，已经持有了互斥锁，即将向下执行。   
    2. 线程B通过了if(x)检测，阻塞在(2)处。    
接下来发生什么就不知道了。   

2. 如果要同时读写一个class的两个对象，有潜在的死锁的可能。比如有一个swap():   

	void swap(Counter &a, Counter &b)
	{
	    MutexLockGuard a_lock(a.mutex_); //potential dead lock
	    MutexLockGuard b_lock(b.mutex_);
	    int64_t value = a.value_;
	    a.value_ = b.value_;
	    b.value_ = value;
	}

如果线程A执行swap(a,b)，而同时线程B执行swap(b,a)，就有可能死锁。operator=()也是这个道理。    

	Counter& Counter::operator::operator=(const Counter &rhs)
	{
	    if (this == &rhs)
	    {
	        return *this;
	    }
	
	    MutexLockGuard my_lock(mutex_); //potential dead lock
	    MutexLockGuard its_lock(rhs.mutex_);
	    value_ = rhs.value_;// 改成value_ = rhs.value()会死锁
	    return *this;
	}

一个函数如果要锁住相同类型的多个对象，为了保证始终按相同的顺序加锁，我们可以比较mutex对象的地址，始终先加锁地址较小的mutex。     
3. 对象的关系主要有三种：composition(组合)/aggregation(聚合)/association(关联)。组合关系在多线程里面不会遇到什么麻烦，因为对象x的生命周期由其唯一的拥有者owner控制(我猜测就是一个class A内定义另一个class B，class B不是指针和引用，仅仅是class B的一个实例)。关联表示一个对象a用到了另一个对象b，调用了后者的成员函数。从代码形式看，a持有b的指针(或引用)，但是b的生命周期不由a单独控制。聚合关系从形式上看雨关联相同，除了a和b有逻辑上的整体与部分的关系。如果b是动态创建的并在整个程序结束前有可能被释放，就会出现竞态条件。一个简单的解决办法是：只创建不销毁。程序使用一个对象池来暂存用过的对象，下次申请新的对象时，如果对象池里有存货，就重复利用现有的对象，否则就新建立一个。对象用完了，不是直接放掉，而是放回池子里。这个方法有缺点，但是至少能避免访问失效对象的情况发生。这种山寨的方法是:   
    1. 对象池的线程安全，如果安全的、完整的把对象放回池子里，防止出现“部分放回”的竞态？(线程A认为对象x已经放回，线程B认为对象x还活着)。   
    2. 全局共享数据引发的锁竞争，这个集中化的对象池会不会把多线程并发的操作串行化？   
    3. 如果共享对象的类型不止一种，那么是重复实现对象池还是使用类模板？   
    4. 会不会造成内存泄露与分片？因为对象池占用的内存只增不减，而且多个对象池不能共享内存。    
如果对象x注册了任何非静态成员函数回调，那么必然会在某处持有了指向x的指针，这就暴露在竞态条件之下。    