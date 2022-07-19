# 2 Class Tempalte 类模版

## 2.1 Implement of Class Template *Stack* Stack类模版的实现 

一个自制Stack类

```cpp
#include <cassert>
#include <vector>

template<typename T>
class Stack {
private:
    std::vector<T> elems;

public:
    void push(T const &elem);
    void pop();
    T const& top() const;
    bool empty() const {
        return elems.empty();
    }
};

template<typename T>
void Stack<T>::push(T const &elem) {
    elems.push_back(elem);
};

template<typename T>
void Stack<T>::pop() {
    assert(!empty());
    elems.pop_back();
}

template<typename T>
T const& Stack<T>::top() const {
    assert(!empty());
    return elems.back();
}
```

### 2.1.1 Declaration of Class Templates 类模版的声明

在类模版内使用不带模版参数的类名表示表示以模版参数为实参(暂时没看懂.但是似乎表达的东西很简单) *细节见13.2.3节*

带<T>表示特殊模版参数时的特殊处理,所以一般还是不带

但是如果在类模版外实现类成员函数就需要加上<T>,例如:

```cpp
template<typename T>
class Stack{
    //...
   Stack (Stack const &);
    //...
}

template<typename T>
bool operator==(Stack<T> const &lhs, Stack<T> const &rhs);
```

注意,运算符重载最好定义为非成员函数,见《Effective C++》

### 2.1.2 Implementation of Member Functions 成员函数的实现

告诉我们在类模版外实现成员函数要在函数名前加*类名::*

```cpp
template<typename T>
void Stack<T>::pop() {
	assert(!empty());
    elems.pop_back();
}
```

这里的assert()是为了避免在vector为空时仍然弹出元素导致undifined-behaviour

## 2.2 Use of Class Template *Stack* Stack类模版的使用

在C++17标准之前,Stack对象的声明必须显式指定模版参数

```cpp
Stack<int> s;	//指这个<int>
```

小知识: 在C++11之前`Stack<Stack<int>>`是无法通过编译的,编译器会把>>理解为运算符,所以必须写为`Stack<Stack<int> >`

### 2.3 Partial Usage of Class Templates 类模版的局部使用

为了降低程序的大小和复杂度,类模版中没有使用的成员函数将不会实例化

### 2.3.1 Concepts 概念

我们如何知道模版需要哪些操作才能实例化?Concepts作为一种术语用来表示模版库中需要反复使用的约束.例如,C++标准库依赖于随机访问迭代器与缺省构造函数等概念.

截至C++17,concept仍然只能在文档中表述(C++20有一个叫做concept的东西,回头查查cppconference捏).如果不遵循这些约束可能导致严重的问题.

从C++11开始,至少可以使用static_assert()和一些预定义的type traits来检查基本的约束.比如

```cpp
template<typename T>
class C{
    static_assert(std::is_default_constructible_v<T>, "(Error Msg)");
	//...
};
```

如果不可缺省构造将会编译失败, 但是错误信息可能包含整个模版实例化的原因到检测到错误的实际模版定义 *见Section 9.4*

*更复杂的代码检查见19.6.3节*

*有关concepts的更多信息见附录E*

## 2.4 Friends 友元

就是类模版的友元,原理大致相同

## 2.5 Specializations of Class Templates 类模版特化

```cpp
template<>
class Stack<std::string> {
private:
    std::deque<std::string> elems;	//注意这里改成std::deque了哦

public:
    void push(std::string const &elem);
    void pop();
    std::string const& top() const;
    bool empty() const {
        return elems.empty();
    }
};

void Stack<std::string>::push(std::string const &elem) {
    elems.push_back(elem);
};
//同理实现其他特化成员函数
```

也可以只单独特化一个成员函数,但是这个特化成员函数所在的实例将无法再特化.

小知识:这里的push实现并不好,现在我觉得应该这样实现

```cpp
void Stack<std::string>::push(auto&& elem) {
    elems.push_back(std::forward<std::string>(elem));
};
```

*本书提供的更好版本见Section9.1*

## 2.6 Partial Specialization 偏特化

比如我们可以针对指针进行偏特化

```cpp
template<typename T>
class Stack<T*> {
private:
    std::vector<T*> elems;

public:
    void push(T* const &elem);
    void pop();
    T* const& top() const;
    bool empty() const {
        return elems.empty();
    }
};

template<typename T>
void Stack<T*>::push(T* const &elem) {
    elems.push_back(elem);
};
//同理进行成员函数的实现
```

**Partial Specialization with Multiple Parameters**

```cpp
template<typename T>
class Myclass<T, T> {
	//...
};

template<typename T>
class Myclass<T*, T*> {
	//...
};bie ming

template<typename T>
class Myclass<T, int> {
	//...
};

int main() {
    Myclass<int, float> mif;	//1
    Myclass<float, float> mif;	//1
    Myclass<float, int> mif;	//3
    Myclass<int*, float*> mif;	//2
    
    Myclass<int, int> mif;		//Error! 同时匹配1和3
    Myclass<int*, int*> mif;	//Error! 同时匹配1和2
	//...
}
```

*关于偏特化的更多细节见Section16.4*

## 2.7 Default Class Template Arguments 缺省类模版参数

比如,我们可以指定缺省时Stack类模版的container类型为std::vector

```cpp
template<typename T, typename Cont = std::vector<T>>
class Stack {
private:
    Cont elems;

public:
    void push(T const &elem);
    void pop();
    T const& top() const;
    bool empty() const {
        return elems.empty();
    }
};

template<typename T, typename Cont = std::vector<T>>
void Stack<T, Cont>::push(T const &elem) {
    elems.push_back(elem);
};

template<typename T, typename Cont = std::vector<T>>
void Stack<T, Cont>::pop() {
    assert(!empty());
    elems.pop_back();
}

template<typename T, typename Cont = std::vector<T>>
T const& Stack<T, Cont>::top() const {
    assert(!empty());
    return elems.back();
}
```

注意,指定的container类型必须实现成员函数中使用的所有方法和运算符

用例

```cpp
Stack<int> intStack;
Stack<double, std::deque<double>> doubleStack;
```

## 2.8 Type Aliases 类型别名

**Typedefs and Alias Declarations**

有typedef(typedef-name)与using(alias declaration)两种方式,无论是从理解上还是实用上,我们都使用后一种.using不仅支持模版化,而且在type traits中有大用.

```cpp
typedef Stack<Int> intStack;
using intStack = Stack<int>;
```

**Alias Templates**

(正文没什么东西, 告诉我们类模版的某种实例可以指定别名)

注意, 模版只能在global/namespace scope或者class declarations中声明和定义

**Alias Template for Member Types**

(没什么内容,  告诉我们类模版中的类型比如iterator可以指定别名)

## 2.9 Class Template Argument Deduction 类模版参数推导

在C++17之后,一定条件下,模版参数可以由编译器推导

```cpp
Stack<int> s1;
Stack s2 = s1;	//OK since C++17
```

通过提供初始化参数,编译器可以推导出模版参数,比如我们为Stack类添加新构造函数

```cpp
template<typename T>
class Stack {
private:
    std::vector<T> elems;

public:
    Stack() = default;	//提供default constructor
    Stack(T const &elem)
   	: elems({elem}) {}; 
    //...
};

Stack s1 = 0;	//模版参数T推导为int
```

注意`elems({elem})`部分中的大括号,这是为了将elem转换为std::initialized_list,否则elems将初始化为元素数为0的vector.也就是说

```cpp
std::vector<int> v1{10, 20};	//前两个元素为10,20
std::vector<int> v2(10, 20);	//10个元素,全是20
```

原则上()和{}应该起相同的作用,除了{}内的参数不支持隐式转换以外,但是如果类中含有参数为std::initialized_list的构造函数,用大括号初始化时几乎一定会使用这个构造函数,所以我们说std::vector的接口设计是失败的.

注意,与函数模版不同,类模版不能显式指定一些参数让编译器推导其他的.*细节见Section15.12*

**Class Template Arguments Deduction with Stirng Literals**

对于

```cpp
Stack s1 = "bottom";	//s1被推导为Stack<char const[7]>
```

当按引用传参时,参数不会发生decay,这将导致我们不能往里添加其他string

当按值传参时,参数会发生decay,使得推导的类别变为char const*,这正是我们想要的,所以我们应该使用

```cpp
template<typename T>
class Stack {
private:
    std::vector<T> elems;

public:
    Stack() = default;	//提供default constructor
    Stack(T elem)
   	: elems({elem}) {}; 
    //...
};
```

为了避免非必要的拷贝也可以使用

```cpp
template<typename T>
class Stack {
private:
    std::vector<T> elems;

public:
    Stack() = default;	//提供default constructor
    Stack(T elem)
   	: elems({std::move(elem)}) {}; 
    //...
};
```

**Deduction Guides**

C++17之后,我们可以在global/namespace scope使用这种技术对推导进行控制

```cpp
template<typename T>
class Stack {
private:
    std::vector<T> elems;

public:
    Stack() = default;	//提供default constructor
    Stack(T const &elem)
   	: elems({elem}) {}; 
    //...
};
Stack(char const *) -> Stack<std::string>
```

在我们的指导下,传入的字符串字面值或C字符串将会转换为std::string进行实例化

*往Section15.12看更多有关类模版参数推导的细节*

## 2.10 Aggregate Class 聚合类

没有用户提供的,显式的或继承的构造函数,没有private或protected的非静态成员,没有虚函数,没有private或protected的基类的class或struct被称为Aggregate Class.它也可以是类模版.例如

```cpp
template<typename T>
class Test{
    T value;
    std::string comment;
};

Test<int> t;
t.value = 12;
```

我们也可以对Aggregate Class用Deduction Guides

```cpp
Test(char const*, char const*) -> Test<std::string>
Test t2 = {"hello", "world"};
```

如果没有Deduction Guides,这种操作是不可能的,因为Test没有支持这种操作的构造函数.

std::array也是参数化了元素类型和大小的Aggregate Class,C++17为其提供了Deduction Guides.*见Section4.4.4*

## 2.10 Summary

* 类模版是实现时有一个或多个类型参数的类
* 想要使用类模版,需要传递模版参数,类模版之后会为这些类型实例化
* 对于类模版,只有被调用的成员函数会被实例化
* 可以为特定类型特化类模版
* 可以为特定类型偏特化类模版
* 从C++17开始,类模版参数可以被编译器自动推导
* 可以定义聚合类模版
* 如果按值传参,参数会发生decay
* 模版只能在global/namespace scope或类定义内声明和定义