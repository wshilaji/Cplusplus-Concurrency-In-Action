## __5. C++ 标准库的变更__ ##

### 5.1 标准库组件上的升级 ###

### 5.2 多线程支持 ###

### 5.3 元组(tuple)类型 ###

### 5.4 散列表(hash table) ###

### 5.5 正则表达式 ###

参考[Python正则表达式操作指南](https://github.com/wshilaji/Cplusplus-Concurrency-In-Action/blob/master/zh/appendix%20C%2B%2B11%20standards/Python%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%93%8D%E4%BD%9C%E6%8C%87%E5%8D%97%20-%20Ubuntu%E4%B8%AD%E6%96%87.pdf)

### 5.6 通用智能指针 ###

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
            unique_ptr& operator=(unique_ptr&& u) noexcept;   
            nique_ptr& operator=(const unique_ptr&) = delete;
            unique_ptr& operator=(nullptr_t) noexcept;
            T& operator[](size_t i) const;
            //get()返回 stored_ptr。
            pointer get() const noexcept;
            deleter_type& get_deleter() noexcept;
            const deleter_type& get_deleter() const noexcept;
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
    
[unique_ptr](https://github.com/wshilaji/Cplusplus-Concurrency-In-Action/blob/master/zh/appendix%20C%2B%2B11%20standards/pic/5.4.1.png)
然而，unique_ptr可移动使用新的移动语义：

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
__shared_ptr__ 是共享所有权的智能指针。都是copyable和movable。多个智能指针实例可以拥有相同的资源。拥有资源的最后一个智能指针一旦超出范围，资源将被释放。在内部，shared_ptr还有很多事情要做：有一个引用计数，该计数被原子地更新以允许在并发代码中使用。另外，还有大量的分配工作，一个分配用于内部簿记“reference control block”，另一个分配给实际的成员对象（通常）。这还有另一个很大的区别：共享指针类型始终为 __template <typename T> class shared_ptr__ ;，尽管您可以使用自定义删除器和自定义分配器对其进行初始化，但是共享指针类型始终如此。

### 5.7 可扩展的随机数功能 ###

### 5.8 包装引用 ###

### 5.9 多态函数对象包装器 ###

### 5.10 用于元编程的类型属性 ###

### 5.11 用于计算函数对象返回类型的统一方法 ###

### 5.12 std::chrono库详解 ###
