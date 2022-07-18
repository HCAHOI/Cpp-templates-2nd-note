# 5 Tricky Basics

## 5.1  Keyword *typename* 关键字*typename*

当一种类型存在依赖时，必须在前面加typename, 比如subType与stl容器内迭代器

*更多细节见Section13.3.2*

*无论如何,先考虑用using*

*在C++20之后,许多情况下将不再需要typename, 见Section17.1*

## 5.2 Zero Initialization 零初始化

我们都知道,当声明一个变量的时候最好对其进行初始化

但是如果使用了模版的话,等号右侧应该是什么捏

```cpp
template<typename T>
void foo() {
	T x{};	//要么使用零构造,要么使用提供的构造函数
}
```

(这节后面的部分讨论了C++17以来{}与()初始化的区别)

在C++17前,`T x();`只有在复制构造函数未标为explicit时才可以成立

注意

```cpp
template<typename T>
void foo(T x{});		//错误的
void foo(T x = T{});	//正确的,但是在C++11之前必须是T()
```

## 5.3 Using *this->* 使用*this->*

在派生类中`x`并不总等于`this->x`,例如

```cpp
template<typename T>
class Base {
public:
    void bar();
};

template<typename T>
class Derived : public Base {
public:
    void foo() {
        bar();	//调用外部的bar()或者报错
    }
};
```

基于经验,对于从基类继承来的所有符号(symbol),我们都应该限定,对于上面的例子,我们可以使用`this->bar()`或`Base::bar()`

*细节见Section13.4.2*

## 5.4 Templates for Raw Array and String Literials 原始数组和字符串字面量模版

注意在将原始数组和字符串字面量传入模版参数时,如果将形参标为引用,实参将不会发生decay,只有按值传递,字符串才会乖乖decay为char const*而不是什么char const[6]之类的

我们也可以进行偏特化,如下

```cpp
template<typename T>
struct Myclass;	//原始结构体

template<typename T, std::size_t SZ>
struct Myclass<T[SZ]>;	//已知大小的数组

template<typename T, std::size_t SZ>
struct Myclass<T(&)[SZ]>;	//已知大小数组的引用

template<typename T>
struct Myclass<T[]>;	//未知大小的数组

template<typename T>
struct Myclass<T(&)[]>;	//未知大小数组的引用

template<typename T>
struct Myclass<T*>;	//指针

template<typename T1, typename T2, typename T3>\
void foo(int a1[7], int a2[], int (&a3)[42], int (&x0)[], T1 x1, T2 &x2, T3&& x3) {
    Myclass<decltype(a1)> m3;    //Myclass<T*>
    Myclass<decltype(a2)> m4;    //Myclass<T*>
    Myclass<decltype(a3)> m5;    //Myclass<T(&)[SZ]> 	未decay
    Myclass<decltype(x0)> m6;    //Myclass<T(&)[]>		未decay
    Myclass<decltype(x1)> m7;    //Myclass<T*>
    Myclass<decltype(x2)> m8;    //Myclass<T(&)[]>		未decay
    Myclass<decltype(x3)> m9;    //Myclass<T(&)[]>  	涉及到完美转发，这里传入的是左值
}

int main() {

    int a[42];
    Myclass<decltype(a)> m1;    //Myclass<T[SZ]>

    extern int x[];
    Myclass<decltype(x)> m2;    //Myclass<T[]>
    return 0;

    foo(a,a,a,x,x,x,x);
}
```

注意声明为数组的形参实际上具有指针类型

注意未知边界的数组模版可以用于不完整类型

*另一个在泛型中使用不同数组模版的例子见Section19.3.1*

## 5.5 Member Templates 成员模版

类成员函数类可以成为模版,以Stack类为例

```cpp
template<typename T>
class Stack {
private:
    std::deque<T> elems;

public:
    void push(T const &elem);
    void pop();
    T const& top() const;
    bool empty() const {
        return elems.empty();
    }
};
```

默认情况下,只有模版类型相同的Stack才可以互相赋值,如果想让不同类型的Stack互相赋值就可以使用成员模版

```cpp
template<typename T>
class Stack {
private:
    std::deque<T> elems;

public:
    void push(T const &elem);
    void pop();
    T const& top() const;
    bool empty() const {
        return elems.empty();
    }
    
    template<typename T2>
    Stack& operator=(Stack<T2> const&);
};

//implementation
template<typename T>
template<typename T2>	//模版内的模版
Stack<T>& Stack<T>::operator= (Stack<T2> const& op2) {
    Stack<T2> tmp = op2;
    
    elems.clear();
    while(!tmp.empty()) {
        elems.push_front(tmp.top());
        tmp.pop();
    }
    
    return *this;
}
```

这里我们只能访问op2的公共接口,如果想访问op2的其他元素,我们可以声明Stack的其他实例都是友元

```cpp
template<typename T>
class Stack {
private:
    std::deque<T> elems;

public:
    void push(T const &elem);
    void pop();
    T const& top() const;
    bool empty() const {
        return elems.empty();
    }
    
    template<typename T2>
    Stack& operator=(Stack<T2> const&);
	template<typename> friend class Stack;	//因为类型名没有被使用,所以我们省略了它
};

//new implementation
template<typename T>
template<typename T2>
Stack<T>& Stack<T>::operator= (Stack<T2> const& op2) {
    using std::begin;
    using std::cbegin;
    using std::cend;
    
    elems.clear();
    elems.insert(begin(elems), cbegin(op2.elems), cend(op2.elems));
    
    return *this;
}
```

但是在执行过程中还会进行必要的类型检查,如果把string塞进`Stack<int>`,肯定是会报错的

我们可以将容器类型同样设置为模版参数

```cpp
template<typename T, typename Cont = std::deque<T>>
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
    
    template<typename T2, typename Cont2>
    Stack& operator=(Stack<T2, Cont2> const&);
	template<typename, typename> friend class Stack;	//因为类型名没有被使用,所以我们省略了它
};

//略过implementation
```

**Specialization of Member Fucntion Templates**

模版成员函数也可以特化或偏特化

注意你不需要也不能声明特化成员函数,你只需要重新实现.如果定义在另一个翻译单元(translation unit)的话,需要在函数前加*inline*

```cpp
class BoolString {
private:
    std::string value;
public:
    explicit BoolString(const std::string &value) : value(value) {};
    template<typename T = std::string>
    T get() const {
        return value;
    }        
};

template<>
inline bool BoolString::get<bool>() const {
    return value == "true" || value == "yes" || value == "on";
}
```

**Special Member Function Templates**

只要特殊成员函数可以复制或者移动对象, 就可以使用模版成员函数,但是需要注意,模版特殊成员函数的存在并不能阻止默认函数的生成

good:这样生成的模版特殊成员函数可能比预定义的复制/移动构造函数更加匹配 *详细信息见Section6.2*

bad:模版化复制/移动构造函数并不容易,比如在约束其生成的时候 *详细信息见Section6.4*

### 5.5.1 The *.template* Construct *.template*构造

有的时候需要显式限定模版参数,这时就需要*template*关键字确保符号<是模版参数列表的开头

```cpp
template<unsigned long N>
void printBitset (std::bitset<N> const &bs) {
    std::cout << bs.template to_string<char, std::char_traits<char>, std::allocator<char>>();
}
```

对于bs,我们在调用其成员函数to_string()同时指定字符串的详细信息,如果没有额外使用`.template`,编译器就不知道后面的<不是小于号

只有在句点之前的构建依赖于模版参数才会出现这个问题,在这里,bs依赖于N

`.template`以及类似记号`->template`,`::template`只能在模版内且当它们跟着依赖于模版参数的对象时使用 *更多信息见Section13.3.3*

### 5.5.2 Generic Lambdas and Member Templates 泛型lambda表达式和成员模版

泛型lambda表达式是C++14以来引进的缩小版成员模版

*详见Section15.10.6*

## 5.6 Variable Templates 变量模版

C++14以来,变量也可以通过特定参数模版化.注意区分variable template(变量模版)和variadic template(可变参数模版)

```cpp
template<typename T>
constexpr T pi{3.1415926};
```

这个声明也不能存在于function 和 block scope

想要使用变量模版,我们必须指定类型

```cpp
std::cout << pi<double> << '\n';
std::cout << pi<int> << '\n';	\\Oops, you change the world.
```

变量模版可以跨翻译单元(translation unit)使用

虽然变量模版可以指定缺省类型,但是必须以`pi<>`的形式使用`pi`是不合法的

变量模版也可以使用非类型模版参数

```cpp
template<auto N>
constexpr decltype(N) val = N;
```

**Variable Templates for Data Members**

变量模版的一种应用是类模版中的成员,对于

```cpp
template<typename T>
class Myclass {
public:
	static constexpr int max = 1000;
}
```

然后我们就可以通过对Myclass进行特化将max设置为不同的值

```cpp
template<typename T>
int myMax = Myclass<T>::max;
```

于是我们可以

```cpp
auto i = myMax<std::stirng>;
```

而不必

```
auto i = Myclass<std::string>::max;
```

**Type Traits Suffix _v**

C++17的_v后缀的定义方法

```cpp
namespace std {
    template<typename T> constexpr bool is_const_v = is_const<T>::value;
}
```

## 5.7 Template Template Parameters 模版模版参数

模版模版参数是很有用的,以Stack为例,如果想要使用不同的容器,必须两次指定类型

```cpp
Stack<int, std::vector<int>> s
```

如上,int被传入两次,使用模版模版参数可以解决,像这样

```cpp
Stack<int, std::vector> s
```

为了做到这件事,我们必须将第二个模版参数指定为模版模版参数,如下

```cpp
template<typename T, template<typename Elem> class Cont = std::deque>	//在C++17之前必须使用class,C++17可以仍然选用typename
class Stack {															//由于Elem没有被使用,我们可以将其省略
private:
    Cont<T> elems;	//可以使用任意类型实例化第二个模版参数,而非必须是第一个

public:
    void push(T const &elem);
    void pop();
    T const& top() const;
    bool empty() const {
        return elems.empty();
    }
};
```

当然,成员函数也需要进行修改,比如push()

```cpp
template<typename T, template<typename Elem> class Cont = std::deque>
void Stack<T, Cont>::push //...
```

注意,函数模版和变量模版没有这种用法.

**Template Template Argument Matching**

在C++17之前,这个Stack类可能不能很好的运行,编译器告诉你std::deque的默认值与Cont不兼容

这是因为模版模版参数必须与其所替代的类型的模版参数完全匹配,而std::deque有两个模版参数,所以报错.一些例外与可变参数模版有关 *见Section12.3.4*

我们可以重写类声明,使得Cont期望具有两个模版参数的容器

```cpp
template<typename T, template<typename Elem, typename Alloc = std::allocator<Elem>> class Cont = std::deque>	
class Stack {
private:
    Cont<T> elems;	//可以使用任意类型实例化第二个模版参数,而非必须是第一个
	//...
};
```

并不是所有容器都可以用于Cont,比如std:;array不行,因为它包含了一个表示大小的非类型模版参数,这与我们的声明不符

*更多关于模版模版参数的讨论和例子见Section12.2.3, Section12.3.4, Section19.2.2*

## 5.8 Summary

* 要使用依赖于模版参数的类型名称,必须在前面加typename(或者using)
* 想要访问依赖于模版参数的基类成员,使用`this->`或`Base::`
* 嵌套的类和成员函数也可以是模版.可以应用于实现内部赋值的泛型操作
* 模版版本的构造函数和赋值操作符不能阻止缺省函数的生成
* 通过使用{}或缺省构造函数进行初始化,可以保证使用默认值初识模版的变量和成员,即使它们是内建类型
* 可以为原始数组和字符串字面量提供特化的模版
* 按引用传入原始数组和字符串字面量时,它们不会进行decay(即从数组转换为指针)
* 从C++14开始,我们可以定义变量模版
* 可以使用类模版作为模版参数,即模版模版参数
* 模版模版参数通常要与它们参数精确匹配