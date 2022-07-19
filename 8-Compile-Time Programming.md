# 8 Compile-Time programming 编译时编程

C++包含一些在编译时求值的方法,随着标准的更新,其可能性大大增强

*  在C++98之前,模版已经提供了编译时计算的能力,包括循环和路径选择
* 使用偏特化,我们可以根据特定的约束在不同类模版间选择
* 使用SFINAE原则,我们可以根据特定的约束在不同函数模版间选择
* 在C++11与C++14中,通过使用直观的执行路径选择和constexpr,编译时计算获得了更好的支持
* C++17引入了编译时if

本章将介绍这些特性

## 8.2 Template Metaprogramming 模版元编程

与动态语言的泛型不同,模版的实例化在编译时进行.而C++模版的一些特性可以与实例化过程相结合, 在程序中产生原始的递归语言.关于黑魔法的所有内容将在23章介绍,但是现在我们先看一个计算素数的例子

```cpp
template<unsigned p, unsigned d>
struct DoIsPrime {
    static constexpr bool value = (p % d != 0) && DoIsPrime<p,d-1>::value;
};

template<unsigned p>
struct DoIsPrime<p,2> {
    static constexpr bool value = (p % 2 != 0);		//出口
};

template<unsigned p>
struct IsPrime {
    static constexpr bool value = DoIsPrime<p, p/2>::value;
};

//为了防止无限递归产生的特例
template<>
struct IsPrime<0> {static constexpr bool value = false};
template<>
struct IsPrime<1> {static constexpr bool value = false};
template<>
struct IsPrime<2> {static constexpr bool value = true};
template<>
struct IsPrime<3> {static constexpr bool value = true};
```

这样当我们调用`IsPrime<9>::value`时,即可在编译期得到结果

* 我们通过递归展开在编译期遍历从p/2到2的所有除数,以确定p能否被整除

* 当d==2时,使用偏特化结束递归

*更多细节见Chapter 23*

## 8.2 Computing with *constexpr* 利用*constexpr*计算

C++11引入的constexpr关键字简化了编译期计算,如果有适当的输入,constexpr函数可以在编译期返回结果,虽然C++11还规定constexpr函数内容不能多于一条return语句,但是这些限制大部分都在C++14被废除了,当然,在编译期得到函数结果需要函数内所有计算都是可在编译期得到结果的,到目前为止,我们还不能做到堆分配或者抛出异常等.

在C++11中,判断素数的函数可以这样写

```cpp
constexpr bool
doIsPrime(unsigned const p, unsigned const d) {
    return d != 2 ? (p % d != 0) && doIsPrime(p, d - 1)
                    : (p % 2 != 0);
}

constexpr bool
isPrime (unsigned const p) {
    return p < 4 ? p >= 2 : doIsPrime(p, p / 2);
}
```

C++14以后,if之类的条件判断语句可以在constexpr函数内使用

```cpp
constexpr bool
isPrime (unsigned const p) {
    for(unsigned d = p / 2; d >= 2; --d) {
        if(p % d == 0) return false;
    }
    return p > 1; //没有发现因子
}
```

注意,标记为constexpr的函数不一定就会在编译期返回结果

```cpp
constexpr bool b1 = isPrime(9);	//return at compile time
std::cout << isPrime(9);		//return at run time 
```

注意,编译期优化等级也会产生影响,在-O3等级下,编译器会尽可能在编译期得到结果并将函数调用直接替换为结果

## 8.3 Execution Path Selection with Partial Specialization 偏特化的执行路径选择

isPrime()等函数的一种应用是在编译期利用偏特化在不同路径间选择,例如

```cpp
template<int SZ, bool = isPrime(SZ)>
struct helper;

template<int SZ>
struct helper<SZ, false> {
    //...
};

template<int SZ>
struct helper<SZ, true> {
    //...
};
```

这里我们使用了两个偏特化,当然我们也可以

```cpp
template<int SZ, bool = isPrime(SZ)>
struct helper;

template<int SZ>
struct helper<SZ, true> {	//当SZ是质数时,这个偏特化版本更匹配
    //...
};
```

因为函数模版不支持偏特化,所以我们必须找其他方法,包括:

* 将函数包装为类的静态函数
* 使用std::enable_if<>,其在*Section 6.3*有介绍
* 使用马上就要介绍的SFINAE
* 使用编译期if(C++17以后),*将在Section 8.5介绍*

*Chapter 20讨论了基于限制选择函数实现的技术*

## 8.4 SFINAE(Substitution Failure Is Not An Error) 替换失败不是一种错误 

在C++中,重载函数以符合各种参数类型是很常见的.因此,当编译器看到对重载函数的调用时,它必须单独考虑每个候选重载函数,评估参数并选择最匹配的候选函数*(有关此过程的一些细节,请参阅附录C)*.
在候选函数包括函数模板的情况下,编译器首先必须确定应该使用哪些模板参数,然后在函数参数列表及其返回类型中替换这些参数,然后评估其匹配程度(就像普通函数一样).但是,替换过程可能会遇到问题:它可能产生毫无意义的构造.语言规则不会认定这种无意义的替换会导致错误,而是说,具有这种替换问题的候选函数被简单地忽略了.
我们称这个原则为SFINAE(发音类似于sfee-nay),它代表“替换失败不是一种错误”.
注意,这里描述的替换过程不同于按需实例化过程*(参阅第27页的2.2节)*:即使是并不需要的潜在实例化,也可以进行替换(因此编译器可以评估它们是否确实不需要).
考虑下面的例子:

```cpp
template<typename T, unsigned N>
std::size_t len(T (&)[N]) {
    return N;
}

//number of elements for a type have *size_type*
template<typename T>
typename T::size_type len(T const &t) {
    return t.size();
}
```

这里第一个函数模版将形参声明为`T [&][N]`,这意味着形参必须是有N个T元素的数组

第二个函数将形参声明为没有约束的T,返回类型T::size_type,这就要求传递的参数类型有一个对应的成员size_type

```cpp
int a[10];
std::cout << len(a);	//OK only len() for array matchs
std::cout << len("hi");	//OK only len() for array matchs
std::vector<int> v;
std::cout << len(v);	//OK only len() for a type with size_type matchs
```

虽然第二个函数模版在使用int[10]和char[2]替换T的时候也能匹配,但是这会导致不存在返回类型size_type的潜在错误,于是第二个函数模版被忽略

当传递一个指针(两个模版都不匹配),编译期就会抱怨

```cpp
int *p;
len(p);	//Oops 两个模版都会被忽略
```

注意,这与传递具有size_type成员但是没有size()成员函数的对象不同,比如std::allocator<>

```cpp
std::allocator<int> alloc;
len(alloc);	//match the second function template, but Error
```

如果两个函数都不能匹配,那么我们可以提供一个最差匹配

```cpp
template<typename T, unsigned N>
std::size_t len(T (&)[N]) {
    return N;
}

//number of elements for a type have *size_type*
template<typename T>
typename T::size_type len(T const &t) {
    return t.size();
}

std::size_t len(...) {
    return 0;
}
```

这里,我们提供了一个通用len()函数,这个函数总是匹配,但是匹配最差,*关于在重载匹配中使用...,参阅Section C.2*

对于指针来说,第三个函数是可以匹配的,但是对于allocator,因为第二个函数是更好的匹配,所以仍然会因为没有size()而报错

*关于SFINAE的更多细节前往Section 15.7*

*关于SFINAE的一些应用前往Section 19.3*

**SFINAE and Overload Resolution**

随着时间推移,SFINAE的应用已经如此广泛以至于它已经成为一个动词.如果我们应用SFINAE机制保证函数模版在一定限制下被忽略,我们就可以说我们SFINAE了一个函数.

如果你在C++标准中读到一个函数模版"不应该参加重载解析(overload resolution), 除非",这意味着在一些情况下, SFINAE被用来"SFINAE out"这个函数模版

例如std::thread声明了一个构造函数

```cpp
namespace std{
	class thread {
	public:
        //...
        template<typename F, typename... Args>
        explicit thread(F&& f, Args... args);
		//...
    }
}
```

Remarks: This constructor shall not participate in overload resolution if `decay_t<F>` is the same type as std::thread

这意味着如果使用std::thread作为第一个也是唯一的参数调用模板构造函数,它将被忽略.原因是这样的模板成员函数有时可能比任何预定义的复制或移动构造函数更匹配*(请参阅Section 6.2和Section 16.2.4节了解详细信息)*.通过在调用线程时SFINAE'ing out构造函数模板,我们确保在从另一个线程构造线程时始终使用预定义的复制或移动构造函数
在具体情况的基础上应用这种技术可能很麻烦.幸运的是,标准库提供了更容易禁用模板的工具.这种特性中最著名的是std:: enable_if<>
因此,std::thread的实际声明通常是这样的

```cpp
namespace std{
	class thread {
	public:
        //...
        template<typename F, typename... Args,
        typename = std::enable_if_t<!is_same_v<std::decay_t<F>, thread>>>
        explicit thread(F&& f, Args... args);
		//...
    }
}
```

### 8.4.1 Expression SFINAE with *decltype* 带有*decltype*的SFINAE表达式

有的时候制定正确的SFINAE出函数模版不是那么容易的

对于上一节的三个函数,如何使得有size_type但是没有size()的类型在匹配第二个模版函数时被忽略

有一种常用的方法来解决这个问题

* 使用trailing return type syntax指定返回指定类型
* 使用decltype和逗号操作符定义返回类型
* 格式化所有表达式,使其在逗号表达式之前(将逗号前的表达式转换为void以免逗号表达式被重载)
* 在逗号操作符后定义实际返回类型的对象

```cpp
template<typename T>
auto len (T const& t) -> decltype((void)(t.size()), T::size_type() ) {
    return t.size();
}
```

decltype的操作数是一个逗号表达式,会返回最后一部分的结果,但是在逗号之前的表达式必须是有效的,本例中就是t.size(),这里使用(void)以避免逗号表达式被重载

注意decltype的操作数是一个未计算的操作数(unevaluated operand),这意味着我们可以创建虚拟对象(dummy object)而不调用构造函数, *见Section 11.2.3*

## 8.5 Compile-Time if 编译时if

偏特化,SFINAE,std::enable_if<>允许我们整体上开启或关闭模版. C++17引入了编译时if使得我们可以根据编译时条件启用或关闭语句

作为例子,考虑Section 4.1.1介绍的可变参数模版的print(),与单独提供一个函数结束递归不同,constexpr if允许我们局部决定是否继续递归

```cpp
template<typename T, typename ...Types>
void print(T firstArg, Types... args) {
    std::cout << firstArg << '\n';
	constexpr if(sizeof...(args) > 0) {
        print(args...);
    }	//Error!
}
```

这里如果只用一个参数调用print(),args将为空,这样print()的递归调用将被discard, 这部分代码将不会实例化.相应的函数不需要存在

这意味着只进行了第一阶段的翻译(definition time),只检查了语法和不依赖模版的名称 (*见Section 1.1.3*)

比如

```cpp
template<typename T>
void foo(T t) {
    if constexpr(std::is_integral_v<T>) {
        if(t > 0) {
            foo(t - 1);  //OK
        }
    }
    else {
        undeclared(t);  //error if not declared and not discarded(i.e. T is not integral)
        undeclared();   //error if not declared (even if discarded)
        static_assert(false, "no integral"); //always asserts (even if discarded)
        static_assert(std::is_integral_v<T>, "no integral"); //OK
    }
}
```

注意编译时if不止可以用于模版,也可以用于普通语句

利用这点,我们可以利用Section 8.2介绍的isPrime()函数

```cpp
template<typename T, std::size_t SZ>
void foo(std::array<T, SZ> const& coll) {
	if constexpr(!isPrime(SZ)) {
		//...
    }
    //...
}
```

*Section 14.6介绍了更多细节*

## 8.6 Summary

* 模板提供了在编译时进行计算的能力(使用递归进行迭代(使用偏特化或operator?: 进行选择).
* 使用constexpr函数,我们可以将大多数编译时计算替换为可在编译时于上下文中调用的“普通函数”.
* 使用偏特化,我们可以根据特定的编译时约束在类模板的不同实现之间进行选择.
* 模板只在需要的时候使用,在函数模板声明中的替换不会报错.这个原则被称为SFINAE
* SFINAE可以用于为特定类型和约束提供函数模板.
* 自c++ 17以来,编译时if允许我们根据编译时条件启用或丢弃语句.