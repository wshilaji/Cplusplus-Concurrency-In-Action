## __4. C++ 标准库的变更__ ##

### 4.1 变长参数模板 ###

### 4.2 新的字符串字面值 ###

### 4.3 用户自定义的字面值 ###

### 4.4 多线程内存模型 ###

### 4.5 线程本地存储 ###

### 4.6 特殊成员函数 ###

 一个类，只有数据成员时
 
        class DataOnly {
        private:
            int  data_;
        };
        
C++98 编译器会隐式的产生四个函数：缺省构造函数，析构函数，拷贝构造函数 和 拷贝赋值算子，它们称为特殊成员函数 (special member function) 在 C++11 中，除了上面四个外，特殊成员函数还有两个：移动构造函数 和 移动赋值算子。

        class DataOnly {
        public:
            DataOnly ()                  // default constructor
            ~DataOnly ()                 // destructor
            DataOnly (const DataOnly & rhs)         　 　  // copy constructor
            DataOnly & operator=(const DataOnly & rhs)    // copy assignment operator
            DataOnly (const DataOnly && rhs)         // C++11, move constructor
            DataOnly & operator=(DataOnly && rhs)    // C++11, move assignment operator
        };


### 4.7 显式地使用或禁用某些特殊成员函数（构造函数，拷贝构造，赋值操作符，析构等） ###

#### final 说明符 ####

可以使用最终关键字来指定不能在派生类中重写的虚函数。 您还可以使用它指定无法继承的类。下面的示例使用最终关键字来指定，不能重写虚函数。

    class BaseClass
    {
        virtual void func() final;
    };
    class DerivedClass: public BaseClass
    {
        virtual void func(); // compiler error: attempting to
                             // override a final function
    };

下面的示例使用最终关键字指定无法继承类。

    class BaseClass final{ };
    class DerivedClass: public BaseClass // compiler error: BaseClass is
                                         // marked as non-inheritable
    {
    };
    
#### override 说明符 ####

可以使用重写关键字来指定重写基类中的虚函数的函数的成员。使用重写以帮助防止在代码中的意外的继承行为。 下面的示例演示的位置，而无需使用重写，派生类的成员函数的行为可能不适用。编译器不会发出此代码的任何错误。

    class BaseClass
    {
        virtual void funcA();
        virtual void funcB() const;
        virtual void funcC(int = 0);
        void funcD();
    };
    class DerivedClass: public BaseClass
    {
        virtual void funcA(); // ok, works as intended
        virtual void funcB(); // DerivedClass::funcB() is non-const, so it does not
                              // override BaseClass::funcB() const and it is a new member function
        virtual void funcC(double = 0.0); // DerivedClass::funcC(double) has a different
                                          // parameter type than BaseClass::funcC(int), so
                                          // DerivedClass::funcC(double) is a new member function
    };
    
当你使用重写，编译器将生成错误而不是以无提示方式创建新的成员函数。

    class BaseClass
    {
        virtual void funcA();
        virtual void funcB() const;
        virtual void funcC(int = 0);
        void funcD();
    };
    class DerivedClass: public BaseClass
    {
        virtual void funcA() override; // ok
        virtual void funcB() override; // compiler error: DerivedClass::funcB() does not
                                       // override BaseClass::funcB() const
        virtual void funcC( double = 0.0 ) override; // compiler error:
                                                     // DerivedClass::funcC(double) does not
                                                     // override BaseClass::funcC(int)
        void funcD() override; // compiler error: DerivedClass::funcD() does not
                               // override the non-virtual BaseClass::funcD()
    };
    
若要指定函数不能重写和不能继承的类，请使用最终关键字。

#### 显式默认设置的函数和已删除的函数 ####

在 C++11 中，默认函数和已删除函数使你可以显式控制是否自动生成特殊成员函数。 已删除的函数还可为您提供简单语言，以防止所有类型的函数（特殊成员函数和普通成员函数以及非成员函数）的参数中出现有问题的类型提升，这会导致意外的函数调用。
__显式默认设置的函数和已删除函数的好处__
在 C++ 中，如果某个类型未声明它本身，则编译器将自动为该类型生成默认构造函数、复制构造函数、复制赋值运算符和析构函数。这些函数称为特殊成员函数，并且它们是什么发出简单用户定义的类型在 C++ 行为方式就类似结构往来 c。也就是说，你可以创建、 复制和销毁它们而无需任何额外编码工作。 C++11 会将移动语义引入语言中，并将移动构造函数和移动赋值运算符添加到编译器可自动生成的特殊成员函数的列表中。
这对于简单类型非常方便，但是复杂类型通常自己定义一个或多个特殊成员函数，这可以阻止自动生成其他特殊成员函数。 实践操作：
* 如果显式声明了任何构造函数，则不会自动生成默认构造函数。
* 如果显式声明了虚拟析构函数，则不会自动生成默认析构函数。
* 如果显式声明了移动构造函数或移动赋值运算符，则：
  + 不自动生成复制构造函数。
  +	不自动生成复制赋值运算符。
* 如果显式声明了复制构造函数、复制赋值运算符、移动构造函数、移动赋值运算符或析构函数，则：
  +	不自动生成移动构造函数。
  +	不自动生成移动赋值运算符。
  
备注：此外，C++11 标准指定将以下附加规则：
* 如果显式声明了复制构造函数或析构函数，则弃用复制赋值运算符的自动生成。
* 如果显式声明了复制赋值运算符或析构函数，则弃用复制构造函数的自动生成。
在这两种情况下，Visual Studio 将继续隐式自动生成所需的函数且不发出警告。这些规则的结果也可能泄漏到对象层次结构中。 例如，如果出于任何原因基类不具有默认构造函数可从派生类调用 — 即，公共或受保护的不带参数的构造函数 — 然后类派生不能自动生成其自己的默认构造函数。

这些规则可能会使本应直接的内容、用户定义类型和常见 C++ 惯例的实现变得复杂 — 例如，通过以私有方式复制构造函数和复制赋值运算符，而不定义它们，使用户定义类型不可复制。


 作为开发方，如果不想让用户使用某个成员函数，不声明即可；但对于特殊成员函数，则是另一种情况。例如，设计一个树叶类
 
         class LeafOfTree{ ... };
莱布尼茨说过，“世上没有两片完全相同的树叶” (Es gibt keine zwei Blätter, die gleich bleiben)，因此，对于一片独一无二的树叶，下面的操作是错误的：

        LeafOfTree  leaf1;
        LeafOfTree  leaf2;
        LeafOfTree  leaf3(leaf1);     // attempt to copy Leaf1 — should not compile!
        Leaf1 = Leaf2;              // attempt to copy Leaf2 — should not compile!
        
因此，此时需要避免使用 “拷贝构造函数” 和 “拷贝赋值算子”

~~C++98 中，可声明这些特殊成员函数为私有型 (private)，且不实现该函数，具体如下：~~

        class LeafOfTree{
        private:
            LeafOfTree( const LeafOfTree& );    　　　　　　 // not defined
            LeafOfTree & operator=( const LeafOfTree& );    // not defined
        };
        
  程序中如果调用了 LeafOfTree 类的拷贝构造函数 (或拷贝赋值操作符)，则在编译时，会出现链接错误 (link-time error)
  为了将报错提前到编译时 (compile time)，可增加了一个基类 Uncopyable，并将拷贝构造函数和拷贝赋值算子声明为私有型，具体可参见 《Effective C++_3rd item 6》 在谷歌 C++ 编码规范中，使用了一个宏定义来简化，如下所示：
  
        // A macro to disallow the copy constructor and operator= functions 
        // This should be used in the priavte:declarations for a class
        #define    DISALLOW_COPY_AND_ASSIGN(TypeName) \
            TypeName(const TypeName&);                \
            TypeName& operator=(const TypeName&)
    
在 C++11 之前，此代码段是不可复制的类型的惯例形式。 但是，它具有几个问题：
*	复制构造函数必须以私有方式进行声明以隐藏它，但因为它进行了完全声明，所以会阻止自动生成默认构造函数。 如果你需要默认构造函数，则必须显式定义一个（即使它不执行任何操作）。
*	即使显式定义的默认构造函数不执行任何操作，编译器也会将它视为重要内容。 其效率低于自动生成的默认构造函数，并且会阻止 noncopyable 成为真正的 POD 类型。
*	尽管复制构造函数和复制赋值运算符在外部代码中是隐藏的，但成员函数和 noncopyable 的友元仍可以看见并调用它们。 如果它们进行了声明但是未定义，则调用它们会导致链接器错误。
*	虽然这是广为接受的惯例，但是除非你了解用于自动生成特殊成员函数的所有规则，否则意图不明确。在 C++11 中，不可复制的习语可通过更直接的方法实现。

            struct LeafOfTree
            {
              LeafOfTree() =default;
              LeafOfTree( const LeafOfTree& )=delete;
              LeafOfTree & operator=( const LeafOfTree& )=delete;
            };
    
请注意如何解决与 C++11 之前的惯例有关的问题：
* 仍可通过声明复制构造函数来阻止生成默认构造函数，但可通过将其显式设置为默认值进行恢复。
* 显式设置的默认特殊成员函数仍被视为不重要的，因此性能不会下降，并且不会阻止 noncopyable 成为真正的 POD 类型。
* 复制构造函数和复制赋值运算符是公共的，但是已删除。 定义或调用已删除函数是编译时错误。
* 对于了解 =default 和 =delete 的人来说，意图是非常清楚的。 你不必了解用于自动生成特殊成员函数的规则。
对于创建不可移动、只能动态分配或无法动态分配的用户定义类型，存在类似惯例。 所有这些惯例都具有 C++11 之前的实现，这些实现会遭受类似问题，并且可在 C++11 中通过按照默认和已删除特殊成员函数实现它们，以类似方式进行解决。

#### 显式默认设置的函数 ####

可以默认设置任何特殊成员函数 — 以显式声明特殊成员函数使用默认实现、定义具有非公共访问限定符的特殊成员函数或恢复其他情况下被阻止其自动生成的特殊成员函数。可通过如此示例所示进行声明来默认设置特殊成员函数：

    struct widget
    {
      widget()=default;
      inline widget& operator=(const widget&);
    };
    inline widget& widget::operator=(const widget&) =default;
    
请注意，只要特殊成员函数可内联，便可以在类主体外部默认设置它。由于普通特殊成员函数的性能优势，因此我们建议你在需要默认行为时首选自动生成的特殊成员函数而不是空函数体。 你可以通过显式默认设置特殊成员函数，或通过不声明它（也不声明其他会阻止它自动生成的特殊成员函数），来实现此目的。

#### 已删除的函数 ####

可以删除特殊成员函数以及普通成员函数和非成员函数，以阻止定义或调用它们。 通过删除特殊成员函数，可以更简洁地阻止编译器生成不需要的特殊成员函数。 必须在声明函数时将其删除；不能在这之后通过声明一个函数然后不再使用的方式来将其删除。

    struct widget
    {
      // deleted operator new prevents widget from being dynamically allocated.
      void* operator new(std::size_t) = delete;
    };

删除普通成员函数或非成员函数可阻止有问题的类型提升导致调用意外函数。 这可发挥作用的原因是，已删除的函数仍参与重载决策，并提供比提升类型之后可能调用的函数更好的匹配。 函数调用将解析为更具体的但可删除的函数，并会导致编译器错误。

    // deleted overload prevents call through type promotion of float to double from succeeding.
    void call_with_true_double_only(float) =delete;
    void call_with_true_double_only(double param) { return; }
    template < typename T >
    void call_with_true_double_only(T) =delete; //prevent call through type promotion of any T to double from succeeding.
    void call_with_true_double_only(double param) { return; } // also define for const double, double&, etc. as needed.
    
请注意，在上述示例中，调用call_with_true_double_only通过使用 __float__  参数会导致编译器错误，但调用call_with_true_double_only通过 __int__ 参数不会; 在 __int__ 的情况下，该参数将从提升 __int__ 到 __double__ 并成功调用 __double__ 版本的函数即使这不可能的用途是什么。 若要确保使用非双精度自变量对此函数进行的任何调用均会导致编译器错误，你可以声明已删除的函数的模板版本。 
    
delete在模板特化。在模板特例化中，也可以用 delete 来过滤一些特定的形参类型。例如，Widget 类中声明了一个模板函数，当进行模板特化时，要求禁止参数为 void* 的函数调用。如果按照 C++98 的 “私有不实现“ 思路，应该是将特例化的函数声明为私有型，如下所示：

    class Widget {
    public:
        template<typename T>
        void ProcessPointer(T* ptr) { … }
    private:
        template<>             
        void ProcessPointer<void>(void*);    // error!
    };
    
问题是，模板特化应该被写在命名空间域 (namespace scope)，而不是类域 (class scope)，因此，该方法会报错。而在 C++11 中，因为有了 delete 关键字，则可以直接在类域外，将特例化的模板函数声明为 delete， 如下所示：

    class Widget {
    public:
        template<typename T>
        void ProcessPointer(T* ptr) { … }
    };
    template<> 
    void Widget::ProcessPointer<void>(void*) = delete; // still public, but deleted

### 4.7 long long int类型 ###

### 4.8 继承构造/委托构造 ###

继承构造。C++ 11允许派生类继承基类的构造函数（默认构造函数、复制构造函数、移动构造函数除外）。注意：继承的构造函数只能初始化基类中的成员变量，不能初始化派生类的成员变量。如果基类的构造函数被声明为私有，或者派生类是从基类中虚继承，那么不能继承构造函数。一旦使用继承构造函数，编译器不会再为派生类生成默认构造函数

      class A
      {
      public:
          A(int i) { std::cout << "i = " << i << std::endl; }
          A(double d, int i) {}
          A(float f, int i, const char* c) {}
          // ...
      };
      class B : public A
      {
      public:
          using A::A; // 继承构造函数
          // ...
          virtual void ExtraInterface(){}
      };
  
委托构造和继承构造函数类似，委托构造函数也是C++11中对C++的构造函数的一项改进，其目的也是为了减少程序员书写构造函数的时间。// 如果一个类包含多个构造函数，C++ 11允许在一个构造函数中的定义中使用另一个构造函数，但这必须通过初始化列表进行操作，如下：

      class Info
      {
      public:
          Info() : Info(1) { }    // 委托构造函数
          Info(int i) : Info(i, 'a') { } // 既是目标构造函数，也是委托构造函数
          Info(char e): Info(1, e) { }
      private:
          Info(int i, char e): type(i), name(e) { /\* 其它初始化 \*/} // 目标构造函数      
          int  type;
          char name;
          // ...
      };

### 4.9 静态断言 assertions ###

### 4.10 允许 sizeof 运算符作用在类型的数据成员上，无须明确的对象 ###

### 4.11 垃圾回收机制 ###

### 4.12 属性 ###


