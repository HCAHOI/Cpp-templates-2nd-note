# 10 Basic Template Terminology 基本模版术语

到目前为止,我们已经介绍了C++模板的基本概念.在我们更深入之前,先复习我们使用的术语.这是需要的,因为C++社区里(甚至在更早版本的C++标准里）,有时候也缺乏准确的相应术语.

## 10.1 "Class Template" or "Template Class" ? 类模版还是模版类

在C++中,struct, class和union统称为class type.没有额外的限定时,class指用关键字class或者关键字struct引入的class type.注意,"class type"包含union,但是"class"不包含union.

对于"class that is a template"的称呼有一些混淆：

* 术语类模版(class template) 表明这个类是一个模版,也就是说,这是一系列类的参数化描述.
* 术语模板类(template class)被用于:
  * 类模板的别称
  * 指模板生成的类
  * 指名字是template-id的类（template-id指模板名称加上后面跟在<和>之间的模板实参）

第二个和第三个的含义有些许微妙,但对于后面的内容并不重要.

由于这种不准确,我们避免使用术语模版类(template class).

相似地,我们使用函数模板(function template),成员模板(member template),成员函数模板(member function template),变量模板(variable template),但是避开模板函数,模板成员,模板成员函数和模板变量之类的说法.

## 10.2 Substitution, Instantiation, and Specialization 替换、实例化和特化

当处理使用模板的源代码时,C++编译器必须在不同时间将模板中的模板形参替换为模板实参.有时候,这种替换是试验性的：编译器可能需要检查替换是否有效.

从一个模板中通过将形参替换为实参而实际创建普通类、类型别名、函数、成员函数或者变量的过程称为模板实例化.出人意料地,当前没有标准或者广泛一致的术语来专门表示创建声明（不是通过模板形参替换的进行的定义）的过程.我们看到过一些团队使用局部实例化（partial instantiation）或者声明实例化（instantiation of a declaration）这些习语,但是这些并不通用（对于类模板,生成了不完整的类）.

实例化或者不完整的实例化生成的实体（比如类、函数、成员函数或者变量）通常称为特化.然而,C++中实例化过程不是产生特化的唯一方法.另一种机制允许程序员显式指定模板形参特殊替换的声明.如我们在[Section 2.5](2-Class Template.md) 所述,这样的特化从template<>前缀开始：

```cpp
template<typename T1, typename T2> // primary class
template
class MyClass 
	//...
};

template<> // explicit specialization
class MyClass<std::string,float> {
	//...
};
```

严格来讲,这个叫显式特化（与实例化的或者生成的特化截然不同）.

如[Section 2.6](2-Class Template.md) 所述,特化后还有模板参数的称为偏特化:

当讨论特化(显式特化或者偏特化),通用的模板(general template)也称为主模板(primary template).

## 10.3 Declarations versus Definitions 声明和定义

标准C++中,声明和定义有相当准确的含义.C++的声明将一个名字引入或者重新引入C++ scope.这个引入总是包含名字的部分分类(classification),但是不需要细节来完成有效的声明.比如:

```cpp
class C; // a declaration of C as a class
void f(int p); // a declaration of f() as a function and p as a named parameter
extern int v; // a declaration of v as a variable
```

尽管有一个名字,但是宏定义和goto标签不被考虑为声明.

当声明的结构细节变得已知,或者变量的存储空间必须被分配时,声明变成了定义.对于类定义,这意味着必须提供大括号包含的主体.对于函数定义,这意味着大括号包含的主体必须提供（通常情况下）,或者函数必须有=default或者=delete进行指派.对于变量,初始化或者没有extern说明符就会引起声明变为定义.这里有个例子是对前面的非定义声明的补充：

```cpp
class C {}; // definition (and declaration) of class C
void f(int p) { //definition (and declaration) of function f()
    std::cout << p << '\n';
} 
extern int v = 1; // an initializer makes this a definition for v
int w; // global variable declarations not preceded by extern are also definitions
```

通过拓展,类模板或者函数模板的声明如果有主体,也成为定义,因此:

```cpp
template<typename T>
void func (T);
```

是个声明不是定义,但是

```cpp
template<typename T>
class S {};
```

是个定义.

## 10.3.1 Complete versus Incomplete Types 完整的和不完整的类型

类型可以是完整的,也可以是不完整的,这是个和声明与定义非常相近的概念.一些语言构造需要完整的类型,但是其他语言构造使用不完整的类型也可以进行.

不完整的类型包含：

- 已经声明但是没有定义的class类型
- 数组类型但是没有指定边界
- 元素为不完整类型的数组
- void
- 只有潜在类型或者枚举值没有定义的枚举类型
- 以上任何类型加上const或者volatile.

所有其他类型都是完整的类型,比如:

```cpp
class C; // C is an incomplete type
C const* cp; // cp is a pointer to an incomplete type
extern C elems[10]; // elems has an incomplete type
extern int arr[]; // arr has an incomplete type…
class C { }; // C now is a complete type (and therefore cp and elems
             // no longer refer to an incomplete type)
int arr[10]; // arr now has a complete type
```

## 10.4 The One-Definition Rule 一处定义规则

C++语言的定义给不同实体的重新声明增加了一些限制.这些限制统称为一处定义规则（one-definition rule, ODR）.这个规则的细节有点复杂,覆盖各种情况.后面的章节将描述其在不同方面的影响.此时,记住以下基本内容就足够:

- 普通(非模板)的非内联函数和成员函数,非内联全局变量,静态数据成员在整个程序中都应当只定义一次.
- class type(包括struct和union), 模板(包括偏特化但是不包括全特化), inline函数和inline变量在单个编译单元中最多定义一次,并且这些定义应该完全一样.

一个编译单元是源文件预处理后的结果,也就是说,它包含#include指令和宏拓展后的内容.

在本书的剩余部分,可链接实体(linkable entity)指以下任何内容 : 函数或者成员函数,全局变量或者静态数据成员,包括从模板中生成的这些实体,它们对链接器是可见的.

## 10.5 Template Arguments versus Template Parameters 模板实参与模板形参

比较以下类模板:

```cpp
template<typename T, int N>
class ArrayInClass {
public:
    T array[N];
};
```

和相似的普通类:

```cpp
class DoubleArrayInClass {
public:
    double array[10];
};
```

如果我们用double和10分别代替模板参数T和N,后者本质上等价于前者.在C++中,这种代替名称记为`ArrayInClass<double,10>`. 注意,模板的名字后面跟着的、在尖括号内的模板实参.不管这些实参本身是否依赖于模板参数,模板名称+尖括号内的实参这一组合称为**模板id（template-id）**.

该名称可以像对应的非模板实体一样使用.比如:

```cpp
int main()
{
    ArrayInClass<double,10> ad; 
    ad.array[0] = 1.0;
}
```

区分模板实参和模板形参非常重要.简短地说,形参由实参进行初始化(学术上,实参(arguments)通常指实际的参数,形参(parameters)通常称为形式上的参数).更准确地说:

- 模板形参是模板声明或定义时在关键字template后面的列表(这个例子里是T和N).
- 模板实参是替换模板形参的条目(这个例子里是double和10).不像模板形参,模板实参不仅仅是名字.

当使用template-id时,模板形参由模板实参进行显式替换,但是有许多情形替换是隐式的(比如,如果模板形参由默认实参替换).一条基本原则是：任何形参必须是可以在编译期确定的数量或者值.这个要求为run-time模板实体的运行带来巨大的好处.因为模板形参最终由编译时的值进行替换,他们本身可以用于构成编译期表达式.这可以在ArrayInClass的模板中被用于确定成员数组的长度.任何数组的size必须是常量表达式,模板参数N满足这个条件.

我们可以把这个推论推得更远一些：因为模板形参是编译时实体,他们可以用于创建有效的模板实参.以下例子:

```cpp
template<typename T>
class Dozen {
public:
    ArrayInClass<T,12> contents;
};
```

注意,这个例子中,名字T同时是模板形参和模板实参.因此,这样的机制可以用于从简单的模板构建更复杂模板.当然,这与允许我们集合类型与函数的机制没有本质区别.
