## __5. C++ 标准库的变更__ ##

### 5.1 标准库组件上的升级 ###

### 5.2 多线程支持 ###

### 5.3 元组(tuple)类型 ###

### 5.4 散列表(hash table) ###

### 5.5 正则表达式 ###

参考[Python正则表达式操作指南](https://github.com/wshilaji/Cplusplus-Concurrency-In-Action/blob/master/zh/appendix%20C%2B%2B11%20standards/Python%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%93%8D%E4%BD%9C%E6%8C%87%E5%8D%97%20-%20Ubuntu%E4%B8%AD%E6%96%87.pdf)

### 5.6 通用智能指针(smarter pointer) ###

也有std::auto_ptr。它非常类似于作用域指针，不同之处在于它还具有“特殊”危险功能，可被复制-还会意外地转移所有权。它在C ++ 11中已弃用，在C ++ 17中已删除，因此您不应该使用它。

智能指针是包装“原始”（或“裸露”）C ++指针的类，用于管理所指向对象的生命周期。没有单一的智能指针类型，但是它们都尝试以一种实用的方式抽象一个原始指针。
智能指针应优于原始指针。如果您觉得需要使用指针（首先考虑是否确实需要使用指针），则通常希望使用智能指针，因为这可以减轻原始指针的许多问题，主要是忘记删除对象和泄漏内存。

使用原始指针，程序员必须在不再有用时显式销毁该对象。

        // Need to create the object to achieve some goal
        MyObject* ptr = new MyObject(); 
        ptr->DoSomething(); // Use the object in some way
        delete ptr; // Destroy the object. Done with it.
        // Wait, what if DoSomething() raises an exception...?
通过比较，智能指针定义了有关销毁对象的时间的策略。您仍然必须创建对象，但是不必担心销毁它。

        SomeSmartPtr<MyObject> ptr(new MyObject());
        ptr->DoSomething(); // Use the object in some way.
        // Destruction of the object happens, depending on the policy the smart pointer class uses.
        // Destruction would happen even if DoSomething() raises an exception
使用的最简单策略涉及智能指针包装器对象的范围，例如由boost::scoped_ptr或实现std::unique_ptr。

        void f()
        {
            {
               std::unique_ptr<MyObject> ptr(new MyObject());
               ptr->DoSomethingUseful();
            } // ptr goes out of scope -- 
              // the MyObject is automatically destroyed.

            // ptr->Oops(); // Compile error: "ptr" not defined
                            // since it is no longer in scope.
        }
请注意，std::unique_ptr实例无法复制。这样可以防止多次（不正确）删除指针。但是，您可以将对其的引用传递给您调用的其他函数。

std::unique_ptr如果要将对象的生存期绑定到特定代码块，或者将其作为成员数据嵌入另一个对象的生存期，则s很有用。该对象将一直存在，直到退出包含代码的块，或者直到包含对象本身被销毁为止。

更复杂的智能指针策略涉及对指针进行引用计数。这确实允许复制指针。当该对象的最后一个“引用”被销毁时，该对象将被删除。此政策由boost::shared_ptr和实施std::shared_ptr。

        void f()
        {
            typedef std::shared_ptr<MyObject> MyObjectPtr; // nice short alias
            MyObjectPtr p1; // Empty

            {
                MyObjectPtr p2(new MyObject());
                // There is now one "reference" to the created object
                p1 = p2; // Copy the pointer.
                // There are now two references to the object.
            } // p2 is destroyed, leaving one reference to the object.
        } // p1 is destroyed, leaving a reference count of zero. 
          // The object is deleted.
当对象的生存期非常复杂，并且不直接与代码的特定部分或另一个对象绑定时，引用计数的指针非常有用。

引用计数的指针有一个缺点-可能创建悬挂的引用：

        // Create the smart pointer on the heap
        MyObjectPtr* pp = new MyObjectPtr(new MyObject())
        // Hmm, we forgot to destroy the smart pointer,
        // because of that, the object is never destroyed!
另一种可能性是创建循环引用：

        struct Owner {
           std::shared_ptr<Owner> other;
        };

        std::shared_ptr<Owner> p1 (new Owner());
        std::shared_ptr<Owner> p2 (new Owner());
        p1->other = p2; // p1 references p2
        p2->other = p1; // p2 references p1

        // Oops, the reference count of of p1 and p2 never goes to zero!
        // The objects are never destroyed!        
#### __unique_ptr__ ####

        //Specialization for arrays:
        template <class T, class D>
        class unique_ptr<T[], D> {
        public:
            typedef pointer;
            typedef T element_type;
            typedef D deleter_type;
              ..... 
            unique_ptr(unique_ptr&& u) noexcept;           
            ~unique_ptr();
            unique_ptr& operator=(unique_ptr&& u) noexcept;nique_ptr& operator=(const unique_ptr&) = delete;
            unique_ptr& operator=(nullptr_t) noexcept;T& operator[](size_t i) const;
            //get()返回 stored_ptr。
            pointer get() const noexcept;
            deleter_type& get_deleter() noexcept;const deleter_type& get_deleter() const noexcept;
            explicit operator bool() const noexcept;
            pointer release() noexcept;
            //得指针参数的所有权，然后删除原始存储的指针。 如果新指针与原始存储指针相同 reset删除指针并将存储的指针设置为nullptr。
            void reset(pointer p = pointer()) noexcept;void reset(nullptr_t = nullptr) noexcept;
            ...
            //交换两个 unique_ptr 对象之间的指针。
            void swap(unique_ptr& u) noexcept;  // disable copy from lvalue unique_ptr(const unique_ptr&) = delete;
            u
        };

~~Differences between unique_ptr and shared_ptr ?????~~
__unique_ptr and shared_ptr__  这两个类都是智能指针，这意味着当不再可以引用该对象时，它们将自动（在大多数情况下）释放它们所指向的对象。两者之间的区别在于每种类型可以引用一个资源有多少个不同的指针。

    std::unique_ptr<int> fPtr2(new int(4));///这里不是开辟内存，开辟内存是new int[4]  这里new int(4)是初始化，初始化指针指向4  new int(0) 指向Null

unique_ptr开销很小，是首选的轻质智能指针。其类型为 __template <typename D, typename Deleter> class unique_ptr__;，因此它取决于两个模板参数。unique_ptr也是auto_ptr想在旧的C ++中实现的功能，但由于该语言的局限性而无法实现。使用时unique_ptr，最多只能有一个unique_ptr指向任何一种资源中的a。当unique_ptr被破坏，资源被自动收回。由于unique_ptr任何资源只能有一个，因此任何尝试复制资源a的unique_ptr都会导致编译时错误。它不可复制，但可以移动。将指针包装到中unique_ptr时，不能有的多个副本unique_ptr。例如，此代码是非法的：

    //unique_ptr& operator=(const unique_ptr&) = delete;
    unique_ptr<T> myPtr(new T);       // Okay
    unique_ptr<T> myOtherPtr = myPtr; // Error: Can't copy unique_ptr
    
![unique_ptr](https://github.com/wshilaji/Cplusplus-Concurrency-In-Action/blob/master/zh/appendix%20C%2B%2B11%20standards/pic/5.4.1.png)
然而，要更改唯一ptr指向的对象，请使用move语义：

    //unique_ptr(unique_ptr&& u) noexcept;移动构造函数	
    unique_ptr<T> myPtr(new T);                  // Okay
    unique_ptr<T> myOtherPtr = std::move(myPtr); // Okay, resource now stored in myOtherPtr
    
同样，您可以执行以下操作：

     unique_ptr<T> MyFunction() {
        unique_ptr<T> myPtr(/* ... */);
        /* ... */
        return myPtr;
    }

这种idiom的意思是“我正在向您返回托管资源managed resource。如果您未明确捕获capture返回值，则将清除该资源cleaned up。如果您这样做了，那么您现在对该资源具有独占所有权exclusive ownership of resource” 这样，~~您可以认为unique_ptr是的更安全，更好的替代auto_ptr。~~

__unique_ptr在删除器__ 方面可能会有些麻烦。shared_ptr只要它是使用创建的，它将始终做“正确的事情” make_shared。但是，如果创建了 __unique_ptr<Derived>__ ，然后 __将其转换为unique_ptr<Base>__ ，并且如果Derived是virtual而Base不是，则指针将通过错误的类型删除，并且可能存在未定义的行为。可以在中使用适当的 __deleteer-type__ 在unique_ptr<T, DeleterType>中修复此问题，但默认设置是使用高风险版本，因为它效率更高。
>
        
#### unique_ptr使用场景 ####

##### 1、为动态申请的资源提供异常安全保证 #####

        void Func(){
            int *p = new int(5);
            // ...（可能会抛出异常）
            delete p;
        }        
这是我们传统的写法：当我们动态申请内存后，有可能我们接下来的代码由于抛出异常或者提前退出（if语句）而没有执行delete操作。解决的方法是使用unique_ptr来管理动态内存，只要unique_ptr指针创建成功，其析构函数都会被调用。确保动态资源被释放。

        void Func(){
            unique_ptr<int> p(new int(5));
            // ...（可能会抛出异常）
        }        
##### 2、返回函数内动态申请资源的所有权 #####   [同上]();
##### 3、在容器中保存指针 #####

      vector<unique_ptr<int>> vec;
      unique_ptr<int> p(new int(5));
      vec.push_back(std::move(p));    // 使用移动语义
##### 4、管理动态数组 #####
        unique_ptr<int[]> p(new int[5] {1, 2, 3, 4, 5});
        p[0] = 0;   // 重载了operator[]
##### 5、作为auto_ptr的替代品 #####

#### __shared_ptr__ ####

        shared_ptr::get
        shared_ptr::operator->
        shared_ptr::owner_before
        shared_ptr::use_count
        
          std::shared_ptr<int> sp1(new int(5));
          std::cout << "sp0.get() == 0 == " << std::boolalpha<< (sp0.get() == 0) << std::endl;//sp0.get() == 0 == true
          std::cout << "*sp1.get() == " << *sp1.get() << std::endl;//*sp1.get() == 5
__shared_ptr__ 是共享所有权的智能指针。都是copyable和movable。资源可由多个 shared_ptr 对象拥有；当拥有特定资源的最后一个 shared_ptr 对象被销毁后，资源将释放。shared_ptr 对象有效保留一个指向其拥有的资源的指针或保留一个 null 指针。在重新分配或重置资源后，shared_ptr 将停止拥有该资源。拥有资源的最后一个智能指针一旦超出范围，资源将被释放。在内部，shared_ptr还有很多事情要做：有一个引用计数，该计数被原子地更新以允许在并发代码中使用。另外，还有大量的分配工作，一个分配用于内部簿记“reference control block”，另一个分配给实际的成员对象（通常）。这还有另一个很大的区别：共享指针类型始终为 __template <typename T> class shared_ptr__ ;，尽管您可以使用自定义删除器和自定义分配器对其进行初始化，但是共享指针类型始终如此。 

        #include <memory>
        T a ; 
             //shared_ptr<T> shptr(new T) ; not recommended but works 
             shared_ptr<T> shptr = make_shared<T>(); // faster + exception safe
             
             std::cout << shptr.use_count() ; // 1 //  gives the number of "things " pointing to it. 
             T * temp = shptr.get(); // gives a pointer to object
             // shared_pointer used like a regular pointer to call member functions
              shptr->memFn();
             (*shptr).memFn(); 
            //
             shptr.reset() ; // frees the object pointed to be the ptr 
             shptr = nullptr ; // frees the object 
             shptr = make_shared<T>() ; // frees the original object and points to new object
             
使用引用计数实现，以跟踪指针指向的对象有多少个“事物”。当此计数变为0时，将自动删除对象，即，当所有指向该对象的share_ptr超出范围时，删除对象。这消除了必须删除使用new分配的对象的麻烦。 

当从类型 shared_ptr<T> 的资源指针或 G* 中构造 shared_ptr<G> 对象时，指针类型 G* 必须可转换为 T*。 如果不是这样，则代码将不进行编译
         
        class F {};
        class G : public F {};
        shared_ptr<G> sp0(new G);   // okay, template parameter形参 G and argument实参 G*
        shared_ptr<G> sp1(sp0);     // okay, template parameter G and argument shared_ptr<G>
        shared_ptr<F> sp2(new G);   // okay, G* convertible to F*      
        //shared_ptr<int> sp5(new G); // error, G* not convertible to int*
        //shared_ptr<int> sp6(sp2);   // error, template parameter int and argument shared_ptr<F>
        
 当对象的生存期非常复杂，并且不直接与代码的特定部分或另一个对象绑定时，引用计数的指针非常有用。
 
 引用计数的指针有一个缺点-可能创建悬挂的引用：       
  
__weak_ptr__
 帮助处理使用共享指针时出现的循环引用如果您有两个共享指针指向的两个对象，并且有一个内部共享指针指向彼此的共享指针，则将有一个循环引用，而该对象不会当共享指针超出范围时被删除。要解决此问题，请将内部成员从shared_ptr更改为weak_ptr。注意：要使用弱指针指向的元素，请使用lock（），这将返回一个weak_ptr。
 
        T a ; 
        shared_ptr<T> shr = make_shared<T>() ; 
        weak_ptr<T> wk = shr ; // initialize a weak_ptr from a shared_ptr 
        wk.lock()->memFn() ; // use lock to get a shared_ptr 
        //   ^^^ Can lead to exception if the shared ptr has gone out of scope
        if(!wk.expired()) wk.lock()->memFn() ;
        // Check if shared ptr has gone out of scope before access
### 5.7 可扩展的随机数功能 ###

### 5.8 包装引用 ###

### 5.9 多态函数对象包装器 ###

### 5.10 用于元编程的类型属性 ###

### 5.11 用于计算函数对象返回类型的统一方法 ###

### 5.12 std::chrono库详解 ###
