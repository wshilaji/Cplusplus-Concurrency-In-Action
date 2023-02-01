## 1. 核心语言的运行时性能强化 ##

### 1.1 右值引用和 move 语义 ###

> 本小节主要参考了 qicosmos 上的一篇博文: 《从4行代码看右值引用》: [https://www.cnblogs.com/qicosmos/p/4283455.html](https://www.cnblogs.com/qicosmos/p/4283455.html)，在此特向原作者表示感谢。

##### 概述 #####

右值引用的概念有些读者可能会感到陌生，其实他和C++98/03中的左值引用有些类似，例如，c++98/03中的左值引用是这样的：

    int i = 0;
    int& j = i;这个i是左值
    
这里的int&是对左值进行绑定（但是int&却不能绑定右值），相应的，对右值进行绑定的引用就是右值引用，他的语法是这样的A&&，通过双引号来表示绑定类型为A的右值。通过&&我们就可以很方便的绑定右值了，比如我们可以这样绑定一个右值：
    
    int&& i = 0;

   这里我们绑定了一个右值0，关于右值的概念会在后面介绍。右值引用是C++11中新增加的一个很重要的特性，他主是要用来解决C++98/03中遇到的两个问题，第一个问题就是临时对象非必要的昂贵的拷贝操作，第二个问题是在模板函数中如何按照参数的实际类型进行转发。通过引入右值引用，很好的解决了这两个问题，改进了程序性能，后面将会详细介绍右值引用是如何解决这两个问题的。
　 和右值引用相关的概念比较多，比如：右值、纯右值、将亡值、universal references、引用折叠、移动语义、move语义和完美转发等等。很多都是新概念，对于刚学习C++11右值引用的初学者来说，可能会觉得右值引用过于复杂，概念之间的关系难以理清。
   右值引用实际上并没有那么复杂，其实是关于4行代码的故事，通过简单的4行代码我们就能清晰的理解右值引用相关的概念了。本文希望带领读者通过4行代码来理解右值引用相关的概念，理清他们之间的关系，并最终能透彻地掌握C++11的新特性--右值引用

#### 四行代码的故事 ####
---

#### 第1行代码的故事 ####

    int i = getVar();

   上面的这行代码很简单，从getVar()函数获取一个整形值，然而，这行代码会产生几种类型的值呢？答案是会产生两种类型的值，一种是左值i，一种是函数getVar()返回的临时值，这个临时值在表达式结束后就销毁了，而左值i在表达式结束后仍然存在，这个临时值就是右值，具体来说是一个纯右值，右值是不具名的。区分左值和右值的一个简单办法是：看能不能对表达式取地址，如果能，则为左值，否则为右值。所有的具名变量或对象都是左值，而匿名变量则是右值，比如，简单的赋值语句：
    
    int i = 0;
    
   在这条语句中，i 是左值，0 是字面量（！！！），就是右值。在上面的代码中，i 可以被引用，0 就不可以了。具体来说上面的表达式中等号右边的0是纯右值（prvalue），在C++11中所有的值必属于左值、将亡值、纯右值三者之一。比如，非引用返回的临时变量、运算表达式产生的临时变量、原始字面量（数字int i = 42  42就是字面量）和lambda表达式等都是纯右值。而将亡值是C++11新增的、与右值引用相关的表达式，比如，将要被移动的对象、T&&函数返回值、std::move返回值和转换为T&&的类型的转换函数的返回值等。关于将亡值我们会在后面介绍，先看下面的代码：

    int j = 5;
    auto f = []{return 5;};

   上面的代码中5是一个原始字面量， \[]{return 5;}是一个lambda表达式，都是属于纯右值，他们的特点是在表达式结束之后就销毁了。通过地行代码我们对右值有了一个初步的认识，知道了什么是右值，接下来再来看看第二行代码。

#### 第2行代码的故事 ####

    T&& k = getVar();
    
   第二行代码和第一行代码很像，只是相比第一行代码多了“&&”，他就是右值引用，我们知道左值引用是对左值的引用，那么，对应的，对右值的引用就是右值引用，而且右值是匿名变量，我们也只能通过引用的方式来获取右值。虽然第二行代码和第一行代码看起来差别不大，但是实际上语义的差别很大，这里，getVar()产生的临时值不会像第一行代码那样，在表达式结束之后就销毁了，而是会被“续命”，他的生命周期将会通过右值引用得以延续，和变量k的声明周期一样长。
右值引用的第一个特点
　　通过右值引用的声明，右值又“重获新生”，其生命周期与右值引用类型变量的生命周期一样长，只要该变量还活着，该右值临时量将会一直存活下去。让我们通过一个简单的例子来看看右值的生命周期。如代码清单1-1所示。
代码清单1-1 

    int g_constructCount=0;
    int g_copyConstructCount=0;
    int g_destructCount=0;
    struct A
    {
        A(){cout<<"construct: "<<++g_constructCount<<endl;}
        A(const A& a){cout<<"copy construct: "<<++g_copyConstructCount <<endl;}
        ~A(){cout<<"destruct: "<<++g_destructCount<<endl;}
    };
    A GetA(){
        return A();
    }
    int main() {
        A a = GetA();//GetA()是右值，没法被取地址，临时的 用引用去接 和元素值去接效果不同
        return 0;
    }
    
为了清楚的观察临时值，在编译时设置编译选项-fno-elide-constructors用来关闭返回值优化效果。输出结果：<br>  
construct : 1 <br>  
copy construct: 1 <br>  
destruct: 1 <br>  
copy construct: 2 <br>  
destruct: 2 <br>  
destruct: 3  <br>  
从上面的例子中可以看到，在没有返回值优化的情况下，拷贝构造函数调用了两次，一次是GetA()函数内部创建的对象返回出来构造一个临时对象产生的，另一次是在main函数中构造a对象产生的。第二次的destruct是因为临时对象在构造a对象之后就销毁了。如果开启返回值优化的话，输出结果将是：
construct: 1
destruct: 1
可以看到返回值优化将会把临时对象优化掉，但这不是c++标准，是各编译器的优化规则。我们在回到之前提到的可以通过右值引用来延长临时右值的生命周期，如果上面的代码中我们通过右值引用来绑定函数返回值的话，结果又会是什么样的呢？在编译时设置编译选项-fno-elide-constructors。

    int main() {
        A&& a = GetA();
        return 0;
    }
    
输出结果：<br> 
construct: 1 <br> 
copy construct: 1  <br> 
destruct: 1  <br> 
destruct: 2  <br> 
通过右值引用，比之前少了一次拷贝构造和一次析构，原因在于右值引用绑定了右值，让临时右值的生命周期延长了。我们可以利用这个特点做一些性能优化，即避免临时对象的拷贝构造和析构，事实上，在c++98/03中，通过常量左值引用也经常用来做性能优化。上面的代码改成：
    
    const A& a = GetA();
    
   输出的结果和右值引用一样，因为常量左值引用是一个“万能”的引用类型，可以接受左值、右值、常量左值和常量右值。需要注意的是普通的左值引用不能接受右值，比如这样的写法是不对的：A& a = GetA();上面的代码会报一个编译错误，因为非常量左值引用只能接受左值。
   右值引用的第二个特点
　右值引用独立于左值和右值。意思是右值引用类型的变量可能是左值也可能是右值。比如下面的例子：
    
    int&& var1 = 1;
 
 var1类型为右值引用，但var1本身是左值，因为具名变量都是左值。关于右值引用一个有意思的问题是：T&&是什么，一定是右值吗？让我们来看看下面的例子：

    template<typename T>
    void f(T&& t){}
    f(10); //t是右值
    int x = 10;
    f(x); //t是左值

从上面的代码中可以看到，T&&表示的值类型不确定，可能是左值又可能是右值，这一点看起来有点奇怪，这就是右值引用的一个特点。
右值引用的第三个特点
　　T&& t在发生自动类型推断的时候，它是未定的引用类型（universal references），如果被一个左值初始化，它就是一个左值；如果它被一个右值初始化，它就是一个右值，它是左值还是右值取决于它的初始化。我们再回过头看上面的代码，对于函数template<typename T>void f(T&& t)，当参数为右值10的时候，根据universal references的特点，t被一个右值初始化，那么t就是右值；当参数为左值x时，t被一个左值引用初始化，那么t就是一个左值。需要注意的是，仅仅是当发生自动类型推导（如函数模板的类型自动推导，或auto关键字）的时候，T&&才是universal references。再看看下面的例子：

        template<typename T>
        void f(T&& param); 
        template<typename T>
        class Test {
            Test(Test&& rhs); 
        };

上面的例子中，param是universal reference，rhs是Test&&右值引用，因为模版函数f发生了类型推断，而Test&&并没有发生类型推导，因为Test&&是确定的类型了。
　　正是因为右值引用可能是左值也可能是右值，依赖于初始化，并不是一下子就确定的特点，我们可以利用这一点做很多文章，比如后面要介绍的移动语义和完美转发。　　这里再提一下引用折叠，正是因为引入了右值引用，所以可能存在左值引用与右值引用和右值引用与右值引用的折叠，C++11确定了引用折叠的规则，规则是这样的：
* 所有的右值引用叠加到右值引用上仍然还是一个右值引用；
* 所有的其他引用类型之间的叠加都将变成左值引用。

#### 第3行代码的故事 ####

    T(T&& a) : m_val(val){ a.m_val=nullptr; }
    
   这行代码实际上来自于一个类的构造函数，构造函数的一个参数是一个右值引用，为什么将右值引用作为构造函数的参数呢？在解答这个问题之前我们先看一个例子。如代码清单1-2所示。代码清单1-2

    class A
    {
    public:
        A():m_ptr(new int(0)){cout << "construct" << endl;}
        A(const A& a):m_ptr(new int(*a.m_ptr)) //深拷贝的拷贝构造函数
        {
            cout << "copy construct" << endl;
        }
        ~A(){ cout << "destruct" << endl; delete m_ptr;}
    private:
        int* m_ptr;
    };
    A GetA()
    {
        return A();
    }
    int main() {
        A a = GetA();
        return 0;
    }
    
输出：<br> 
Construct  看不见的临时变量  <br> 
copy construct 看不见的临时变量 copy 给 返回传递 的临时变量 <br> 
destruct 看不见的临时变量析构  <br> 
copy construct返回传递 的临时变量 拷贝 给 a   <br> 
destruct返回传递 的临时变量 析构  <br>   
destruct a析构  <br> 
   这个例子很简单，一个带有堆内存的类，必须提供一个深拷贝拷贝构造函数，因为默认的拷贝构造函数是浅拷贝，会发生“指针悬挂”的问题。如果不提供深拷贝的拷贝构造函数，上面的测试代码将会发生错误（编译选项-fno-elide-constructors），第一次析构m_ptr是 看不见 临时变量 构造函数A():m_ptr(new int(0))的析构   拷贝构造函数的 内部的m_ptr将会被删除两次，一次是临时右值析构的时候删除一次，第二次外面构造的a对象释放时删除一次，而这两个对象的m_ptr是同一个指针，这就是所谓的指针悬挂问题。提供深拷贝的拷贝构造函数虽然可以保证正确，但是在有些时候会造成额外的性能损耗，因为有时候这种深拷贝是不必要的。比如下面的代码：
      ![](https://github.com/wshilaji/Cplusplus-Concurrency-In-Action/blob/master/images/stdc%2B%2B11/1.1.png)
      
上面代码中的GetA函数会返回临时变量，然后通过这个临时变量拷贝构造了一个新的对象a，临时变量在拷贝构造完成之后就销毁了，如果堆内存很大的话，那么，这个拷贝构造的代价会很大，带来了额外的性能损失。每次都会产生临时变量并造成额外的性能损失，有没有办法避免临时变量造成的性能损失呢？答案是肯定的，C++11已经有了解决方法，看看下面的代码。如代码清单1-3所示。

            int *p = new int(5); 
            这句是从堆上分配一个int型变量所占的字节内存，这个内存单元存放的整数值为5，然后让一个整形的指针变量p指向它的地址。
            释放方式：delete p;
            int *p = new int[5]; 

    class A
    {
    public:
        A() :m_ptr(new int(0)) {cout << "construct" << endl;}
        A(const A& a):m_ptr(new int(*a.m_ptr)) //深拷贝的拷贝构造函数
        {
            cout << "copy construct" << endl;
        }
        A(A&& a) :m_ptr(a.m_ptr)
        {
            a.m_ptr = nullptr;
            cout << "move construct" << endl;
        }
        ~A(){ cout << "destruct" << endl ;delete m_ptr;}
    private:
        int* m_ptr;
    };
    int main(){
        A a = GetA(); 
    } 

输出：
construct   <br> 
move construct  <br> 
destruct  <br> 
move construct  <br> 
destruct  <br> 
destruct   <br> 
代码清单1-3和1-2相比只多了一个构造函数，输出结果表明，并没有调用拷贝构造函数，因为GetA()是右值，只调用了move construct函数，让我们来看看这个move construct函数：

    A(A&& a) :m_ptr(a.m_ptr)
    {
        a.m_ptr = nullptr;
        cout << "move construct" << endl;
    }
    
   这个构造函数并没有做深拷贝，仅仅是将指针的所有者转移到了另外一个对象，同时，将参数对象a的指针置为空，这里仅仅是做了浅拷贝，因此，这个构造函数避免了临时变量的深拷贝问题。
　　上面这个函数其实就是移动（拷贝）构造函数，他的参数是一个右值引用类型，这里的A&&表示右值，为什么？前面已经提到，这里没有发生类型推断，是确定的右值引用类型。为什么会匹配到这个构造函数？因为这个构造函数只能接受右值参数，而函数返回值是右值就是getA()是右值,（函数的返回值是临时的，右值，不能取s地址），所以就会匹配到这个构造函数。这里的A&&可以看作是临时值的标识，对于临时值我们仅仅需要做浅拷贝即可这里的浅拷贝只有这一步<font color=Crimson> m_ptr(a.m_ptr) -----> m_ptr = a.m_ptr</font>,然后a.m_pt开辟的内存也成为了m_ptr的了，即m_ptr也指向a.m_pt开辟的内存，最后要拷贝的对象a.m_ptr = nullptr，无需再做深拷贝，从而解决了前面提到的临时变量拷贝构造产生的性能损失的问题。这就是所谓的移动语义，右值引用的一个重要作用是用来支持移动语义的。 
　　需要注意的一个细节是，我们提供移动构造函数的同时也会提供一个拷贝构造函数，以防止移动不成功的时候还能拷贝构造，使我们的代码更安全。我们知道移动语义是通过右值引用来匹配临时值的，那么，普通的左值是否也能借助移动语义来优化性能呢，那该怎么做呢？事实上C++11为了解决这个问题，提供了std::move方法来将左值转换为右值，从而方便应用移动语义。move是将对象资源的所有权从一个对象转移到另一个对象，只是转移，没有内存的拷贝，这就是所谓的move语义。如图1-1所示是深拷贝和move的区别。  
        ![图1-1](https://github.com/wshilaji/Cplusplus-Concurrency-In-Action/blob/master/images/stdc%2B%2B11/1.1.1.jpg)        
再看看下面的例子：

    {
        std::list< std::string> tokens;
        //省略初始化...
        std::list< std::string> t = tokens; //这里存在拷贝 
    }
    std::list< std::string> tokens;
    std::list< std::string> t = std::move(tokens);  //这里没有拷贝 
    
   如果不用std::move，拷贝的代价很大，性能较低。使用move几乎没有任何代价，只是转换了资源的所有权。他实际上将左值变成右值引用，然后应用移动语义，调用移动构造函数，就避免了拷贝，提高了程序性能。如果一个对象内部有较大的对内存或者动态数组时，很有必要写move语义的拷贝构造函数和赋值函数，避免无谓的深拷贝，以提高性能。事实上，C++11中所有的容器都实现了移动语义，方便我们做性能优化。
   这里也要注意对move语义的误解，move实际上它并不能移动任何东西，它唯一的功能是将一个左值强制转换为一个右值引用。移动东西的还是移动构造函数中浅拷贝m_ptr = a.m_ptr。如果是一些基本类型比如int和char[10]定长数组等类型，使用move的话仍然会发生拷贝（因为没有对应的移动构造函数）。所以，move对于含资源（堆内存或句柄）的对象来说更有意义。
  
#### 第4行代码的故事 ####

    template <typename T>void f(T&& val){ foo(std::forward<T>(val)); }

　C++11之前调用模板函数时，存在一个比较头疼的问题，如何正确的传递参数。比如： 

    template <typename T>
    void forwardValue(T& val)
    {
        processValue(val); //右值参数会变成左值 
    }
    template <typename T>
    void forwardValue(const T& val)
    {
        processValue(val); //参数都变成常量左值引用了 
    }
    
都不能按照参数的本来的类型进行转发。C++11引入了完美转发：在函数模板中，完全依照模板的参数的类型（即保持参数的左值、右值特征），将参数传递给函数模板中调用的另外一个函数。C++11中的std::forward正是做这个事情的，他会按照参数的实际类型进行转发。看下面的例子：

    void processValue(int& a){ cout << "lvalue" << endl; }
    void processValue(int&& a){ cout << "rvalue" << endl; }
    template <typename T>
    void forwardValue(T&& val)
    {
        processValue(std::forward<T>(val)); //照参数本来的类型进行转发。
    }
    void Testdelcl()
    {
        int i = 0;
        forwardValue(i); //传入左值 
        forwardValue(0);//传入右值 
    }
输出：  <br> 
lvaue   <br> 
rvalue   <br> 
右值引用T&&是一个universal references，可以接受左值或者右值，正是这个特性让他适合作为一个参数的路由，然后再通过std::forward按照参数的实际类型去匹配对应的重载函数，最终实现完美转发。我们可以结合完美转发和移动语义来实现一个泛型的工厂函数，这个工厂函数可以创建所有类型的对象。具体实现如下：

    template<typename…  Args>
    T* Instance(Args&&… args)
    {
        return new T(std::forward<Args >(args)…);
    }

这个工厂函数的参数是右值引用类型，内部使用std::forward按照参数的实际类型进行转发，如果参数的实际类型是右值，那么创建的时候会自动匹配移动构造，如果是左值则会匹配拷贝构造。
 
std::move 函数源码如下（可在 VS 工具中右键函数名转到定义查看源码）：

// TEMPLATE FUNCTION move
template<class _Ty> inline
	constexpr typename remove_reference<_Ty>::type&&
		move(_Ty&& _Arg) _NOEXCEPT
	{	// forward _Arg as movable
    	return (static_cast<typename remove_reference<_Ty>::type&&>(_Arg));
	}
代码中 std::remove_reference<T>::type 作用是脱去引用剩下类型本身，假设 T 为 X&、X&&，则 std::remove_reference<T>::type 为 X；

注意 std::move 的主要作用是将一个左值转换为右值，因此如果某个函数代码和上述相近，则该函数功能和 std::move 一致。

### 1.2 泛化的常量表达式 constexpr ###

C++ 本来就已具备常量表示式(constant expression)的概念。C++11引进关键字 constexpr 允许用户保证函数或是对象建构式是编译期常量。

constexpr 可以修饰变量，函数，和类的构造函数。

当 constexpr 修饰变量时，应满足：

- 如果该变量是某个类的对象，则它应该立即被构造，如果是基本类型，则它应该被立即赋值。
- 构造函数的参数或者赋给该变量的值必须是字面常量，constexpr 变量或 constexpr 函数。
- 如果是使用构造函数创建该变量（无论显式构造或者隐式构造），构造函数必须满足 constexpr 特性。

当 constexpr 修饰函数时，应满足：

- 该函数不能是虚函数。
- return 返回值应该是字面类型的常量。
- 该函数每个参数必须是字面类型常量。
- 函数体只能包含：
    1. null 语句。
    2. [[../static_assert|static_assert]] 语句。
    3. typedef 声明或 模板别名声明（但该模板别名声明不定义类类型与枚举类型）。
    4. using 声明。
    5. using 指导语句。
    6. 只能存在唯一的 return 语句，并且 return 语句只能包含字面常量，constexpr 变量或 constexpr 函数。
-很多人都把constexpr和const相比较。

其实，const并不能代表“常量”，它仅仅是对变量的一个修饰，告诉编译器这个变量只能被初始化，且不能被直接修改（实际上可以通过堆栈溢出等方式修改）。而这个变量的值，可以在运行时也可以在编译时指定。
constexpr可以用来修饰变量、函数、构造函数。一旦以上任何元素被constexpr修饰，那么等于说是告诉编译器 “请大胆地将我看成编译时就能得出常量值的表达式去优化我”。

    const int func() {
        return 10;
    }
    main(){
      int arr[func()];
    }
    //error : 函数调用在常量表达式中必须具有常量值
    
对于func() ，胆小的编译器并没有足够的胆量去做编译期优化，哪怕函数体就一句return 字面值;
而

    constexpr func() {
        return 10;
    }
    main(){
      int arr[func()];
    }
    //编译通过
    
则编译通过。编译期大胆地将func()做了优化，在编译期就确定了func计算出的值10而无需等到运行时再去计算。这就是constexpr的第一个作用：给编译器足够的信心在编译期去做被constexpr修饰的表达式的优化。
constexpr还有另外一个特性，虽然它本身的作用之一就是希望程序员能给编译器做优化的信心，但它却猜到了自己可能会被程序员欺骗，而编译器并不会对此“恼羞成怒”中止编译，如：

    constexpr int func(const int n){
      return 10+n;
    }
    main(){
     const  int i = cin.get();
     cout<<func(i);
    }//编译通过
    
程序员告诉编译器尽管信心十足地把func当做是编译期就能计算出值的程式，但却欺骗了它，程序员最终并没有传递一个常量字面值到该函数。没有被编译器中止编译并报错的原因在于编译器并没有100%相信程序员，当其检测到func的参数是一个常量字面值的时候，编译器才会去对其做优化，否则，依然会将计算任务留给运行时。
基于这个特性，constexpr还可以被用来实现编译期的type traits，比如STL中的is_const的完整实现：

    template<class _Ty,
        _Ty _Val>
    struct integral_constant
    {   // convenient template for integral constant types
        static constexpr _Ty value = _Val;

        typedef _Ty value_type;
        typedef integral_constant<_Ty, _Val> type;

        constexpr operator value_type() const noexcept
        {   // return stored value
            return (value);
        }

        constexpr value_type operator()() const noexcept
        {   // return stored value
            return (value);
        }
    };

    typedef integral_constant<bool, true> true_type;
    typedef integral_constant<bool, false> false_type;

    template<class _Ty>
    struct is_const
        : false_type
    {   // determine whether _Ty is const qualified
    };

    template<class _Ty>
    struct is_const<const _Ty>
        : true_type
    {   // determine whether _Ty is const qualified
    };
    int main() {

        static_assert(is_const<int>::value,"error");//error
        static_assert(is_const<const int>::value, "error");//ok
        return 0;
    }

当 constexpr 修饰类构造函数时，应满足：

- 构造函数的每个参数必须是字面类型常量。
- 该类不能有虚基类。
- 构造函数体必须满足以下条件：

    1. null 语句。
    2. [[../static_assert|static_assert]] 语句。
    3. typedef 声明或 模板别名声明（但该模板别名声明不定义类类型与枚举类型）。
    4. using 声明。
    5. using 指导语句。

- 构造函数中不能有 try-语句块。
- 该类的每个基类和非静态成员变量必须是初始化。
- 每一个隐式转换必须是常量表达式。

请看下例([参考](http://en.cppreference.com/w/cpp/language/constexpr))：

    #include <iostream>
    #include <stdexcept>
     
    // constexpr functions use recursion rather than iteration
    constexpr int factorial(int n)
    {
        return n <= 1 ? 1 : (n * factorial(n-1));
    }
     
    // literal class
    class conststr {
        const char * p;
        std::size_t sz;
     public:
        template<std::size_t N>
        constexpr conststr(const char(&a)[N]) : p(a), sz(N-1) {}
        // constexpr functions signal errors by throwing exceptions from operator ?:
        constexpr char operator[](std::size_t n) const {
            return n < sz ? p[n] : throw std::out_of_range("");
        }
        constexpr std::size_t size() const { return sz; }
    };
     
    constexpr std::size_t countlower(conststr s, std::size_t n = 0,
                                                 std::size_t c = 0) {
        return n == s.size() ? c :
               s[n] >= 'a' && s[n] <= 'z' ? countlower(s, n+1, c+1) :
               countlower(s, n+1, c);
    }
     
    // output function that requires a compile-time constant, for testing
    template<int n> struct constN {
        constN() { std::cout << n << '\n'; }
    };
     
    int main()
    {
        std::cout << "4! = " ;
        constN<factorial(4)> out1; // computed at compile time
     
        volatile int k = 8; // disallow optimization using volatile
        std::cout << k << "! = " << factorial(k) << '\n'; // computed at run time
     
        std::cout << "Number of lowercase letters in \"Hello, world!\" is ";
        constN<countlower("Hello, world!")> out2; // implicitly converted to conststr
    }
    
### 1.3 static ###
__变量__  
1. 局部变量  
普通局部变量是再熟悉不过的变量了，在任何一个函数内部定义的变量（不加static修饰符）都属于这个范畴。编译器一般不对普通局部变量进行初始化，也就是说它的值在初始时是不确定的，除非对其显式赋值。

普通局部变量存储于进程栈空间，使用完毕会立即释放。   

静态局部变量使用static修饰符定义，即使在声明时未赋初值，编译器也会把它初始化为0。且静态局部变量存储于进程的全局数据区，即使函数返回，它的值也会保持不变。    

        void fn(void)
        {
            int n = 10;
            printf("n=%d\n", n);
            n++;
            printf("n++=%d\n", n);
        }
        void fn_static(void)
        {
            static int n = 10;
            printf("static n=%d\n", n);
            n++;
            printf("n++=%d\n", n);
        }
        int main(void)
        {
            fn();
            printf("--------------------\n");
            fn_static();
            printf("--------------------\n");
            fn();
            printf("--------------------\n");
            fn_static();
            return 0;
        }
-> % ./a.out 
n=10    
n++=11   
static n=10  
n++=11  
n=10  
n++=11  
static n=11  
n++=12  

__普通全局变量__ 对整个工程可见，其他文件可以使用extern外部声明后直接使用。也就是说其他文件不能再定义一个与其相同名字的变量了（否则编译器会认为它们是同一个变量）。静态全局变量仅对当前文件可见，其他文件不可访问，其他文件可以定义与其同名的变量，两者互不影响。
__函数__ 的使用方式与全局变量类似，在函数的返回类型前加上static，就是静态函数。其特性如下：
静态函数只能在声明它的文件中可见，其他文件不能引用该函数
不同的文件可以使用相同名字的静态函数，互不影响
__类中静态成员函数__   

        class Test
        {
        private:
            static int cCount;
        public:
            Test()  {cCount++;}
            ~Test() {  --cCount;  }
            int getCount() {  return cCount;}
        };
        int Test::cCount = 0;
        Test gTest;
        int main()
        {
            Test t1;
            Test t2;
            printf("count = %d\n", gTest.getCount());
            printf("count = %d\n", t1.getCount());
            printf("count = %d\n", t2.getCount());
            Test* pt = new Test();
            printf("count = %d\n", pt->getCount());
            delete pt;
            printf("count = %d\n", gTest.getCount());
            return 0;
        }
很明显，我们获取对象的数目是需要调用getCount这个函数，每次调用都需要某一个对象来调用，比如：gTest.getCount()，那么问题来了，如果整个程序的对象的个数为0呢？那么岂不是没法调用getCount这个函数，也就没法知道对象的个数（别人是不知道你的对象的个数为0的）？   

可不可以这样呢？我把静态成员变量cCount变为public，然后直接让Test（作用域调用）这个类来调用？先上一波代码看看：

        class Test
        {
        public:
            static int cCount;
        public:
            Test()
            {
                cCount++;
            }
            ~Test()
            {
                --cCount;
            }
            int getCount()
            {
                return cCount;
            }
        };
        int Test::cCount = 0;
        int main()
        {
            printf("count = %d\n", Test::cCount);    
            //Test::cCount = 1000;    
            //printf("count = %d\n", Test::cCount);    
            return 0;
        }
此代码与上面的代码的不同的是cCount变量变为public便于Test（作用域访问）类调用，没有任何的对象了，那么编译运行，输出结果应该是：
count = 0说明这样做，符合了我们的需求了。但是，不要高兴的太早了，我们这样真的可以么？加入我把代码中的被注释的那两行加上（下面这两行）：   

     Test::cCount = 1000;    
    printf("count = %d\n", Test::cCount);   
    
再次运行会发生什么呢？我们编译运行，结果输出为：   
count = 0
 count = 1000   
 那么多问题来了，客户现在懵逼了，他会以为对象的个数有1000个，所以，这也是一个大的Bug啊，从本上讲，这样做也是无法满足客户的需求的，而且将cCount变为public，本身就是无法保证静态成员变量的安全性，我们需要什么？      
 
__不依赖对象就可以访问静态成员变量__,    
必须保证静态成员变量的安全性   
方便快捷得获取静态成员变量的值  
那么静态成员函数就要出马了：	
静态成员函数的定义：  
直接通过static关键字修饰成员函数即可   

为了便于理解，我们先上一段代码来理解一下静态成员函数的性质：  

       class Test
        {
        private:
            static int cCount;
        public:
            Test()
            {
                cCount++;
            }
            ~Test()
            {
                --cCount;
            }
            static int GetCount()
            {
                return cCount;
            }
        };

        int Test::cCount = 0;

        int main()
        {
            printf("count = %d\n", Test::GetCount());

            Test t1;
            Test t2;

            printf("count = %d\n", t1.GetCount());
            printf("count = %d\n", t2.GetCount());

            Test* pt = new Test();

            printf("count = %d\n", pt->GetCount());

            delete pt;

            printf("count = %d\n", Test::GetCount());

            return 0;
        }

总结：

静态成员函数是类中的特殊的成员函数
__静态成员函数没有隐藏的this指针__ 
__静态成员函数可以通过类名直接访问__ 
静态成员函数可以通过对象访问
__静态成员函数只能直接访问静态成员变量（函数），而不能直接访问普通成员变量（函数）__ 

### 1.4 对 POD 类型定义的修正 ###

> 本小节主要来源于[维基百科C++11](http://zh.wikipedia.org/zh-cn/C%2B%2B11)词条中的《[对POD定义的修正](http://zh.wikipedia.org/zh-cn/C%2B%2B11#.E5.B0.8DPOD.E5.AE.9A.E7.BE.A9.E7.9A.84.E4.BF.AE.E6.AD.A3)》：

在标准C++，一个结构(struct)为了能够被当成 POD，必须遵守几条规则。有很好的理由使我们想让大量的类型符合这种定义，符合这种定义的类型能够允许产生与C兼容的对象布局(object layout)。然而，C++03 的规则太严苛了。

C++11 将会放宽关于 POD 的定义。

当 class/struct 是极简的(`trivial`)、属于标准布局(standard-layout)，以及他的所有非静态(non-static)成员都是 POD 时，会被视为 POD。

一个极简的类型或结构符合以下定义：

- 极简的默认建构式。这可以使用默认建构式语法，例如 `SomeConstructor() = default;`
- 极简的复制建构式，可使用默认语法(default syntax)
- 极简的赋值运算符，可使用默认语法(default syntax)
- 极简的析构式，不可以是虚拟的(virtual)

一个标准布局(standard-layout)的类型或结构符合以下定义：

- 只有非静态的(non-static)数据成员，且这些成员也是符合标准布局的类型
- 对所有non-static成员有相同的访问控制(public, private, protected)
- 没有虚函数
- 没有虚拟基类
- 只有符合标准布局的基类
- 没有和第一个定义的non-static成员相同类型的基类
- 若非没有带有non-static成员的基类，就是最底层(继承最末位)的类型没有non-static数据成员而且至多一个带有non-static成员的基类。基本上，在该类型的继承体系中只会有一个类型带有non-static成员。
