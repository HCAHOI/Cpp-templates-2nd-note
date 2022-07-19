# 11 Generic Libraries 泛型库

到目前为止，我们关于模板的讨论主要侧重于从一个立刻完成的任务或者应用(也就是从应用程序员的角度)出发的特性、功能和约束。然而，当用于写通用库和框架时模板才是最高效的，此时我们的设计必须考虑潜在的使用方法，这通常在很大程度上是没有限制的。尽管本书的所有内容适用于这些设计，但是当写对于未知类型的可移植组件时，有一些通用的问题需要考虑。

这里没有包含所有的情形，但是总结了到目前为止介绍的一些特性, 介绍了一些其他特性, 一些本书将涵盖的特性。我们希望这会是通读本书后面章节的巨大动力。

 ## 11.1 Callables 可调用对象

许多库包含客户代码传递将被调用的实体的接口。这样的例子包括在另一个线程中进行调度的运算, 一个描述如何将值进行哈希并存入哈希表中的函数, 一个描述对集合中元素进行排序的对象, 一个提供一些默认实参值的通用封装。标准库在此处也没有例外:它定义了许多以这些可调用实体为输入的组件.

这样的一个术语称为**回调(callback)**。传统地，这个术语已经保留为将函数作为调用实参的实体(与模板实参不同)，我们也保留了这个传统。比如，一个排列函数可能包含一个回调参数作为排列标准，它被调用用于使得一个元素以期望的顺序排在另一个元素之前。

C++中，有多个类型适合回调，因为他们既可以作为函数调用实参进行传递，也可以直接使用语法`f(...)`进行调用:

- 指向函数类型的指针
- 重载了运算符`()`的class type(有时称为函子functors)，包括lambda表达式
- 有一个可以推导出指向函数的指针或者指向函数引用的转换函数的class type

共同地，这些类型都被称为函数对象类型(function object type)，这样类型的值就是函数对象(function object)。

C++标准库引入了比callable type更广泛些的概念，这要么是函数对象类型或者指向成员的指针。可调用类型的对象(An object of callable type)称为可调用对象(callable object)，处于方便，我们通常称为callables。

能够接收任何类型的可调用对象的特性对泛型代码非常有益，模板使这成为可能.

## 11.1.1 Supporting Function Objects 支持函数对象

首先看看标准库的`for_each()`算法是如何实现的(使用我们自己的名字foreach来避免名字冲突，并且处于简洁性考虑，我们不返回任何东西):

```cpp
//foreach.hpp
template<typename Iter, typename Callable>
void foreach (Iter current, Iter end, Callable op)
{
    while (current != end) { //as long as not reached the end
        op(*current); // call passed operator for current element
        ++current; // and move iterator to next element
    }
}
```

下面的程序示范了不同函数对象使用该模板:

```cpp
#include <iostream>
#include <vector>
#include "foreach.hpp"

// a function to call:
void func(int i) { 
    std::cout << "func() called for: " << i << ’\n’;
} 

// a function object type (functor) (for objects that can be used as functions):
class FuncObj {
public:
    void operator() (int i) const 
    { //Note: const member function
        std::cout << "FuncObj::op() called for: " << i << '\n';
    }
};

int main() {
    std::vector<int> primes = { 2, 3, 5, 7, 11, 13, 17, 19 };
    
    foreach(primes.begin(), primes.end(), // range
            func); // function as callable (decays to pointer)
    foreach(primes.begin(), primes.end(), // range
            &func); // function pointer as callable
    foreach(primes.begin(), primes.end(), // range
            FuncObj()); // function object as callable
    foreach(primes.begin(), primes.end(), // range
            [] (int i) { //lambda as callable
                std::cout << "lambda called for: " << i << '\n';
            });
}
```

让我们看看每一个例子的细节:

* 当传入函数作为函数实参，实际传入的并不是函数本身，而是指向它的指针或者引用。与数组一样，当以值方式传递时，函数实参会退化为指针，并且当参数类型为模板参数时会推导出指向函数类型的指针。

  与数组一样，当以引用方式传递时，函数不会退化。然而，函数类型不能使用const限定。如果声明`foreach()`的最后参数类型为可调用的const&，const将被忽略(通常来说，函数引用很少在主流C++代码中使用)。

* 第二个调用显式地以函数指针为输入，通过传递函数名字地地址。这等效于第一个调用(函数名字隐式地退化为指针值)，但是可能更清晰。

* 当传递函子(functor)时，我们传递class type对象作为可调用对象。通过class type调用通常等效于调用它的`()` 运算。因此调用`op(*current)`通常转化为

```cpp
op.operator()(*current); // call operator() with parameter *current for op
```

注意，当定义`()`运算时，应该定义它为constant成员函数。不然当框架或者库期望这个调用不改变传入对象的状态时，可能引发微妙的错误信息 。

对于class type对象，隐式地转化为调用surrogate call function的指针或者引用也是可能的。在这种情形下，调用`op(*current)`将会转化为` (op.operator F())(*current)` , 其中F是可以由class type转换的pointer-to-function或者reference-to-function。这个不大常见。

### 11.1.2 Dealing with Member Functions and Additional Arguments 处理成员函数和额外的参数

一种可能的调用实体在前面的例子中没有使用:成员函数。这是因为调用非静态成员函数通常牵涉到指定一个调用时使用的对象，使用类似`object.memfunc(...)`或者`ptr->memfunc(...)`的语法，这与通常的`func(...)` 模式不匹配。

幸运的是，C++17以后，C++标准库提供了有用的工具`std::invoke()`，用普通函数的调用语法方便地统一了该情形，这样仅使用一种形式调用任何可调用对象。使用`std::invoke()`的`foreach()`模板可以实现如下:

```cpp
template <typename Iter, typename Callable, typename... Args>
void foreach (Iter current, Iter end, Callable op, Args&... args) {
    while(current != end) {
        std::invoke(op, args..., *current);	//如果在op后只有一个参数,那么就是*curent,args为空,否则成员函数所在的对象应该在可调用对象参数之后
        current++;
    }
}
```

此处，除了可调用参数，我们也接受任意数量的额外参数，`foreach()`模板然后使用给定的可调用对象，紧跟着额外的给定参数和被引用的元素，调用`std::invoke()`。 `std::invoke()`这样处理:

- 如果可调用对象是指向成员的指针，它使用第一个额外参数作为该成员所在的对象。所有剩余的参数以实参传递给可调用对象。
- 否则，所有额外参数都以实参传递给可调用对象。

注意，这里不能完美转发可调用对象和额外的参数:第一次调用可能会窃取他们的值，导致后面的迭代中以意想不到的方式调用可调用对象。

基于该实现，我们仍然可以编译原始调用给上面的`foreach()`。现在，我们可以传递额外的实参给可调用对象，并且该可调用对象可以是成员函数(`std::invoke()`允许指向数据成员的指针为回调类型(callback type)。与调用一个函数不同，它返回额外实参指向的对象的对应数据成员的值)。以下用户代码展示了这一点：

```cpp
#include <iostream>
#include <vector>
#include <string>
#include "foreachinvoke.hpp"

// a class with a member function that shall be called
class MyClass {
public:
    void memfunc(int i) const {
        std::cout << "MyClass::memfunc() called for: " << i << '\n';
    }
};

int main()
{
    std::vector<int> primes = { 2, 3, 5, 7, 11, 13, 17, 19 };
    // pass lambda as callable and an additional argument:
    foreach(primes.begin(), primes.end(), //elements for 2nd arg of lambda
        [](std::string const& prefix, int i) { //lambda to call
            std::cout << prefix << i << '\n';
        },
        "- value:"); //1st arg of lambda
// call obj.memfunc() for/with each elements in primes passed as argument
    MyClass obj;
    foreach(primes.begin(), primes.end(), //elements used as args
        &MyClass::memfunc, //member function to call
        obj); // object to call memfunc() for
}
```

第一个调用`foreach()`中传递第四个参数("-value: "字符串)给lambda表达式的第一个参数， 而vector中的当前元素绑定给lambda表达式的第二个参数。第二个调用传递成员函数`memfunc()`为调用的第三个实参，obj传递为第四个实参。

关于std::invoke()是否可以使用可调用对象的type traits,参见*Section D.3.1*

### 11.1.3 Wrapping Function Calls 包装函数调用

`std::invoke()`的一个常见应用是包装单个函数调用(比如，给调用打日志，测量时间，或者开始一个新线程前做准备工作)。现在，我们可以通过完美转发可调用对象和所有传入的参数来支持移动语义:

```cpp
#include <utility> // for std::invoke()
#include <functional> // for std::forward()
template<typename Callable, typename... Args>
decltype(auto) call(Callable&& op, Args&&… args)
{
    return std::invoke(std::forward<Callable>(op), //passed callable with
                       std::forward<Args>(args)...); // any additional args
}
```

另一个有趣的方面是如何处理被调用函数的返回值来完美转发给调用者。为了支持返回引用(比如`std::ostream&`)，必须使用`decltype(auto)`代替`auto`:

```cpp
template<typename Callable, typename... Args>
decltype(auto) call(Callable&& op, Args&&... args)
```

`decltype(auto)` ( C++14以后提供) 是个从相关表达式类型(初始化器、返回值或者模板实参)推断变量类型,返回类型和模板实参的占位符类型，如果希望暂时保存`std::invoke()`的返回值到一个变量中，在进行一些操作(比如处理返回值或者给调用打日志)后返回它，你必须使用`decltype(auto)`声明临时量:

```cpp
decltype(auto) ret{std::invoke(std::forward<Callable>(op),
                               std::forward<Args>(args)...)};
...
return ret;
```

注意，声明ret为`auto&&`是不正确的。作为引用，`auto&&`延长了返回值的寿命到范围结束，但是没有超出返回给函数调用者的语句。

```cpp
template<typename T>
auto&& foo() {
    T x = 1;
    return x;
}

int main() {
    auto x = foo<int>();
    x = 10;	//SIGSEGV Error!
}
```

```cpp
template<typename T>
decltype(auto) foo() {
    T x = 1;
    return x;
}

int main() {
    auto x = foo<int>();
    x = 10;	//OK
}
```

然而，使用`decltype(auto)`也有问题：如果可调用对象的返回类型为void，ret初始化为`decltype(auto)`是不允许的，因为void不是完整的类型。你有以下选择：

- 在该语句之前声明一个对象，它的析构函数执行你想要实现的看得见的行为。比如:

  ```cpp
  struct cleanup {
  ~cleanup() {
      //...code to perform on return
      }
  } dummy;
  return std::invoke(std::forward<Callable>(op),
                     std::forward<Args>(args)...);
  ```

- 针对void和non-void情形分别实现:

```cpp
#include <utility> // for std::invoke()
#include <functional> // for std::forward()
#include <type_traits> // for std::is_same<> and invoke_result<>

template<typename Callable, typename... Args>
decltype(auto) call(Callable&& op, Args&&... args)
{
    if constexpr(std::is_same_v<std::invoke_result_t<Callable, Args...>,
                                void>) {// return type is void:
        std::invoke(std::forward<Callable>(op),
                    std::forward<Args>(args)...);
        ... 
        return;
    }
    else {
        // return type is not void:
        decltype(auto) ret{std::invoke(std::forward<Callable>(op),
                                       std::forward<Args>(args)...)};
        ...
        return ret;
    }
}
```

其中，

```cpp
if constexpr(std::is_same_v<std::invoke_result_t<Callable, Args...>,
                            void>)
```

在编译时测试使用Args...对可调用对象进行调用的返回类型是否是void。*前往Section D.3.1 见std::invoke_result<>的细节*

将来的C++版本可能避免对void的特殊处理。(*见Section 17.7*)

## 11.2 Other Utilities to Implement Generic Libraries 实现泛型库的其他实用程序

`std::invoke()`仅仅是C++标准库提供的用于实现泛型库的常用程序之一 。后面，将涉及其他重要的程序。

### 11.2.1 Type Traits 类型特性

标准库提供了各种各样的称为**类型特性(type traits)**的程序，允许我们评估和修改类型。它支持不同的情况，其中泛型代码必须对实例化的类型进行适应或者做出反应。比如:

```cpp
#include <type_traits>
template<typename T>
class C
{
    // ensure that T is not void (ignoring const or volatile):
    static_assert(!std::is_same_v<std::remove_cv_t<T>,void>,
                  "invalid instantiation of class C for void type");
public:
    template<typename V>
    void f(V&& v) {
        if constexpr(std::is_reference_v<T>) {
            ... // special code if T is a reference type
        };
        if constexpr(std::is_convertible_v<std::decay_t<V>,T>) {
            ... // special code if V is convertible to T
        };
        if constexpr(std::has_virtual_destructor_v<V>) {
            ... // special code if V has virtual destructor
        }
    }
};
```

如这个例子显式，通过检查特定的条件，可以在模板的不同实现之间选择。此处，我们使用编译时if特性(C++17以后提供)，但是我们也可以使用`std::enable_if`进行偏特化，或者SFINAE来开启或者关闭帮助类模板。

然而，类型特性必须小心使用：他们可能和初级程序员预期的表现不一样。比如:

```cpp
std::remove_const_t<int const&> // yields int const&
```

此处，因为引用不是const(尽管你不能修改)，该调用没有影响，最后得到的是传入的类型。

结果便是移除引用和const的顺序很关键：

```cpp
std::remove_const_t<std::remove_reference_t<int const&>> // int
std::remove_reference_t<std::remove_const_t<int const&>> // int const
```

反而，你可以仅仅这样调用：

```cpp
std::decay_t<int const&> // yields int
```

然而，这也会将原始数组或者函数转化为相应的指针类型。

也有一些情形类型有条件。不满足这些条件将导致未定义的行为。比如

```cpp
make_unsigned_t<int> // unsigned int
make_unsigned_t<int const&> // undefined behavior (hopefully error)
```

有时候，结果可能出乎意料，比如：

```cpp
add_rvalue_reference_t<int> // int&&
add_rvalue_reference_t<int const> // int const&&
add_rvalue_reference_t<int const&> // int const& (lvalue-ref remains lvalue-ref)
```

此处，希望`add_rvalue_reference`总是导出右值引用，但是引用折叠(reference-collapsing)规则引起左值引用和右值引用生成左值引用。

另一个例子：

```cpp
is_copy_assignable_v<int> // yields true (generally, you can assign an int to an int)
is_assignable_v<int,int> // yields false (can’t call 42 = 42)
```

`is_copy_assignable`仅仅通常检查是否可以将ints赋给另一个(检查左值运算)，但是`is_assignable`考虑值类型(检查是否可以将一个prvalue赋给另一个prvalue)。也就是说，第一个表达式等效于:

```cpp
is_assignable_v<int&,int&> // yields true
```

同样的理由：

```cpp
is_swappable_v<int> // yields true (assuming lvalues)
is_swappable_v<int&,int&> // yields true (equivalent to the previous check)
is_swappable_with_v<int,int> // yields false (taking value category into account)
```

由于所有这些原因，小心点注意type traits的准确定义。

*我们会在附录D介绍一些基础type traits的实现细节*

### 11.2.2 std::addressof()

`std::addressof<>()`函数模板导出对象或者函数的实际地址。甚至当对象类型有重载运算符&时仍然可以工作。 尽管后者非常罕见，但也会发生(比如智能指针)。因此，如果想要得到任何类型的对象地址，推荐使用std::addressof():

```cpp
template<typename T>
void f (T&& x)
{
    auto p = &x; // might fail with overloaded operator &
    auto q = std::addressof(x); // works even with overloaded operator &
    //...
}
```

### 11.2.3 std::declval()

`std::declval<>()`函数模板可以用来当作特殊类型引用对象的占位符。该函数没有定义，因此也不能被调用(也不创建对象)。因此，它仅可以用于不计算的运算(比如`decltype`和`sizeof`的构建)。因此你可以假设你有对应类型的对象。

std::declval的功能：返回某个类型T的右值引用，不管该类型是否有默认构造函数或者该类型是否可以创建对象。
-返回某个类型T的右值引用 这个动作是在编译时完成的，所以很多人把std::declval也称为编译时工具。

比如，以下声明从传入模板参数T1和T2中推导默认返回类型RT:

```cpp
#include <utility>
template<typename T1, typename T2,
    typename RT = std::decay_t<decltype(true ? std::declval<T1>() 
                                             : std::declval<T2>())>>
RT max (T1 a, T2 b)
{
    return b < a ? a : b;
}
```

为了避免必须调用T1和T2的默认构造函数使得调用表达式中的运算`? :` 来初始化RT，我们使用`std::declval`来“使用”对应类型的对象而不创建他们。这仅仅在decltype的不计算的上下文中才可能。

不要忘记使用`std::decay<>`来确保默认返回类型不是引用，因为`std::declval()`本身得到的是右值引用。不然，诸如`max(1,2)`的调用会得到类型`int&&`。

*前往Section 19.3.4看细节*

## 11.3 Perfect Forwarding Temporaries 完美转发临时对象

可以使用*forwarding references*和`std::forward<>`来完美转发泛型参数:

```cpp
template<typename T>
void f (T&& t) // t is forwarding reference
{
    g(std::forward<T>(t)); // perfectly forward passed argument t to g()
}
```

然而，有时候必须转发泛型代码中不是来自于参数的数据。这种情况，可以使用`auto&&`来创建可以被转发的变量。比如，假设链式调用函数`get()`和`set()`，其中`get()`的返回值应该被完美转发给`set()`:

```cpp
template<typename T>void foo(T x)
{
    set(get(x));
}
```

进一步假设我们需要更新代码来执行对`get()`产生的中间量的运算，我们通过声明为`auto&&`的变量持有这个值:

```cpp
template<typename T>
void foo(T x)
{
    auto&& val = get(x);
    ...//  perfectly forward the return value of get() to set():
    set(std::forward<decltype(val)>(val));
}
```

这避免了中间值的无关拷贝。

## 11.4 References as Template Parameters 引用作为模板参数

尽管不常见，但是模板类型形参也可以是引用类型:

```cpp
#include <iostream>
template<typename T>
void tmplParamIsReference(T) 
{
    std::cout << "T is reference: " << std::is_reference_v<T> <<'\n';
} 

int main()
{
    std::cout << std::boolalpha;
    int i;
    int& r = i;
    tmplParamIsReference(i); // false
    tmplParamIsReference(r); // false
    tmplParamIsReference<int&>(i); // true
    tmplParamIsReference<int&>(r); // true
}
```

尽管引用变量传递给了`tmplParamIsReference()`，模板形参T推断为引用指向的类型。然而，我们可以通过显式指定类型T强制为引用情形:

```cpp
tmplParamIsReference<int&>(r);
tmplParamIsReference<int&>(i);
```

这在根本上改变了模板的行为， 说不定，一个没有被设计成这种可能性的模板，却因此出发了错误或者预期外的行为：

```cpp
// basics/referror1.cpp
template<typename T, T Z = T{}>
class RefMem 
{
private:
    T zero;
public:
    RefMem() : zero{Z} { }
};

int null = 0;

int main()
{
    RefMem<int> rm1, rm2;
    rm1 = rm2; // OK

    RefMem<int&> rm3; // ERROR: invalid default value for Z
    RefMem<int&, 0> rm4; // ERROR: invalid default value for Z

    extern int null;
    RefMem<int&,null> rm5, rm6;
    rm5 = rm6; // ERROR: operator= is deleted due to reference member
}
```

这里有一个类型为模板参数类型T的成员，初始化为非类型模板参数Z(默认零初始化)。用int实例化该类和预期的行为一样。然而，当用引用进行实例化，事情变得蹊跷:

- 默认值初始化失败
- 不再能仅仅传递0进行初始化
- 意外的是，赋值操作也不再可用，因为**存在非静态引用类型成员的类删除了默认的赋值操作**

给非类型模板形参使用引用类型要非常谨慎，可能非常危险:

```cpp
#include <vector>
#include <iostream>

template<typename T, int& SZ> // Note: size is reference
class Arr {
private:
    std::vector<T> elems;
public:
    Arr() : elems(SZ) { }//use current SZ as initial vector size

    void print() const 
    {
        for (int i=0; i<SZ; ++i)  //loop over SZ elements
        {  
            std::cout << elems[i] << ' ';
        }
    }
};

int size = 10;
int main()
{
    Arr<int&,size> y; // compile-time ERROR deep in the code of class std::vector<>
    Arr<int,size> x; // initializes internal vector with 10 elements
    x.print(); // OK
    size += 100; // OOPS: modifies SZ in Arr<>
    x.print(); // run-time ERROR: invalid memory access: loops over 120 elements
}
```

这里，用引用类型实例化Arr会引发`std::vector<>`内部代码的错误，因为它的成员不能实例化为引用：

```cpp
Arr<int&,size> y; // compile-time ERROR deep in the code of class std::vector<>
```

更糟糕的是由于size参数为引用导致的运行时错误:它允许值的改变而容器没有感知(也就是size值变得无效)。因此，使用size(就像`print()`成员)必然导致未定义的行为(因此程序崩溃，甚至更糟糕):

```cpp
int int size = 10;
...
Arr<int,size> x; // initializes internal vector with 10 elements
size += 100; // OOPS: modifies SZ in Arr<>
x.print(); // run-time ERROR: invalid memory access: loops over 120 elements
```

注意，改变模板形参SZ为`int const&`并不解决该问题，因为size本身是可以改变的。

毫无疑问，这个例子很牵强附会。然而在更复杂的情形下，这样的问题经常发生。同样，C++17中，非类型形参也可以推导，比如：

```cpp
template<typename T, decltype(auto) SZ>
class Arr;
```

使用`decltype(auto)`也能轻易生成引用类型，因此在这种情况下也通常要避免(默认使用auto)

出于这个原因，C++标准库有时候有令人意外的指定和限制，比如：

- 尽管模板形参实例化为引用，为了有赋值操作，类`std::pair<>`和`std::tuple<>`实现了赋值运算而不是使用默认行为，比如：

```cpp
namespace std {
template<typename T1, typename T2>
struct pair {
        T1 first;
        T2 second; 
        ...
        // default copy/move constructors are OK even with references:
        pair(pair const&) = default;
        pair(pair&&) = default;
        ...
        // but assignment operator have to be defined to be available with references:
        pair& operator=(pair const& p);
        pair& operator=(pair&& p) noexcept(…);
        ...
	};
}
```

- 因为可能的副作用导致的复杂性，用引用类型进行实例化`C++17`标准库类模板`std::optional<>`和`std::variant<>` 是不合法的(至少在C++17中)。

  禁止引用，简单的static assertion就足够:

```cpp
template<typename T>
class optional
{
    static_assert(!std::is_reference<T>::value,
                 "Invalid instantiation of optional<T> for references");
    ...
};
```

引用类型通常与其他类型很不像，受制于一些特殊的语言规则。这会影响调用形参的声明和定义类别特性的方式。

## 11.5 Defer Evaluations 推迟评估

当实现模板，有时候需要处理不完整的类型：

```cpp
template<typename T>
class Cont {
private:
    T* elems;
public:
    ...
};
```

到目前为止，这个类可以使用不完整的类型。这很有用，比如，类指向自身类型的元素:

```cpp
struct Node
{
    std::string value;
    Cont<Node> next; // only possible if Cont accepts incomplete types
};
```

然而，仅使用一些特性，就会丧失使用不完全类型的能力:

```cpp
template<typename T>
class Cont {
private:
    T* elems;
public:
    ...
typename
    std::conditional<std::is_move_constructible<T>::value,
                     T&&,
                     T&
                    >::type
    foo();
};
```

此处，使用`std::conditional`特性来决定成员函数`foo()`的返回类型是`T&&`还是`T&`。该决定依赖于模板形参类型T是否支持移动语义。

该问题是特性`std::ismoveconstructible`需要它的实参为完整类型(不能是void和未知边界的array, *见Section D.5*)。因此，因为`foo()`的定义的存在，声明struct node失败。(如果`std::is_move_constructible`不是完整的类型，不是所有的编译器都会导致错误，因为对于这种错误的诊断不是必须的。因此这至少是个一致性问题)。

我们可以通过使用成员模板替代`foo()` 来解决这个问题，这样`std::is_move_constructible`的评估就被推迟到实例化`foo()`的时间点：

```cpp
template<typename T>
class Cont 
{
private:
    T* elems;
public:
    template<typename D = T>
    typename std::conditional<std::is_move_constructible<D>::value,
                              T&&,
                              T&
                              >::type
    foo();
};
```

现在，该特性依赖于模板参数D(默认为T，我们想要的任何值)，在评估之前，编译器必须等直到`foo()`为一个类似于Node的完整类型调用。(此时，Node才是个完整的类型, 它在定义时是不完整的)

## 11.6 Things to Consider When Writing Generic Libraries 写泛型库时必须考虑的事情

这里列出实现泛型库时要记住的事(注意，其中一些在这本书的后面介绍):

- 使用转发引用来转发模板中的值(*Section 6.1*)。如果该值不依赖于模板参数，使用`auto&&` (*Section 11.3*)
- 当参数声明为转发引用，当传递左值时，做好模板参数为引用类型的准备
- 当需要依赖于模板参数的对象的地址时，使用`std::addressof()`来避免当它绑定到一个重载了运算符`&`的类型时发生的意外(*Section 11.2.2*)
- 对于成员函数模板，确保他们没有更好地匹配预定义的拷贝/移动构造或者赋值运算(*Section 6.4*)
- 考虑使用`std::dacay`当模板参数可能是字符串常量并且不是以值方式传递(*Section 7.4*)
- 如果有依赖于模板形参的`out`或者`inout`参数 ，准备好处理const模板实参可能被传递的情形(*Section 7.2.2*)。
- 准备好处理模板形参为引用导致的副作用的情形(*Section 11.4*)。特别地，你可能想要确保返回类型不能是引用(*Section 7.5*)
- 准备好处理对不完整类型的支持，比如递归数据结构(*Section 11.5*)
- 重载所有数组类型，不只是`T[SZ]`(*Section 5.4*)

## 11.7 Summary

* 模板允许您传递函数, 函数指针, 函数对象,函子(仿函数, functor)和lambda表达式作为callable。
* 当在定义类时重载了operator()时, 将其声明为const (除非调用改变了它的状态)。
* 使用std:: invoke(), 您可以实现可以处理所有可调用对象的代码，包括成员函数。
* 使用decltype(auto)来完美地转发返回值。
* type traits是检查类型属性和功能的类型函数。
* 当需要模板中对象的地址时，使用std:: addressof()。
* 使用std::declval()在未计算的表达式中创建特定类型的值。
* 如果对象的类型不依赖于模板参数, 可以使用auto&&在泛型代码中完美地转发对象。
* 准备好处理模板参数为引用类型时的副作用。
* 您可以使用模板来延迟表达式的求值(例如，支持在类模板中使用不完整类型)。