本节将详细介绍 `std::thread` 的用法。

`std::thread` 在 `<thread>` 头文件中声明，因此使用 `std::thread` 需包含 `<thread>` 头文件。

## `<thread>` 头文件摘要 ##

`<thread>` 头文件声明了 std::thread 线程类及 `std::swap` (交换两个线程对象)辅助函数。另外命名空间 `std::this_thread` 也声明在 `<thread>` 头文件中。下面是 C++11 标准所定义的 `<thread>` 头文件摘要：

>参见 N3242=11-0012 草案第 30.3 节 Threads(p1133)。

    namespace std {
        #define __STDCPP_THREADS__ __cplusplus
        class thread;
        void swap(thread& x, thread& y);
        namespace this_thread {
            thread::id get_id();
            void yield();

            template <class Clock, class Duration>
            void sleep_until(const chrono::time_point<Clock, Duration>& abs_time);

            template <class Rep, class Period>
            void sleep_for(const chrono::duration<Rep, Period>& rel_time);
        }        
    }

`<thread>` 头文件主要声明了 `std::thread` 类，另外在 `std::this_thread` 命名空间中声明了 `get_id`，`yield`，`sleep_until` 以及 `sleep_for` 等辅助函数，本章稍微会详细介绍 `std::thread` 类及相关函数。


### `std::thread` 类摘要 ###

`std::thread` 代表了一个线程对象，C++11 标准声明如下：

    namespace std {
        class thread {
            public:
                // 类型声明:
                class id;
                typedef implementation-defined native_handle_type;

                // 构造函数、拷贝构造函数和析构函数声明:
                thread() noexcept;
                template <class F, class ...Args> explicit thread(F&& f, Args&&... args);
                ~thread();
                thread(const thread&) = delete;
                thread(thread&&) noexcept;
                thread& operator=(const thread&) = delete;
                thread& operator=(thread&&) noexcept;

                // 成员函数声明:
                void swap(thread&) noexcept;
                bool joinable() const noexcept;
                void join();
                void detach();
                id get_id() const noexcept;
                native_handle_type native_handle();
                
                // 静态成员函数声明:
                static unsigned hardware_concurrency() noexcept;
        };
    }

`std::thread` 中主要声明三类函数：(1). 构造函数、拷贝构造函数及析构函数；(2). 成员函数；(3). 静态成员函数。另外， `std::thread::id` 表示线程 ID，同时 C++11 声明如下：

    namespace std {
        class thread::id {
            public:
                id() noexcept;
        };
    
        bool operator==(thread::id x, thread::id y) noexcept;
        bool operator!=(thread::id x, thread::id y) noexcept;
        bool operator<(thread::id x, thread::id y) noexcept;
        bool operator<=(thread::id x, thread::id y) noexcept;
        bool operator>(thread::id x, thread::id y) noexcept;
        bool operator>=(thread::id x, thread::id y) noexcept;
    
        template<class charT, class traits>
        basic_ostream<charT, traits>&
            operator<< (basic_ostream<charT, traits>& out, thread::id id);
    
    
        // Hash 支持
        template <class T> struct hash;
        template <> struct hash<thread::id>;
    }

## `std::thread` 详解 ##

### `std::thread` 构造和赋值 ###

#### `std::thread` 构造函数 ####

<table>
<tbody>
<tr class="odd"><th>默认构造函数 (1)</th>
<td>thread() noexcept;</td>
</tr>
<tr class="even"><th>初始化构造函数 (2)</th>
<td>
template &lt;class Fn, class... Args&gt;<br/>
explicit thread(Fn&amp;&amp; fn, Args&amp;&amp;... args);
</td>
</tr>
<tr class="odd"><th>拷贝构造函数 [deleted] (3)</th>
<td>
thread(const thread&amp;) = delete;
</td>
</tr>
<tr class="even"><th>Move 构造函数 (4)</th>
<td>
thread(thread&amp;&amp; x) noexcept;
</td>
</tr>
</tbody>
</table>

1. 默认构造函数(1)，创建一个空的 `std::thread` 执行对象。
2. 初始化构造函数(2)，创建一个 `std::thread` 对象，该 `std::thread` 对象可被 `joinable`，新产生的线程会调用 `fn` 函数，该函数的参数由 `args` 给出。
3. 拷贝构造函数(被禁用)(3)，意味着 `std::thread` 对象不可拷贝构造。
4. Move 构造函数(4)，move 构造函数(move 语义是 C++11 新出现的概念，详见附录)，调用成功之后 `x` 不代表任何 `std::thread` 执行对象。

>注意：可被 `joinable` 的 `std::thread` 对象必须在他们销毁之前被主线程 `join` 或者将其设置为 `detached`.

std::thread 各种构造函数例子如下（[参考](http://en.cppreference.com/w/cpp/thread/thread/thread)）：

    void f1(int n)
    {
        for (int i = 0; i < 5; ++i) {
            std::cout << "Thread " << n << " executing\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    } 
    void f2(int& n)
    {
        for (int i = 0; i < 5; ++i) {
            std::cout << "Thread 2 executing\n";
            ++n;
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    } 
    int main()
    {
        int n = 0;
        std::thread t1; // t1 is not a thread
        std::thread t2(f1, n + 1); // pass by value
        std::thread t3(f2, std::ref(n)); // pass by reference
        std::thread t4(std::move(t3)); // t4 is now running f2(). t3 is no longer a thread
        t2.join(); t4.join();
        std::cout << "Final value of n is " << n << '\n';
    }
    
#### `std::thread` 启动一个线程  ####

    class Task{
    public:
        void operator()(int i){  //当函数调用
            cout << i << endl;
        }
    };
    for (uint8_t i = 0; i < 4; i++)
        {
            Task task;
            thread t(task, i);
            t.detach();
        }


在并行多线程下，其执行的结果就多种多样了，下图是代码一次运行的结果：
01\n
\n
23	
\n
\n
*/

把函数对象传入std::thread的构造函数时，要注意一个C++的语法解析错误（C++'s most vexing parse）。向std::thread的构造函数中传入的是一个临时变量，而不是命名变量就会出现语法解析错误。如下代码：std::thread t(Task());这里相当于声明了一个函数t，其返回类型为thread，而不是启动了一个新的线程。可以使用新的初始化语法避免这种情况std::thread t((Task()));std::thread t{ Task() };

    auto fn = [](int *a) {
    for (int i = 0; i < 10; i++)
        cout << *a << endl;
	};
	[=] {
		int a = 100;
		thread t(fn, &a);
		t.detach();//换成 t.join（）就没下面这么多事了，但是我们要分离它
	}();
	/*
	fn启动了一个新的线程，在装个新的线程中使用了局部变量a的指针，
	并且将该线程的运行方式设置为detach。这样，在下面这个lamb表达式 [=] {int a = 100;thread t(fn, &a);t.detach();}();执行结束后，变量a被销毁，但是在后台运行的线程仍然在使用已销毁变量a的指针,a是局部变量
    
  100    <-------------------在这只有第一个输出的100 剩下的都是随机地址值 
10418084 
10418084  
10418084  
10418084  
10418084  
10418084  
10418084  
10418084 
10418084 

#### `std::thread` 赋值操作 ####

<table style="width: 475px; height: 87px;">
<tbody>
<tr class="odd"><th>Move 赋值操作 (1)</th>
<td>
thread&amp; operator=(thread&amp;&amp; rhs) noexcept;
</td>
</tr>
<tr class="even"><th>拷贝赋值操作 [deleted] (2)</th>
<td>
thread&amp; operator=(const thread&amp;) = delete;
</td>
</tr>
</tbody>
</table>

1. Move 赋值操作(1)，如果当前对象不可 `joinable`，需要传递一个右值引用(`rhs`)给 `move` 赋值操作；如果当前对象可被 `joinable`，则会调用 `terminate`() 报错。
2. 拷贝赋值操作(2)，被禁用，因此 `std::thread` 对象不可拷贝赋值。

#### `std::thread` 传递参数 ####

	void func(int *a, int n) {}
	void oops()
	{
		int buffer[10];
		std::thread t(func, buffer, 10);
		//向线程调用的函数传递参数也是很简单的，只需要在构造thread的实例时，依次传入即可
		t.join();
	}
	//需要注意的是，  默认  的会将传递的参数以拷贝的方式复制到线程空间，即使参数的类型是引用
	void func1(int a, const std::string& str){}
	void oop(){
		std::thread t(func1, 3, "hello");
		//std::thread t(f,3,std::string(buffer));使用std::string避免
	}	
	<br> func的第二个参数是string &，而传入的是一个字符串字面量。该字面量以const char*类型传入线程空间后，在线程的空间内转换为string。 </br>
	如果在线程中使用引用来更新对象时，就需要注意了。默认的是将对象拷贝到线程空间，其引用的是拷贝的线程空间的对象，而不是初始希望改变的对象。如下

	class _tagNode{public: int a;int b;};
	void func2(_tagNode &node)
	{
		node.a = 10;
		node.b = 20;
	}
	void f()
	{
		_tagNode node;
		//thread t(func2, std::ref(node));
		thread t(func2, node);
		t.join();
		cout << node.a << endl;
		cout << node.b << endl;
	}
	//在线程内，将对象的字段a和b设置为新的值，但是在线程调用结束后，这两个字段的值并不会改变。
	//这样由于引用的实际上是局部变量node的一个拷贝，而不是node本身。在将对象传入线程的时候，调用std::ref，将node的引用传入线程，
	//而不是一个拷贝。改成这句thread t(func, std::ref(node));即可
	//也可以使用类的成员函数作为线程函数，示例如下
	class _Node {
	public:
		void do_some_work(int a) {};
	};
	_Node node;
	thread t(&_Node::do_some_work, &node, 20);//上面创建的线程会调用node.do_some_work(20)，第三个参数为成员函数第一个参数
	
### 其他成员函数 ###

> 本小节例子来自 [http://en.cppreference.com ](http://en.cppreference.com/w/cpp/thread/thread)

- `get_id`: 获取线程 ID，返回一个类型为 `std::thread::id` 的对象。请看下面例子：

		void hello() {
			std::cout << "Hello from thread " << std::this_thread::get_id() << std::endl;}
		int main() {
			std::vector<std::thread> threads;
			for (int i = 0; i < 5; ++i) {
				threads.push_back(std::thread(hello));
			}
			for (auto& thread : threads) {
				thread.join();
			}
			std::cin.get();
			return 0;
		} 
依次启动线程并将他们存入 vector 是管理多个线程的常用伎俩，这样你便可以轻松地改变线程的数量。
回到正题，就算是上面这样短小简单的例子，也不能断定其输出结果。理论情况是：  
Hello from thread 140276650997504  
Hello from thread 140276667782912  
Hello from thread 140276659390208  
Hello from thread 140276642604800  
Hello from thread 140276676175616  
   但实际上（至少在我这里）上述情况并不常见，你很可能得到的是如下结果：   
Hello from thread Hello from thread Hello from thread 139810974787328Hello from thread 139810983180032Hello from thread
139810966394624
139810991572736
139810958001920   
这是因为线程之间存在 interleaving 。你没办法控制线程的执行顺序，某个线程可能随时被抢占，
又因为输出到 ostream 分几个步骤（首先输出一个 string，然后是 ID，最后输出换行），
因此一个线程可能执行了第一个步骤后就被其他线程抢占了，直到其他所有线程打印完之后才能进行后面的步骤。

- `joinable`: 检查线程是否可被 join。检查当前的线程对象是否表示了一个活动的执行线程，由默认构造函数创建的线程是不能被 join 的。另外，如果某个线程 已经执行完任务，但是没有被 join 的话，该线程依然会被认为是一个活动的执行线程，因此也是可以被 join 的。

       
        void foo()
        {
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }         
        int main()
        {
            std::thread t;
            std::cout << "before starting, joinable: " << t.joinable() << '\n';         
            t = std::thread(foo);
            std::cout << "after starting, joinable: " << t.joinable() << '\n';         
            t.join();
        }

- `join`: Join 线程，调用该函数会阻塞当前线程，直到由 `*this` 所标示的线程执行完毕 join 才返回。
       
        void foo()
        {
            // simulate expensive operation
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }         
        void bar()
        {
            // simulate expensive operation
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }         
        int main()
        {
            std::cout << "starting first helper...\n";
            std::thread helper1(foo);         
            std::cout << "starting second helper...\n";
            std::thread helper2(bar);         
            std::cout << "waiting for helpers to finish..." << std::endl;
            helper1.join();
            helper2.join();         
            std::cout << "done!\n";
        }

- `detach`: Detach 线程。
将当前线程对象所代表的执行实例与该线程对象分离，使得线程的执行可以单独进行。一旦线程执行完毕，它所分配的资源将会被释放。

调用 detach 函数之后：

1. `*this` 不再代表任何的线程执行实例。
2. joinable() == false
3. get_id() == std::thread::id()

另外，如果出错或者 `joinable() == false`，则会抛出 `std::system_error`.  

### 异常情况下等待线程完成 ###

当决定以detach方式让线程在后台运行时，可以在创建 thread的实例后 立即调用detach，这样线程就会后thread的实例分离，即使出现了异常 thread的实例被 销毁 ，仍然能保证线程在后台运行。但线程以join方式运行时，需要在 主线程 的合适位置 调用join方法，如果调用 join 前出现了异常，thread被销毁，线程就会被 异常所终结。为了避免异常将线程终结，或者由于某些原因，例如线程访问了局部变量，就要保证线程一定要在 函数退出 前完成，就要保证要在 函数退出前调用join

	  void func() {
		thread t([] {
			cout << "hello C++ 11" << endl;
		});
		try
		{
			//do_something_else();
		}
		catch (...)
		{
			t.join();
			throw;
		}
		t.join();
	}    
上面代码能够保证在正常或者异常的情况下，都会调用join方法，这样线程一定会在函数func退出前完成。但是使用这种方法，不但代码冗长，而且会出现一些作用域的问题，并不是一个很好的解决方法。一种比较好的方法是资源获取即初始化（RAII,Resource Acquisition Is Initialization)，该方法提供一个类，在析构函数中调用join。

	class thread_guard
	{
		thread &t;
	public:
		explicit thread_guard(thread& _t) :t(_t) {}
		~thread_guard()
		{
			if (t.joinable())
				t.join();
		}
		thread_guard(const thread_guard&) = delete;
		thread_guard& operator=(const thread_guard&) = delete;
	};//无论是何种情况，当函数退出时，局部变量g调用其析构函数销毁，从而能够保证join一定被调用	
	void func() {
		thread t([] {
			cout << "Hello thread" << endl;
		});
		thread_guard g(t);	
	}
	
If you don’t need to wait for a thread to finish, you can avoid this exception-safety issue by detaching it. This breaks the association of the thread with the std::thread object and ensures that std::terminate() won’t be called when the std::thread object is destroyed, even though the thread is still running in the background.

### Returning a std:: fthreadrom a function ###
	
	std::thread g()
	{
		void some_other_function(int);
		std::thread t(some_other_function,42);
		return t;//返回右值
	}
	void f(std::thread t);
	void main()
	{
		void some_function(); 
		f(std::thread(some_function));//临时值
		f(g());
		std::thread t(some_function);
		f(std::move(t));//仔细看清 是哪个函数
	}
### scoped_thread ###

	class scoped_thread
	{
	    std::thread t;
	public:
	    explicit scoped_thread(std::thread t_):
	       t(std::move(t_))
	   {
	       if(!t.joinable())
	       throw std::logic_error(“No thread”);
	   }
	   ~scoped_thread()
	   {
	       t.join();
	   }
	   scoped_thread(scoped_thread const&)=delete;
	   scoped_thread& operator=(scoped_thread const&)=delete;
	};
	struct func;
	void f()
	{
	    int some_local_state;
	    scoped_thread t(std::thread(func(some_local_state)));
	    do_something_in_current_thread();
	}
#### Spawn some threads and wait for them to finish ####

	void do_work(unsigned id);
	void f()
	{
	    std::vector<std::thread> threads;
	    for(unsigned i=0;i<20;++i)
	    {
		threads.push_back(std::thread(do_work,i));
	    }
	    std::for_each(threads.begin(),threads.end(),std::mem_fn(&std::thread::join));
	    //函数模板men_fn()相当于STL中内置的仿函数,将一个成员函数作用在一个容器上
	}
#### 其他函数 ####

- `hardware_concurrency` [static]: 检测硬件并发特性，返回当前平台的线程实现所支持的线程并发数目，但返回值仅仅只作为系统提示(hint)。

        #include <iostream>
        #include <thread>
         
        int main() {
            unsigned int n = std::thread::hardware_concurrency();
            std::cout << n << " concurrent threads are supported.\n";
        }
	
- `swap`: Swap 线程，交换两个线程对象所代表的底层句柄(underlying handles)。    
         
        void foo(){std::this_thread::sleep_for(std::chrono::seconds(1));}         
        void bar(){std::this_thread::sleep_for(std::chrono::seconds(1));}         
        int main()
        {
            std::thread t1(foo);std::thread t2(bar);         
            std::cout << "thread 1 id: " << t1.get_id() << std::endl;std::cout << "thread 2 id: " << t2.get_id() << std::endl;         
            std::swap(t1, t2);
	    std::cout << "after std::swap(t1, t2):" << std::endl;
            std::cout << "thread 1 id: " << t1.get_id() << std::endl;std::cout << "thread 2 id: " << t2.get_id() << std::endl;         
            t1.swap(t2);         
            std::cout << "after t1.swap(t2):" << std::endl;
            std::cout << "thread 1 id: " << t1.get_id() << std::endl;std::cout << "thread 2 id: " << t2.get_id() << std::endl;         
            t1.join();t2.join();
        }

执行结果如下：

    thread 1 id: 1892
    thread 2 id: 2584
    after std::swap(t1, t2):
    thread 1 id: 2584
    thread 2 id: 1892
    after t1.swap(t2):
    thread 1 id: 1892
    thread 2 id: 2584

- `native_handle`: 返回 native handle（由于 `std::thread` 的实现和操作系统相关，因此该函数返回与 `std::thread` 具体实现相关的线程句柄，例如在符合 Posix 标准的平台下(如 Unix/Linux)是 Pthread 库）。

        #include <thread>
        #include <iostream>
        #include <chrono>
        #include <cstring>
        #include <pthread.h>
         
        std::mutex iomutex;
        void f(int num)
        {
            std::this_thread::sleep_for(std::chrono::seconds(1));
         
           sched_param sch;
           int policy; 
           pthread_getschedparam(pthread_self(), &policy, &sch);
           std::lock_guard<std::mutex> lk(iomutex);
           std::cout << "Thread " << num << " is executing at priority "
                     << sch.sched_priority << '\n';
        }
         
        int main()
        {
            std::thread t1(f, 1), t2(f, 2);
         
            sched_param sch;
            int policy; 
            pthread_getschedparam(t1.native_handle(), &policy, &sch);
            sch.sched_priority = 20;
            if(pthread_setschedparam(t1.native_handle(), SCHED_FIFO, &sch)) {
                std::cout << "Failed to setschedparam: " << std::strerror(errno) << '\n';
            }
         
            t1.join();
            t2.join();
        }

执行结果如下：

    Thread 2 is executing at priority 0
    Thread 1 is executing at priority 20



## `std::this_thread` 命名空间中相关辅助函数介绍 ##

- get_id: 获取线程 ID。

        #include <iostream>
        #include <thread>
        #include <chrono>
        #include <mutex>
         
        std::mutex g_display_mutex;
         
        void foo()
        {
            std::thread::id this_id = std::this_thread::get_id();
         
            g_display_mutex.lock();
            std::cout << "thread " << this_id << " sleeping...\n";
            g_display_mutex.unlock();
         
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }
         
        int main()
        {
            std::thread t1(foo);
            std::thread t2(foo);
         
            t1.join();
            t2.join();
        }

- yield: 当前线程放弃执行，操作系统调度另一线程继续执行。

        #include <iostream>
        #include <chrono>
        #include <thread>
         
        // "busy sleep" while suggesting that other threads run 
        // for a small amount of time
        void little_sleep(std::chrono::microseconds us)
        {
            auto start = std::chrono::high_resolution_clock::now();
            auto end = start + us;
            do {
                std::this_thread::yield();
            } while (std::chrono::high_resolution_clock::now() < end);
        }
         
        int main()
        {
            auto start = std::chrono::high_resolution_clock::now();
         
            little_sleep(std::chrono::microseconds(100));
         
            auto elapsed = std::chrono::high_resolution_clock::now() - start;
            std::cout << "waited for "
                      << std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count()
                      << " microseconds\n";
        }

- sleep_until: 线程休眠至某个指定的时刻(time point)，该线程才被重新唤醒。

        template< class Clock, class Duration >
        void sleep_until( const std::chrono::time_point<Clock,Duration>& sleep_time );



- sleep_for: 线程休眠某个指定的时间片(time span)，该线程才被重新唤醒，不过由于线程调度等原因，实际休眠时间可能比 `sleep_duration` 所表示的时间片更长。

        template< class Rep, class Period >
        void sleep_for( const std::chrono::duration<Rep,Period>& sleep_duration );

        #include <iostream>
        #include <chrono>
        #include <thread>
         
        int main()
        {
            std::cout << "Hello waiter" << std::endl;
            std::chrono::milliseconds dura( 2000 );
            std::this_thread::sleep_for( dura );
            std::cout << "Waited 2000 ms\n";
        }

执行结果如下：

    Hello waiter
    Waited 2000 ms
    
## `deadlock` 死锁情况 ##

		mutex m1, m2;
		int main()
		{
			thread t1;
			thread t2;
			t1 = thread([&t2] {
				cout << "thread-1 start!" << endl;
				std::chrono::seconds dura(2);
				std::this_thread::sleep_for(dura);
				t2.join();
				cout << "thread-1 finished!" << endl;
			});
			t2 = thread([&t1] {
				cout << "thread-2 start!" << endl;
				std::chrono::seconds dura(2);
				std::this_thread::sleep_for(dura);
				t1.join();
				cout << "thread-2 finished!" << endl;
			});			
			return 0;
		}


