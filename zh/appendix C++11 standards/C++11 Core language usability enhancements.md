## 3. 核心语言的可用性强化 ##

### 3.1 初始化列表(`std::initializer_list`) ###

C++03 支持数组和简单对象(POD)的初始化值列表，例如：

    int array[5] = {-3, -1, 0, 1, 3}; // 数组初始化列表

    struct Person {
        int id;
        std::string name;
    }；
    Person p = {111, "Tom"}; // 简单对象的初始化列表

但是上面的语法对复杂类型不可用，例如，在 C++03 标准中初始化一个 `std::vector<int> ` 对象可能需编写如下代码：

    int a[] = {0, 1, 2, 3};
    std::vector<int> vec(a, a + sizeof(a));

如果 C++ 对复杂类型也能提供一种类似初始化列表的构造方法，那么上面的代码可以简化为：

    std::vector<int> vec = {0, 1, 2, 3};

事实上，C++11 标准扩大了初始化列表的概念，并提供了 `std::initializer_list` 模板类(在 `<initializer_list>` 头文件中定义)：

    template< class T >
    class initializer_list;

当你使用类的初始化值列表时，C++11 会寻找参数类型为 `std::initializer_list` 的构造函数。

C++11 中引入 `std::initializer_list` 给 C++ 语言可用性带来了极大的提升。现在，初始化列表不再仅限于数组。

`<initializer_list>` 头文件摘要如下：

    namespace std {
        template<class E> class initializer_list {
            public:
                typedef E value_type;
                typedef const E& reference;
                typedef const E& const_reference;
                typedef size_t size_type;
                typedef const E* iterator;
                typedef const E* const_iterator;
                initializer_list() noexcept; // 默认构造函数
                size_t size() const noexcept; // 初始化列表元素的个数.
                const E* begin() const noexcept; // 返回指向初始化列表中第一个元素的指针.
                const E* end() const noexcept; // 返回指向最末尾元素的后续位置的指针.
        };
    
        template<class E> const E* begin(initializer_list<E> il) noexcept;
        template<class E> const E* end(initializer_list<E> il) noexcept;
    }

`std::initializer_list` 默认构造函数将会创建一个空的初始化列表。另外，在以下两种情况下，编译器会自动构建一个非空的初始化列表对象：

1. 在遇到初始化列表表达式(注: {1, 2, 3, 4} 即为一个简单的初始化列表表达式)时，编译器会自动构建一个非空的初始化列表对象，主要用于函数调用时初始化列表对象作为函数参数传入，或者在赋值表达式中设置某初始化列表对象的值。
2. 在 auto 修饰符限定下的初始化表达式中(包括基于范围的 for 循环)，编译器也会自动构建一个非空的初始化列表对象。

请看下例([参考](http://en.cppreference.com/w/cpp/utility/initializer_list/initializer_list))：

    #include <iostream>
    #include <initializer_list>
     
    int main() 
    {
        std::initializer_list<int> empty_list;
        std::cout << "empty_list.size(): " << empty_list.size() << '\n';
     
        // create initializer lists using list-initialization
        std::initializer_list<int> digits{1, 2, 3, 4, 5};
        std::cout << "digits.size(): " << digits.size() << '\n';
     
        // special rule for auto means 'fractions' has the
        // type std::initializer_list<double>
        auto fractions = {3.14159, 2.71828};
        std::cout << "fractions.size(): " << fractions.size() << '\n';
    }

下面例子介绍了初始化列表的基本用法([参考](http://en.cppreference.com/w/cpp/utility/initializer_list))：

    #include <iostream>
    #include <vector>
    #include <initializer_list>
     
    template <class T>
    struct S {
        std::vector<T> v;
        S(std::initializer_list<T> l) : v(l) {
             std::cout << "constructed with a " << l.size() << "-element list\n";
        }
        void append(std::initializer_list<T> l) {
            v.insert(v.end(), l.begin(), l.end());
        }
        std::pair<const T*, std::size_t> c_arr() const {
            return {&v[0], v.size()};  // list-initialization in return statement
                                       // this is NOT a use of std::initializer_list
        }
    };
     
    template <typename T>
    void templated_fn(T) {}
     
    int main()
    {
        S<int> s = {1, 2, 3, 4, 5}; // direct list-initialization
        s.append({6, 7, 8});        // list-initialization in function call
     
        std::cout << "The vector size is now " << s.c_arr().second << " ints:\n";
     
        for (auto n : s.v) std::cout << ' ' << n;
     
        std::cout << '\n';
     
        std::cout << "range-for over brace-init-list: \n";
     
        for (int x : {-1, -2, -3}) // the rule for auto makes this ranged for work
            std::cout << x << ' ';
        std::cout << '\n';
     
        auto al = {10, 11, 12};   // special rule for auto
     
        std::cout << "The list bound to auto has size() = " << al.size() << '\n';
     
        // templated_fn({1, 2, 3}); // compiler error! "{1, 2, 3}" is not an expression,
                                    // it has no type, and so T cannot be deduced
        templated_fn<std::initializer_list<int>>({1, 2, 3}); // OK
        templated_fn<std::vector<int>>({1, 2, 3});           // also OK
    }

函数执行结果如下：

    constructed with a 5-element list
    The vector size is now 8 ints:
     1 2 3 4 5 6 7 8
    range-for over brace-init-list: 
    -1 -2 -3 
    The list bound to auto has size() = 3

### 3.2 统一的初始化方式 ###

C++11 将会提供一种统一的语法初始化任意的对象，它扩充了初始化列表语法，无论是 POD 还是非 POD 类型(有关 POD 类型的定义，请参考[本文](http://en.wikipedia.org/wiki/C%2B%2B11#Modification_to_the_definition_of_plain_old_data))都可以使用 `obj = { ... }` 的方式来进行初始化或赋值。对于非 POD 类型，若通过 `{ ... }` 形式进行对象的初始化和赋值，编译器将自动匹配并调用构造函数或 `"="` 赋值操作，参见 `3.1 初始化列表`。因此，构造函数 `T(x, y)` 现在也可以统一写成：`T{x, y}`、`T var{x, y}` 等。

请看下例([参考](http://en.wikipedia.org/wiki/C%2B%2B11#Uniform_initialization))：

    struct BasicStruct
    {
     int x;
     float y;
    };
     
    struct AltStruct
    {
      AltStruct(int _x, float _y) : x(_x), y(_y) {}
     
    private:
      int x;
      float y;
    };
     
    BasicStruct var1{5, 3.2f};
    AltStruct var2{2, 4.3f};

另外，在需要返回一个 `T` 类型对象时，可以直接写：`return {x, y};`，统一的初始化语法能够免除指明特定类型的必要，例如([参考](http://en.wikipedia.org/wiki/C%2B%2B11#Uniform_initialization))：

    struct IdString
    {
      std::string name;
      int identifier;
    };
     
    IdString var3{"SomeName", 4};

该语法将会使用 `const char *` 参数初始化 `std::string`，在函数返回某个 IdString 对象时，可以直接编写如下代码：

    IdString GetString()
    {
      return {"SomeName", 4}; // 不需指明特定的类型。
    }

### 3.3 类型推导(auto 和 decltype 关键字) ###

C++03 标准中变量和参数必须明确指明类型，但是随着模板类型的出现以及模板元编程的技巧，对象的类型特别是函数的返回类型就不容易表示了。C++11 标准针对上面的情况引入了 auto 和 decltype 关键字(实际上，auto 关键字在旧的 C++ 标准中即存在，只不过在 C++11 标准中新增了类型自动推导语义)。

如果某个对象在初始化时类型已经明确，那么可以使用 auto 关键字来简化对象的声明，该对象的类型会根据初始化算子(initializer: 姑且翻译为"初始化算子"吧，包括类的构造函数或某个基本类型的变量值(比方说整型(int) 5)自动推导出来，使用 auto 来声明变量的语法如下：

    auto variable initializer 

例如：
    
    auto otherVariable = 5; // otherVariable 的类型为 int。
    const auto *v = &x, u = 6; // 正确，v 的类型为 const int*, u 的类型为 const int。
    static auto y = 0.0; // y 推导为 double 类型。
    auto int r; // 错误: auto 在 C++11 中不是存储类型修饰符。

auto 类型修饰符也可以出现在带有返回类型的函数（比如 `std::vector<T>::begin()` 等）的返回值前面，用于指定该函数返回值的类型，在返回值类型很复杂的情况下，auto 的类型推导可以减少大量冗赘代码。例如：

    // someStrangeCallableType 是某个类的成员函数类型，该类型也可以使用 std::function<> 来声明。
    auto someStrangeCallableType = std::bind(&SomeFunction, _2, _1, someObject);

auto 的另外一种用法是修饰函数，主要用于函数声明，在很多情况下，我们并不能提前知道函数的返回值类型(即函数的返回值类型通常由其参数决定)，那么此时  auto 关键字就可以派上用场了，auto 关键字修饰函数的语法如下：

    auto function -> return type

请看下面完整的例子：[参考](http://en.cppreference.com/w/cpp/language/auto)

    #include <iostream>
    #include <cmath>
    #include <typeinfo>
     
    template<class T, class U>
    auto add(T t, U u) -> decltype(t + u) // the return type of add is the type of operator+(T,U)
    {
        return t + u;
    }
     
    auto get_fun(int arg)->double(*)(double) // same as double (*get_fun(int))(double)
    {
        switch (arg) {
            case 1: return std::fabs;
            case 2: return std::sin;
            default: return std::cos;
        }
    }
     
    int main()
    {
        auto a = 1 + 2;
        std::cout << "type of a: " << typeid(a).name() << '\n';
        auto b = add(1, 1.2);
        std::cout << "type of b: " << typeid(b).name() << '\n';
        //auto int c; //compile-time error
        auto d = {1, 2};
        std::cout << "type of d: " << typeid(d).name() << '\n';
     
        auto my_lambda = [](int x) { return x + 3; };
        std::cout << "my_lambda: " << my_lambda(5) << '\n';
     
        auto my_fun = get_fun(2);
        std::cout << "type of my_fun: " << typeid(my_fun).name() << '\n';
        std::cout << "my_fun: " << my_fun(3) << '\n';
    }

decltype 和 auto 一起使用会更为有用，因为 auto 参数的类型只有编译器知道。然而 decltype 对于那些大量运用运算符重载和特化的类型的代码的表示也非常有用。

auto 对于减少冗赘的代码也很有用。例如：

    for (vector<int>::const_iterator itr = myvec.cbegin(); itr != myvec.cend(); ++itr)

可以改写成更简洁的：

    for (auto itr = myvec.cbegin(); itr != myvec.cend(); ++itr)

Decltype 主要对值和表达式的类型推导，decltype 推导规则如下：

1. 如果表达式 e 是一个变量，那么由 decltype 推导出来的类型就是这个变量的类型。
2. 如果表达式 e 是一个函数，那么由 decltype 推导出来的类型就是这个函数返回值的类型。
3. 如果不符合 1 和 2，如果 e 是左值，类型为 T，那么 decltype(e) 是 T&；如果是右值，则是 T。

		//declare type
		int getSize();
		... main(void)
		{
			//1.基本用法
			{
				int tempA = 2;
				/*1.dclTempA为int*/
				decltype(tempA) dclTempA;
				/*2.dclTempB为int，对于getSize根本没有定义，但是程序依旧正常，因为decltype只做分析，并不调用getSize，*/
				decltype(getSize()) dclTempB;
			}
			//2.与const结合
			{
				double tempA = 3.0;
				const double ctempA = 5.0;
				const double ctempB = 6.0;
				const double *const cptrTempA = &ctempA;

				/*1.dclTempA推断为const double（保留顶层const，此处与auto不同）*/
				decltype(ctempA) dclTempA = 4.1;
				/*2.dclTempA为const double，不能对其赋值，编译不过*/
				//dclTempA = 5;
				/*3.dclTempB推断为const double * const*/
				decltype(cptrTempA) dclTempB = &ctempA;
				/*4.输出为4（32位计算机）和5*/
				cout << sizeof(dclTempB) << "    " << *dclTempB << endl;
				//5.保留顶层const，不能修改指针指向的对象，编译不过 const * const    
				//dclTempB = &ctempB;
				/*6.保留底层const，不能修改指针指向的对象的值，编译不过*/
				//*dclTempB = 7.0;
			}
			/*3.与引用结合*/
			{

				int tempA = 0, &refTempA = tempA;

				/*1.dclTempA为引用，绑定到tempA*/
				decltype(refTempA) dclTempA = tempA;
				/*2.dclTempB为引用，必须绑定到变量，编译不过*/
				//decltype(refTempA) dclTempB = 0;
				/*3.dclTempC为引用，必须初始化，编译不过*/
				//decltype(refTempA) dclTempC;
				/*4.双层括号表示引用，dclTempD为引用，绑定到tempA*/
				decltype((tempA)) dclTempD = tempA;

				const int ctempA = 1/*ctempA是常量*/, &crefTempA = ctempA;

				/*5.dclTempE为常量引用，可以绑定到普通变量tempA*/
				decltype(crefTempA) dclTempE = tempA;
				/*6.dclTempF为常量引用，可以绑定到常量ctempA*/
				decltype(crefTempA) dclTempF = ctempA;
				/*7.dclTempG为常量引用，绑定到一个临时变量*/
				decltype(crefTempA) dclTempG = 0;
				/*8.dclTempH为常量引用，必须初始化，编译不过*/
				//decltype(crefTempA) dclTempH;
				/*9.双层括号表示引用,dclTempI为常量引用，可以绑定到普通变量tempA*/
				decltype((ctempA))  dclTempI = ctempA;
			}
				/*4.与指针结合*/
			{
				int tempA = 2;
				int *ptrTempA = &tempA;
				/*1.常规使用dclTempA为一个int *的指针*/
				decltype(ptrTempA) dclTempA;
				/*2.需要特别注意，表达式内容为解引用操作，dclTempB为一个引用，引用必须初始化，故编译不过*/
			//	decltype(*ptrTempA) dclTempB;
			}
			return 0;
		}

### 3.4 基于范围的 for 循环 ###

基于范围的for循环可以用非常简单的方式迭代集合中的每一项，C++11 标准中规定基于范围的 for 循环具有如下形式：

    for (for-range-declaration : expression) statement

基于范围的 for 语句可以构造出在某个范围内执行的循环语句，上述语句可以做如下解释：为 expression 的每个元素重复并按顺序执行 statement。例如：

    int array[5] = { 1, 2, 3, 4, 5 };
    for (int& x : array)
        x *= 2;

更完整的例子[如下](http://msdn.microsoft.com/zh-cn/library/jj203382.ASPX)：

    #include <iostream>
    #include <vector>
    using namespace std;
    
    int main() 
    {
        // Basic 10-element integer array.
        int x[10] = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    
        // Range-based for loop to iterate through the array.
        for( int y : x ) { // Access by value using a copy declared as a specific type. 
                           // Not preferred.
            cout << y << " ";
        }
        cout << endl;
    
        // The auto keyword causes type inference to be used. Preferred.
    
        for( auto y : x ) { // Copy of 'x', almost always undesirable
            cout << y << " ";
        }
        cout << endl;
    
        for( auto &y : x ) { // Type inference by reference.
            // Observes and/or modifies in-place. Preferred when modify is needed.
            cout << y << " ";
        }
        cout << endl;
    
        for( const auto &y : x ) { // Type inference by reference.
            // Observes in-place. Preferred when no modify is needed.
            cout << y << " ";
        }
        cout << endl;
        cout << "end of integer array test" << endl;
        cout << endl;
    
        // Create a vector object that contains 10 elements.
        vector<double> v;
        for (int i = 0; i < 10; ++i) {
            v.push_back(i + 0.14159);
        }
    
        // Range-based for loop to iterate through the vector, observing in-place.
        for( const auto &j : v ) {
            cout << j << " ";
        }
        cout << endl;
        cout << "end of vector test" << endl;
    }

基于范围的 for 循环语句在如下条件下会退出执行，1、`break`，2、`return`，3、`goto` 至该 for 循环语句之外某一的标签语句处。

注意，基于范围的 for 语句会：

- 自动识别数组。
- 自动识别具有 .begin() 和 .end() 成员函数的容器。
- 对于其他类型，需使用依赖于参数的 `begin()` 和 `end()`。
- 如果要在循环中修改数组或容器的每个元素，可使用引用类型。

### 3.5 lambda 表达式 ###

C++11 新增了很多特性，lambda 表达式是其中之一。很多语言都提供了 lambda 表达式，如 Python，Java 8。lambda 表达式可以方便地构造匿名函数，如果你的代码里面存在大量的小函数，而这些函数一般只被调用一次，那么不妨将他们重构成 lambda 表达式。

<table>
<tbody>
<tr>
<td><code><strong>[</strong></code> <span>capture</span> <code><strong>]</strong></code> <code><strong>(</strong></code> <span>params</span> <code><strong>)</strong></code> <span>mutable</span> <span>exception</span> <span>attribute</span> <code><strong>-&gt;</strong></code> <span>ret</span> <code><strong>{</strong></code> <span>body</span> <code><strong>}</strong></code></td>
<td>(1)</td>
<td>&nbsp;</td>
</tr>
<tr>
<td><code><strong>[</strong></code> <span>capture</span> <code><strong>]</strong></code> <code><strong>(</strong></code> <span>params</span> <code><strong>)</strong></code> <code><strong>-&gt;</strong></code> <span>ret</span> <code><strong>{</strong></code> <span>body</span> <code><strong>}</strong></code></td>
<td>(2)</td>
<td>&nbsp;</td>
</tr>
<tr>
<td><code><strong>[</strong></code> <span>capture</span> <code><strong>]</strong></code> <code><strong>(</strong></code> <span>params</span> <code><strong>)</strong></code> <code><strong>{</strong></code> <span>body</span> <code><strong>}</strong></code></td>
<td>(3)</td>
<td>&nbsp;</td>
</tr>
<tr>
<td><code><strong>[</strong></code> <span>capture</span> <code><strong>]</strong></code> <code><strong>{</strong></code> <span>body</span> <code><strong>}</strong></code></td>
<td>(4)</td>
<td>&nbsp;</td>
</tr>
</tbody>
</table>

其中：

- (1) 是完整的 lambda 表达式形式，
- (2) const 类型的 lambda 表达式，该类型的表达式不能改捕获("capture")列表中的值。
- (3) 省略了返回值类型的 lambda 表达式，但是该 lambda 表达式的返回类型可以按照下列规则推演出来：

        (3.1) 如果 lambda 代码块中包含了 return 语句，则该 lambda 表达式的返回类型由 return 语句的返回类型确定。

        (3.2) 如果没有 return 语句，则类似 void f(...) 函数。

- (4) 省略了参数列表，类似于无参函数 f()。

`mutable` 修饰符说明 lambda 表达式体内的代码可以修改被捕获的变量，并且可以访问被捕获对象的 non-const 方法。

`exception` 说明 lambda 表达式是否抛出异常(`noexcept`)，以及抛出何种异常，类似于 `void f() throw(X, Y)`。

attribute 用来声明属性。

另外，`capture` 指定了在可见域范围内 lambda 表达式的代码内可见得外部变量的列表，具体解释如下：

- [a,&b]： a变量以值的方式被捕获，b以引用的方式被捕获。
- [this]： 以值的方式捕获 this 指针。
- [&]： 以引用的方式捕获所有的外部自动变量。
- [=]： 以值的方式捕获所有的外部自动变量。
- []： 不捕获外部的任何变量。

此外，params 指定 lambda 表达式的参数。

请看下面 lambda 表达式的例子：

    #include <vector>
    #include <iostream>
    #include <algorithm>
    #include <functional>
     
   	{

		//嵌套Lambda
		int m = [](int x) { return [](int y) { return y * 2; }(x)+6; }(5);
		std::cout << "m:" << m << std::endl;            		//输出m:16

		 m = [](int x) { return [](int y) { return y * 2; }(x)+10; }(6);
		std::cout << "m:" << m << std::endl;       //输出m:22  看来是将6==> 6*2 ==>12+10   note:不是将(x)+10整体传进去   { return y * 2; }(x)优先级比（x）+10高

		std::cout << "n:" << [](int x, int y) { return x + y; }(5, 4) << std::endl;            //输出n:9

		/*-----*/
		auto gFunc = [](int x) -> std::function<int(int)> { return [=](int y) { return x + y; }; };// auto自动替换为std::function<int(int)> 类型
		auto lFunc = gFunc(4);
		std::cout << lFunc(5) << std::endl;//9
		/*-----*/

		auto hFunc = [](const std::function<int(int)>& f, int z) { return f(z) + 1; };//7+8+1
		auto a = hFunc(gFunc(7), 8);
		std::cout << a << std::endl;//16
	}	
	{
		int a = 111, b = 222;
		auto func = [=, &b]()mutable { a = 22; b = 333; std::cout << "a:" << a << " b:" << b << std::endl; };

		func();
		std::cout << "a:" << a << " b:" << b << std::endl;

		a = 333;
		auto func2 = [=, &a] { a = 444; std::cout << "a:" << a << " b:" << b << std::endl; };
		func2();

		auto func3 = [](int x) ->std::function<int(int)> { return [=](int y) { return x + y; }; };


		std::function<void(int x)> f_display_42 = [](int x) { print_num(x); };
		f_display_42(44);
	}
	{
		{

			using namespace std;

			// Assign the lambda expression that adds two numbers to an auto variable.
			auto f1 = [](int x, int y) { return x + y; };

			cout << f1(2, 3) << endl;

			// Assign the same lambda expression to a function object.
			function<int(int, int)> f2 = [](int x, int y) { return x + y; };

			cout << f2(3, 4) << endl;
		}
	}

	{
		//高阶 Lambda 函数
		{
			using namespace std;

			// The following code declares a lambda expression that returns
			// another lambda expression that adds two numbers.
			// The returned lambda expression captures parameter x by value.
			auto addtwointegers = [](int x) -> function<int(int)> {
				return [=](int y) { return x + y; };
			};

			// The following code declares a lambda expression that takes another
			// lambda expression as its argument.
			// The lambda expression applies the argument z to the function f
			// and multiplies by 2.
			auto higherorder = [](const function<int(int)>& f, int z) {
				return f(z) * 2;
			};

			// Call the lambda expression that is bound to higherorder.
			auto answer = higherorder(addtwointegers(7), 8);

			// Print the result, which is (7+8)*2.
			cout << answer << endl;
		}

	}

### 3.6 另一种可选的函数语法 ###

> 本小节主要来自于维基百科[中文 C++11 介绍](http://zh.wikipedia.org/zh-cn/C%2B%2B11#.E5.8F.A6.E4.B8.80.E7.A8.AE.E7.9A.84.E5.87.BD.E5.BC.8F.E8.AA.9E.E6.B3.95)

标准 C 函数声明语法对于 C 语言已经足够。 演化自 C 的 C++ 除了 C 的基础语法外，又扩充额外的语法。 然而，当 C++ 变得更为复杂时，它暴露出许多语法上的限制， 特别是针对函数模板的声明。 下面的示例，不是合法的 C++03：

    template< typename LHS, typename RHS> 
    Ret AddingFunc(const LHS &lhs, const RHS &rhs) { return lhs + rhs; } //Ret的类型必须是(lhs+rhs)的类型

Ret 的类型由 LHS 与 RHS 相加之后的结果的类型来决定。 即使使用 C++11 新加入的 decltype 来声明 AddingFunc 的返回类型，依然不可行。

    template< typename LHS, typename RHS> 
    decltype(lhs+rhs) AddingFunc(const LHS &lhs, const RHS &rhs) { return lhs + rhs; } //不合法的 C++11

不合法的原因在于 lhs 及 rhs 在定义前就出现了。 直到编译器解析到函数原型的后半部，lhs 与 rhs 才是有意义的。

针对此问题，C++11 引进一种新的函数声明与定义的语法：

    template< typename LHS, typename RHS> 
    auto AddingFunc(const LHS &lhs, const RHS &rhs) -> decltype(lhs+rhs) { return lhs + rhs; }

上述语法也能套用到一般的函数声明与定义：

    struct SomeStruct
    {
        auto FuncName(int x, int y) -> int;
    };
 
    auto SomeStruct::FuncName(int x, int y) -> int
    {
          return x + y;
    }

关键字 `auto` 的使用与其在自动类型推导代表不同的意义。

### 3.7 对象创建优化 ###

在 C++11 中，一个构造函数可以调用该类中的其它构造函数来完成部分初始化任务（类似于 C# 中的委托）。声明成员时可以直接指定默认初始值，例如：

    class SomeType {
      int number;
      string name;
      SomeType( int i, string& s ) : number(i), name(s){}
    public:
      SomeType( )           : SomeType( 0, "invalid" ){}
      SomeType( int i )     : SomeType( i, "guest" ){}
      SomeType( string& s ) : SomeType( 1, s ){ PostInit(); }
    };

委托可以在一定程度上简化与类初始化相关代码，也更利于编译器优化。

### 3.8 显式虚函数重载 ###

### 3.9 空指针常量(nullptr) ###

在C语言中，我们使用NULL表示空指针，也就是我们可以写如下代码：

    int *i = NULL;

实际上在C语言中，NULL通常被定义为如下：

    #define NULL ((void *)0)
    
也就是说NULL实际上是一个void * 的指针，然后吧void * 指针赋值给int * 和 foo_t * 的指针的时候，隐式转换成相应的类型。而如果换做一个C++编译器来编译的话是要出错的，因为C++是强类型的，void * 是不能隐式转换成其他指针类型的，所以通常情况下，编译器提供的头文件会这样定义NULL：	F12 看一看

	#ifdef __cplusplus
	#define NULL 0
	#else
	#define NULL ((void *)0)
	#endif
	
~~实际上C++的书都会推荐说C++中更习惯使用0来表示空指针而不是NULL，尽管NULL在C++编译器下就是0。为什么C++的书都推荐使用0而不是NULL来表示空指针呢？~~
在foo.h文件中声明了一个函数：

	void bar(sometype1 a, sometype2 *b);

这个函数在a.cpp、b.cpp中调用了，分别是：
a.cpp:
bar(a, b);
b.cpp:
bar(a, 0);
好的，这些代码都是正常完美的编译运行。但是突然在某个时候我们功能扩展，需要对bar函数进行扩展，我们使用了重载，现在foo.h的声明如下：

	void bar(sometype1 a, sometype2 *b);
	void bar(sometype1 a, int i);

这个时候危险了，a.cpp和b.cpp中的调用代码这个时候就不能按照期望的运行了。但是我们很快就会发现b.cpp中的0是整数，也就是在overload resolution的时候，我们知道它调用的是void bar(sometype1 a, int i)这个重载函数，于是我们可以做出如下修改让代码按照期望运行：

	bar(a, static_cast<sometype2 *>(0)); //这个类型转换后用这个 void bar(sometype1 a, sometype2 *b)函数

我知道，如果我们一开始就有bar的这两个重载函数的话，我们会在一开始就想办法避免这个问题（不使用重载）或者我们写出正确的调用代码，然而后面的这个重载函数或许是我们几个月或者很长一段时间后加上的话，那我们出错的可能性就会加大了不少。貌似我们现在说道的这些跟C++通常使用0来表示空指针没什么关系，好吧，假设我们的调用代码是这样的：foo.h:

	void bar(sometype1 a, sometype2 *b);
	
a.cpp:

	bar(a, b);
b.cpp:

	bar(a, NULL);

当bar的重载函数在后面加上来了之后，我们会发现出错了，但是出错的时候，我们找到b.cpp中的调用代码也很快可能忽略过去了。因为我们用的是NULL空指针啊，应该是调用的void bar(sometype1 a, sometype2 * b)这个重载函数啊。实际上NULL在C++中就是0，写NULL这个反而会让你没那么警觉，因为NULL不够“明显”，而这里如果是使用0来表示空指针，那就会够“明显”，因为0是空指针，它更是一个整形常量。在C++中，使用0来做为空指针会比使用NULL来做空指针会让你更加警觉。
~~C++ 11的nullptr~~ 虽然上面我们说明了0比NULL可以让我们更加警觉，但是我们并没有避免这个问题。这个时候C++ 11的nullptr就很好的解决了这个问题，
我们在C++ 11中使用nullptr来表示空指针，这样最早的代码是这样的。在我们后来把bar的重载加上了之后，代码是这样：
foo.h：

	void bar(sometype1 a, sometype2 *b);
	void bar(sometype1 a, int i);

b.cpp：

	bar(a, b);
	bar(a, nullptr);

这时候，我们的代码还是能够如预期的一样正确运行。
在没有C++ 11的nullptr的时候，我们怎么解决避免这个问题呢？我们可以自己实现一个（《Imperfect C++》上面有一个实现）

	const
	class nullptr_t
	{
	public:
		template<class T>
		inline operator T*() const
		{
			return 0;
		}

		template<class C, class T>
		inline operator T C::*() const
		{
			return 0;
		}

	private:
		void operator&() const;
	} nullptr = {};	

### 3.10 强类型枚举 ###

> 本小节主要来自维基百科中文词条 [C+11 强类型枚举](http://zh.wikipedia.org/zh-cn/C%2B%2B11#.E5.BC.B7.E5.9E.8B.E5.88.A5.E5.88.97.E8.88.89) 一节

在标准 C++ 中，枚举类型不是类型安全的。枚举类型被视为整数，这使得两种不同的枚举类型之间可以进行比较。C++03 唯一提供的安全机制是一个整数或一个枚举型值不能隐式转换到另一个枚举别型。

此外，枚举所使用整数类型及其大小都由实现方法定义，无法明确指定。 最后，枚举的名称全数暴露于一般作用域中，因此两个不同的枚举，不可以有相同的枚举名。 (例如 `enum Side{ Right, Left };` 和 `enum Thing{ Wrong, Right };` 不能一起使用。)

C++11 引入了一种特别的 "枚举类"，可以避免上述的问题。使用 enum class 的语法来声明：

    enum class Enumeration
    {
      Val1,
      Val2,
      Val3 = 100,
      Val4 /* = 101 */,
    };

上述枚举为类型安全的。枚举类型不能隐式地转换为整数；也无法与整数数值做比较。 (表达式 Enumeration::Val4 == 101 将会触发编译期错误)。

枚举类型所使用类型必须显式指定。在上面的示例中，使用的是默认类型 int，但也可以指定其他类型：

    enum class Enum2 : unsigned int {Val1, Val2};

枚举类型的作用域(scoping)定义为枚举类型的名称范围中。 使用枚举类型的枚举名时，必须明确指定其所属范围。 以前述枚举类型 `Enum2` 为例，`Enum2::Val1` 是有意义的表示法， 而单独的 `Val1` 则不是有意义的表示法。

此外，C++11 允许为传统的枚举指定类型：

    enum Enum3 : unsigned long {Val1 = 1, Val2};

枚举名 `Val1` 定义于 `Enum3` 的枚举范围中(`Enum3::Val1`)，但为了兼容性， `Val1` 仍然可以于一般的范围中单独使用。

在 C++11 中，枚举类型的前置声明 (forward declaration) 也是可行的，只要使用可指定类型的新式枚举即可。 之前的 C++ 无法写出枚举的前置声明，是由于无法确定枚举参数所占的空间大小， C++11 解决了这个问题：

    enum Enum1;                     // 不合法的 C++ 与 C++11; 无法判別大小
    enum Enum2 : unsigned int;      // 合法的 C++11
    enum class Enum3;               // 合法的 C++11，枚举类型默认为 int
    enum class Enum4: unsigned int; // 合法的 C++11
    enum Enum2 : unsigned short;    // 不合法的 C++11，Enum2 已被声明为 unsigned int

### 3.11 右尖括号(>) ###

标准 C++03 的语法分析器一律将 ">>" 视为右移运算符。 但在模板定义式中，很多情况下 ">>" 其实代表了两个连续右角括号。C++11 新标准不在要求声明嵌套模板时使用空格将尖括号分开。

### 3.12 显式类型转换操作符 ###


### 3.13 模板别名 ###

为了定义模板的别名，C++11 增加了以下的语法：

    template< typename first, typename second, int third>
    class SomeType;
 
    template< typename second>
    using TypedefName = SomeType<OtherType, second, 5>;

`using` 也能在 C++11 中定义一般类型的别名，等同 `typedef`：

    typedef void (*Function)(double);            // 传统语法
    using Function = void (*)(double);           // C++11 新增语法

### 3.14 无限制 union ###

在标准 C++ 中，并非任意的类型都能做为 `union` 的成员。比方说，带有 `non-trivial` 构造函数的类型就不能是 `union` 的成员。在新的标准里，移除了所有对 `union` 的使用限制，但规定 `union` 的成员不能为引用类型。 

请看下例：

    struct Point
    {
      Point() {}
      Point(int x, int y): x_(x), y_(y) {}
      int x_, y_;
    };
    
    union
    {
      int z;
      double w;
      Point p;  // 在 C++03 标准中非法; Point 有一 non-trivial 构造函数
                // 但是在 C++11 标准中合法
    };

### 3.15 std::Fuction<> ###

Class template std::function is a general-purpose polymorphic function wrapper. Instances of std::function can store, copy, and invoke any Callable target -- functions, lambda expressions, bind expressions, or other function objects, as well as pointers to member functions and pointers to data members.The stored callable object is called the target of std::function. If a std::function contains no target, it is called empty. Invoking the target of an empty std::function results in std::bad_function_call exception being thrown.
std::function satisfies the requirements of CopyConstructible and CopyAssignable.

	__#include <functional>__
	#include <iostream>
	//std::function  <===#include <functional>
	//保存lambda表达式
	 //std::function<void()> func_1 = []() {std::cout << "hello world" << std::endl; }; 
	 //func_1();
	//运行输出：he0llo world
	struct Foo{
		Foo(int num) : num_(num) {}
		void print_add(int i) const { std::cout << num_ + i << '\n'; }
		int num_;//class中private：
	};
	void print_num(int i){
		std::cout << i << '\n';
	}
	struct PrintNum{//C++11结构体内能写函数了，不是纯C
		void operator()(int i) const{//const默认的是成员函数第一个隐藏参数this 非成员函数，全局函数不能在后面加const	
			std::cout << i << '\n';
		}
	};
	struct TAdd{
		int Add(int x, int y){
			return x + y;
		}
	};
	int main()
	{
		// store a free function
		std::function<void(int)> f_display = print_num;
		f_display(-9);

		// store a lambda
		std::function<void()> f_display_42 = []() { print_num(42); };
		f_display_42();

		// store the result of a call to std::bind
		std::function<void()> f_display_31337 = std::bind(print_num, 31337);
		f_display_31337();

		// store a call to a member function
		std::function<void(const Foo&, int)> f_add_display = &Foo::print_add;

		const Foo foo(314159);
		f_add_display(foo, 1);//314160
		f_add_display(314159, 1);//314160// 这个居然可以，而且f11进去看一下发现先这一行Foo(int num) : num_(num) {} 在调用print_add

		// store a call to a data member accessor存取器
		std::function<int(Foo const&)> f_num = &Foo::num_;
		std::cout << "num_: " << f_num(foo) << '\n';

		// store a call to a member function and object
		using std::placeholders::_1;
		std::function<void(int)> f_add_display2 = std::bind(&Foo::print_add, foo, _1);// foo+2  绑定的是foo 
		f_add_display2(2);
		
		// store a call to a member function and object ptr
		std::function<void(int)> f_add_display3 = std::bind(&Foo::print_add, &foo, _1);// foo+3
		f_add_display3(3);
		
		std::function<int(TAdd *, int, int)> f = &TAdd::Add;
		TAdd tAdd;
		std::cout << f(&tAdd, 2, 3) << std::endl;  // 如果前面的模板参数为传值或引用，直接传入tAdd即可
		//======> std::function<int(TAdd *, int, int)> f = std::bind(&TAdd::Add ,&tAdd, _1, _2);=/===>f(2,3)

		// store a call to a function object
		std::function<void(int)> f_display_obj = PrintNum();
		f_display_obj(18);
		system("pause");
		return 0;
	}
/*Output:
-9
42
31337
314160
314160
num_: 314159
	314161
	314162
	5
	18
*/
