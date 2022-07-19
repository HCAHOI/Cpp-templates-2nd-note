---
typora-copy-images-to: images
typora-root-url: images
---

# 9 Using Templates in Practice 在实践中使用模版

模板代码与普通代码略有不同。在某些方面，模板介于宏和普通(非模板)声明之间。尽管这可能是一种过度简化，但它不仅影响了我们使用模板编写算法和数据结构的方式，而且影响了我们在日常工作中表达和分析涉及模板的程序。
在本章中，我们将讨论其中的一些实例，而不必深入研究它们背后的技术细节。这些细节将在第14章中进行探讨。为了简化讨论，我们假设我们的c++编译系统由相当传统的编译器和链接器组成(不属于这一类的c++系统很少)。

## 9.1 The Inclusion Model 包含模型

有多种方式组织模板源代码。本节展示了最流行的方法：包含模型（the inclusion model）

### 9.1.1 Linker Errors 链接错误

多数C和C++程序员很大程度上以如下方式组织非模板代码：

* 类和其他类型声明完全放在头文件中。通常，这是一个拓展名为.hpp(或者 .H ,  .h , .hh,  hxx )的文件
* 对于全局(noninline)变量和(noninline)函数，只有声明放在头文件中，而定义放在另一个文件中，这个文件编译为另一个编译单元。这样的CPP文件通常是一个拓展名为.cpp(或者.C , .c , .cc 或者 .cxx)的文件。

这很好：这使得被需要的类型定义在整个程序中可轻易获得，并且从编译器角度看这避免了变量和函数的重复定义的错误。

牢记这些约定，模板初级程序员的一个常见错误如以下的错误的程序所示，像普通代码一样在头文件中声明模板：

```cpp
#ifndef MYFIRST_HPP
#define MYFIRST_HPP

// declaration of template
template<typename T>
void printTypeof(T const& x);

#endif
```

printTypeof()是一个简单的辅助函数的声明，打印一些类型信息。它的实现放在了一个CPP文件中:

```cpp
#include <iostream>
#include <typeinfo>
#include "myfirst.hpp"

// implementation/definition of template
template<typename T>
void printTypeof(T const& x)
{
    std::cout << typeid(x).name() << '\n';
}
```

这个例子使用typeid操作来打印一个字符串，这个字符串描述了传递给它的表达式的类型。它返回静态类型std::type_info的左值，std::type_info提供一个成员函数name()，描述一些表达式的类型。C++标准并没有实际表明name()必须返回有意义的内容，但是好的C++实现应该给出传递给typeid的表达式的类型的适当的描述。

最后，我们在另一个CPP文件中使用该模板，我们的模板声明被#included到该文件:

```cpp
#include "myfirst.hpp"

// use of the template
int main()
{
    double ice = 3.0;
    printTypeof(ice);  // call function template for type double
}
```

C++编译器很可能会接受该程序而没有任何问题，但是链接器会有可能报告问题，暗指没有函数printTypeof()的定义。

该错误的原因是函数模板printTypeof()的定义没有被实例化。为了让一个模板实例化，编译器必须知道实例化成什么样的定义、应该为何种模板实参进行实例化。不幸的是，在前面的例子中，这两条信息被独立地编译。因此，当编译器看到printTypeof()的调用时，但没有看到该函数为double类型的定义。它仅仅假设该定义在别处提供，然后建立一个定义的引用（让链接器来解析）。另一方面，当编译器处理myfirst.cpp文件时，没有任何指示说在该点必须为特定类型进行实例化模板定义。

### 9.1.2 Templates in Header Files 在头文件中的模板

前述问题的通用解决方案与宏和inline函数的方法相同：将模板的定义放在模板声明的头文件中。

这样，不再需要myfirst.cpp文件，我们重写myfirst.hpp，让其包含模板声明和模板定义:

```cpp
#ifndef MYFIRST_HPP
#define MYFIRST_HPP

#include <iostream>
#include <typeinfo>

// declaration of template
template<typename T>
void printTypeof(T const& x);

// implementation/definition of template
template<typename T>
void printTypeof(T const& x)
{
    std::cout << typeid(x).name() << '\n';
}

#endif  // MYFIRST_HPP
```

这样组织模板的方式称为包含模型(inclusion model)。这样就可以正确编译、链接和执行。

在这个地方可以有一些发现。最显著的是该方法显著增加了包含该头文件myfirst.hpp的代价。在这个例子中，这个代价不是来自于模板定义本身的大小，而是由于我们在模板定义中包含了其他使用的头文件，这个例子中是`<iostream>`和`<typeinfo>`。你会发现这相当于几万行代码，因为像`<iostream>`这样的头文件本身包含许多模板定义。

实践中这是一个现实的问题，因为这大大增加了编译器编译大型程序的时间。我们将会检查一些可能的方法来处理这个问题，包括预处理头文件(*Section 9.3*), 显式模板实例化(*Section 14.5*)和Pimpl机制(Pointer to Implementation *见《Effective C++》*)

尽管存在这个构建时间问题，我们仍然建议尽可能使用包含模型来组织模板，直到有更好的机制可供选择。在写这本书的时候是2017年，这种机制称为模块(modules)。这种语言机制允许程序员更有逻辑地组织代码，让编译器能够分离编译所有的声明，然后在任何需要的时候有效地、有选择地导入处理好的声明。(Modules机制已经引入C++20)

**TODO: (Modules的介绍写在这里)**

关于包含模型的另一个更微妙的发现是noninline函数模板与inline函数和宏有一个重要的区别：noninline函数模板不会在调用处展开。相反当他们被实例化时，他们创建函数的一份新的拷贝。由于这是一个自动的过程，编译器可以终止在两个不同的文件中创建两个副本，一些编译器当发现同一个函数有两个不同的定义时会发布错误。理论上这不是我们应该关心的，这是C++编译系统来处理的问题。大系统会创建他们自己代码的库，然而问题会偶然出现。在第14章将会讨论实例化的模式，更细致地学习C++翻译系统(编译器)的文档对解决这些问题会有帮助。

最后，我们需要指出我们例子中适用于普通函数模板的方法也适用于类模板的成员函数和静态数据成员，也适用于成员函数模板。

## 9.2 Templates and *inline* 模版和*inline*

声明函数为inline是一个常用的手段来优化程序的运行时间。inline说明符是一种给实现的提示，即在调用处更倾向于函数体的内联替换，而不是普通函数的调用机制。

然而，实际实现可能会忽略这个提示。因此，inline唯一的保证效果就是允许函数定义在程序中出现多次（通常由于它出现在头文件中，而头文件在多个地方被包含）。

就像inline函数，函数模板也可以在多个编译单元中定义。这通过将定义放在头文件，然后该头文件被多个CPP文件包含的方式达到。

然而，这并不意味着函数模板默认使用inline替换。是否在调用点使用inline替换函数模板主体、何时用inline替换函数模板实体，而不是采用普通的函数调用机制，这些都完全取决于编译器。出人意料的是，编译器通常比程序员更善于估计内联展开一个调用是否会导致性能提升。结果就是，每个编译器针对inline的精确的编译器策略不尽相同，甚至依赖于编译选项的选择。

然而，使用合适的性能检测工具，程序员可以有比编译器更好的信息，并且有可能覆盖编译器的决定（比如，当针对特定的平台优化软件，比如移动手机或者特定的输入）。有时候这仅仅通过编译器规定的特性才可能实现，比如noninline或者always_inline。

值得指出的是，函数模板的全特化和普通函数一样:他们的定义只能出现一次，除非他们被定义为inline。*见Section16.3*

*关于这个主题,前往附录A看更多细节*

## 9.3 Precompiled Headers 预编译头文件

即使没有使用模板，C++头文件也可以变得非常庞大，然后导致编译时间很长。模板加重了这种倾向，等待的程序员发出了强烈抗议迫使编译器厂商在许多情形下实现了一种称为“预编译头文件”(precompiled header)的机制。这种机制超出了标准库的范畴，依赖于各厂商指定的编译选项。尽管我们将如何创建和使用预编译头文件的细节留给了不同编译系统的文档，但是理解它是如何工作是非常有用的。

当编译器翻译一个文件时，它从头到最后都是这么做的：当它处理文件中的每一个标记（有可能来自于被包含的文件中），它改变自身状态，包括在符号表中添加条目便于后续查找。当这样做时，编译器会在目标文件（object files）中生成代码。

预编译头文件机制依赖于代码可以用这样的机制进行组织的事实，就是许多文件从相同的一行代码开始。我们假设每个有待编译的文件都从相同的N行代码开始，我们可以编译这N行代码，然后保存编译器当时的完整状态到一个预编译头文件中。然后，程序中的每一个文件都可以重新载入保存的状态，然后从第N+1行开始编译。这里需要注意的是，重载保存好的状态操作比编译这N行代码快好几个数量级。然而，第一次保存这个状态通常比只编译这N行代码代价要高。增加的代价大概从20%到200%。

有效使用预编译头文件的关键是确保尽可能多的文件从最大数量的共同的代码开始。实践中这意味着文件从#include指令开始，也就是之前提到的消耗了大部分编译时间的部分。因此，注意包含头文件的顺序可能是非常有益的。比如:

```cpp
#include <vector>
#include <list>
//...
```

和

```cpp
#include <list>
#include <vector>
//...
```

阻止了预编译头文件，因为源文件中没有共同的初始状态。

一些程序员决定最好#include一些额外的不需要的头文件，而不是错过使用预编译头文件加速一个文件的编译单元的机会。这个决策可以大大减轻包含策略的管理。比如直接地创建一个名为`std.hpp`的头文件，这个头文件包含所有的标准头文件：

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <deque>
#include <list>
//...
```

这个文件可以进行预编译，然后每一个程序文件可以简单地使用头文件库：

```cpp
#include "std.hpp"
```

通常，这将花费一段时间进行编译，但是如果系统有足够的内存，这种预编译头文件机制比包含标准库的一个头文件（没有使用预编译的）的编译速度快很多。这种方法的标准库头文件特别方便，因为他们很少改变，因此预编译头文件std.hpp可以只构建一次。不然预编译头文件通常依赖于项目的配置（比如由make工具或者集成开发环境IDE项目构建工具进行按需更新）。

一个管理预编译头文件有吸引力的方式是创建预编译头文件层，从最广泛使用的、最稳定的头文件（比如我们的std.hpp头文件）到不期望经常改变但仍然值得预编译的头文件。但如果头文件仍在大规模开发，为他们创建预编译头文件会消耗更多的时间。该方法的一个核心的观念是更稳定层的预编译头文件的复用可以比不稳定头文件更能提高编译效率。比如，假设除了std.hpp头文件（已经预编译好了）之外，我们还定义了一个core.hpp头文件，包含项目的额外设施，但是达不到一定级别的稳定性：

```cpp
#include "std.hpp"
#include "core_data.hpp"
#include "core_algos.hpp"
//...
```

因为这个文件从#include "std.hpp"开始，编译器可以载入相关的预编译好的头文件，然后继续后面代码，而不需要重新编译所有的头文件。当该文件完全处理后，一个新的预编译头文件产生。然后应用可以使用`#include "core.hpp"`来提供快速访问大量函数的途径，因为编译器可以载入后面的预编译的头文件。

## 9.4 Decoding the Error Novel 破译错误信息

(这节我完全看不懂, 以后再看)

普通编译错误通常足够简明和准确。比如，当编译器说“class X has no member ’fun’”，这就不难找出代码中的错误（比如我们可能把fun打错成了run）。对于模板来说并不是这样。

**简单类型不匹配**

考虑以下相对简单的使用C++标准库的例子：

```cpp
#include <string>
#include <map>
#include <algorithm>
int main()
{
     std::map<std::string,double> coll;
    //...
    // find the first nonempty string in coll:
    auto pos = std::find_if (coll.begin(), coll.end(),
          [] (std::string const& s){return s != "";}
          );
}
```

它包含一个相当小的错误：lambda表达式用于找到集合中第一个与字符串匹配的对象。然而map中的元素是key/value对，因此我们希望的是`std::pair<std::string const, double>`这样键值对。

GNU C++编译器会报告如下错误:

![img](/v2-efe1a845e6fa2affb12d9721104d90fb_720w-16582300054821.jpg)![img](/v2-a10c6558fbe3c8ded2ea25a9fe39f932_720w.jpg)

这样的信息看起来更像是小说而不是诊断。这些压倒性的信息会让模板的新手沮丧。然而，拥有一些经验后，这样的信息看以来也是可掌控的，错误相对容易定位。

错误信息的第一部分表示错误发生在一个函数模板实例中，深藏于predefined_ops.h头文件中，通过各种其他头文件被main.cpp文件包含。此处和后面的几行中，编译器报告了什么被哪些实参实例化了。本例中：

```cpp
auto pos = std::find_if (coll.begin(), coll.end(),
    [] (std::string const& s){return s != "";}
    );
```

这引起了std_algo.h头文件的115行的find_if模板的实例化，以下代码：

```cpp
_IIter std::find_if(_IIter, _IIter, _Predicate)
```

其中`_IIter`和`_Predicate`实例化为：

```cpp
_IIter = std::_Rb_tree_iterator<std::pair<const std::__cxx11::basic_string<char>, double> >
_Predicate = main()::<lambda(const string&)>
```

编译器报告所有的这些是为了防止我们不希望简单地将所有的模板进行实例化。这允许我们判定引起实例化的事件链。

然而，在我们的例子中，我们愿意相信所有种类的模板都需要被实例化，我们只是疑惑为什么这不行。最后一段的信息说明：没有匹配的调用，这意味着函数调用无法被解析，因为实参的类型和形参的类型不匹配。它列出了

```cpp
(main()::<lambda(const string&)>) (std::pair<const
std::__cxx11::basic_string<char>,
double>&)
```

和引起该调用的代码：

```cpp
{ return bool(_M_pred(*__it)); }
```

此外，包含“note: candidate:”的那一行解释了有一个期望const string&的单个候选类型，这个候选定义在errornovel1.cpp第11行，以lambda表达式` [] (std::string const& s)` 和为什么可能的候选不适用的原因的组合。

```
no known conversion for argument 1
from  'std::pair<const std::__cxx11::basic_string<char>, double>'
to 'const string& {aka const std::__cxx11::basic_string<char>&}'
```

这句话描述了我们的问题。

毫无疑问，错误信息可以更好一些。实际问题可以在实例化的历史之前发出，并且不是使用全展开的模板实例化，比如std::__cxx11::basic_string<char>，而是使用std::string就已经足够。然而，所有的诊断信息在某些情形下都将是有用的。这也不奇怪其他编译器提供相似的信息（尽管一些使用组织化技术）。

比如，Visual C++编译器输出类似于以下的信息：

![img](/v2-9494e028836dc97aa74c8cb6afd27c6a_720w.jpg)

这里，再次提供了实例化链的信息，提示什么被什么实参进行实例化，并且我们可以看到两次这个信息：

```cpp
cannot convert from 'std::pair<const _Kty,_Ty>' to 'const std::string'
with
[
    _Kty=std::string,
    _Ty=double
]
```

**Missing const on Some Compilers**

考虑以下代码：

```cpp
#include <string>
#include <unordered_set>
class Customer
{
private:
    std::string name;
public:
    Customer (std::string const& n): name(n) {} 
    std::string getName() const { return name; }
};

int main()
{
    // provide our own hash function:
    struct MyCustomerHash 
    {
        // NOTE: missing const is only an error with g++ and clang:
        std::size_t operator() (Customer const& c) 
        {
            return std::hash<std::string>()(c.getName());
        }
    };
    // and use it for a hash table of Customers:
    std::unordered_set<Customer,MyCustomerHash> coll; 
    //...
}
```

使用Visual Studio 2013或者 2015，代码将和期望的一样进行编译。然而，使用g++或者clang，这个代码会引起重要的错误信息。g++ 6.1中，比如，第一个错误信息如下所示：

![img](/v2-eaf7b5e66d19bc5abe766342e3be8eda_720w.jpg)

紧跟着二十余条的其他错误信息：

![img](/v2-e652cda47af9d6a257221efe98d4b25c_720w.jpg)

这还是很难读错误信息（甚至找到每条信息的头和尾都是个累活）。其本质是在头文件hashtable_policy.h中，实例化 std::unordered_set<>被以下语句需要：

```cpp
std::unordered_set<Customer,MyCustomerHash> coll;
```

没有该调用的匹配：

```cpp
const main()::MyCustomerHash (const Customer&)
```

在实例化以下代码时

```cpp
noexcept(declval<const _Hash&>()(declval<const _Key&>()))>
    ~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~
```

`(declval<const _Hash&>()`是类型 `main()::MyCustomerHash`的表达式。一个可能的相近的匹配候选是：

```cpp
std::size_t main()::MyCustomerHash::operator()(const Customer&)
```

它被声明为

```cpp
std::size_t operator() (const Customer& c) {
        ^~~~~~~~
```

最后一条信息说明了一些问题：

```
passing ’const main()::MyCustomerHash*’ as ’this’ argument
discards qualifiers
```

你可以看出问题所在吗？std::unordered_set类模板的实现需要hash对象的函数调用操作为const成员函数。当不满足时，错误在算法内部引发。当const修饰符加入后，从第一个开始的所有的其他错误信息都会消失。

```cpp
std::size_t operator() (const Customer& c) const { ... }
```

Clang 3.9给出稍微好点的提示，在第一条错误信息的结尾处：hash仿函体的operator()没有标记为const:

![img](/v2-68c384d6b7b998b05dcc0cebe6a764ef_720w.jpg)

注意clang这里提到默认模板参数，比如std::allocator<Customer>，但是gcc跳过了。

正如你所看到的，通常你不只有一个可用的编译器来测试代码。这会有助于写出更具有移植性的代码，但有的编译器会产生特别不可理解的错误信息，其他的可能提供更多的见解。

(**这是在说啥啊,看不懂捏**)

## 9.5 Afternotes 后记

将源文件组织在头文件和cpp文件中是“一处定义规则（one-definition rule, ODR）”的不同典型的实践结果。*扩展阅读见Section附录A*

包含模型是实用主义的答案，很大程度上受到现有C++编译器实现的支配。然而，第一个C++编译器实现是不同的：模板定义的包含是隐式的，它创建了一定程度的分离的幻想。第一个C++标准（C++98）提供了显式的模板编译的分离模型支持，通过导出的模板（exported templates）。分离模型允许模板在头文件中声明标记为export，然而他们对应的定义放在cpp文件中，就像非模板代码的声明和定义。不像包含模型，该模型是个不依赖于任何现有实现的理论模型，并且该实现本身表现出远比比C++标准委员会预料的复杂。五年之后才发布第一个实现（2002年5月），那年以后没有其他实现出现。为了让C++标准和现有的实现更一致，C++标准委员会在C++11开始移除了exported templates。

有时候，想象拓展预编译头文件的概念的方法有时是有诱惑力的，这样为单个编译不只有一个头文件可以载入。这原则上允许更细粒度的预编译方法。这里的障碍主要是预处理器：一个头文件中的宏可以完全改变后续头文件的含义。然而，一旦一个文件已经预编译，宏处理就已经完成了，因此为其他头文件引起的预处理器效果尝试给预编译好的头文件打补丁是不实际的。新的语言特性modules将被引入来解决这个问题。

## 9.6 Summary

* 模版的包含模型是组织模版代码最广泛使用的模型,备选方案将在14章讨论
* 只有在头文件中于类或结构外部定义函数模版的全特化才需要inline

* 利用预编译头文件(PCH), 确保#include指令保持相同的顺序
* 在引入模版后调试代码 is challenging