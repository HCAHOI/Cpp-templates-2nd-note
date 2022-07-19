# 7 By Value or by Reference? 按值传递还是按引用传递

C++17之后,即使没有可用的复制或移动构造函数,也可以通过按值传递临时实体(右值/rvalues)*参见Section B.2.1*.因此C++17的附加约束是不能复制左值

在此之前,熟悉值分类的术语是很必要的,比如:左值(lvalue),右值(rvalue),将亡值(xvalue),纯右值(pvalue),泛左值(gvalue)*这些在附录B有解释*

## Passing by Value 按值传递

按值传递时对于类通常使用复制构造函数传递副本.很多人会觉得代价很高,但是事实上有各种方法避免昂贵的复制,编译器可能会使用包括移动语义在内的各种优化

对于

```cpp
template<typename T>
void foo(T x) {
}

std::string returnString();
std::string s = "hi";

foo(s);					//copy(lvalue)
foo(std::string("hi"));	//被优化(pvalue)
foo(returnString());	//被优化(pvalue)
foo(std::move(s));		//移动构造函数(xvalue)
```

**Passing by Value Decays**

术语decay来自于C语言,也适用于从函数到函数指针的转换*见Section 11.1.1*

像用auto传递一样,按值传递时参数会发生decay,原始数组变为指针,const,volatile等限定符被删除

```cpp
template<typename T>
void print(T arg) {
}

std::string const c = "hi";
print(c);	//std::string
print("hi");	//char const*
int arr[4];
print(arr);	//int*
```

这种特性一方面简化了操作,另一方面使得我们无法分辨传入的是裸指针还是数组*在Section 7.4 讨论如何处理字符串字面量和其他原始数组*

## 7.2 Passing by Reference 按引用传递

### 7.2.1 Passing by Constant Reference 按常引用传递

```cpp
template<typename T>
void foo(T const &x) {
}

std::string returnString();
std::string s = "hi";

foo(s);					//no copy
foo(std::string("hi"));	//no copy
foo(returnString());	//no copy
foo(std::move(s));		//no copy
```

如果按非常量引用传递,就意味着函数可以修改这个参数在内存内可到达的所有值,那么编译器就必须假设其可能缓存的所有值(通常在寄存器中)都将无效,而如果参数很大,将其重新加载是很昂贵的

**Passing by Reference Does Not Decay**

注意

```cpp
template<typename T>
void foo(T const& x) {
}
```

由于函数参数已经有const和引用符号,推导出的T将不会带有const和引用符号

```cpp
foo("hi");	//x为char const(&) [2], T为char[2]
```

### 7.2.2 Passing by Nonconstant Reference 按非常引用传递

如果想要通过参数返回值,那么就必须按非常引用传递

```cpp
template<typename T>
void foo(T &out) {
}

std::string returnString();
std::string s = "hi";

foo(s);					//OK out是string&, T是string
foo(std::string("hi"));	//Error! 不允许传递pvalue
foo(returnString());	//Error! 不允许传递pvalue
foo(std::move(s));		//Error! 不允许传递xvalue
```

问题在于,对于模版来说,如果传递一个const实参,可能将out推导为const&,这使得我们可以将右值传入本需要左值的函数

```cpp
template<typename T>
void foo(T &out) {
}

std::string returnString();
std::string const s = "hi";

foo(s);					//OK T是cosnt string
foo("hi");				//OK T是char const[3]
foo(returnString());	//OK 如果函数返回的是const string
foo(std::move(s));		//OK T是string const(这里发生了引用折叠)
```

虽然可以传入,但是任何修改传入参数的尝试都将报错,这可能会发生模版函数的深处*见Section 9.4*

如果想阻止这种事情的发生,使用static_assrt()和`std::is_const_v<T>`

或者std::enable_if<>

或者concepts

### 7.2.3 Passing by Forwarding Reference 按转发引用传递

使用转发引用的一个原因是想要完美转发参数,但是需要理解类型推导规则

```cpp
template<typename T>
void foo(T&& x) {
}

std::string returnString();
std::string s = "hi";
int arr[4];

foo(s);					//OK T为std::string&
foo(std::string("hi"));	//OK T为std::string&&
foo(returnString());	//OK T为std::string&&
foo(std::move(s));		//OK T为std::string&&
foo(arr)				//OK T为int(&)[4]
```

而

```cpp
std::string const s = "hi";
foo(s);					//OK T为std::string const&
foo("hi");				//OK T为char const(&)[2]
```

这说明转发引用可以区分左右值和左值是否为const

但是有一个问题

```cpp
template<typename T>
void foo(T&& x) {
	T arg;	//Error arg为引用,需要初始化
}
```

*前往Section 15.6.2看如何解决这个问题*

## 7.3 Using *std::ref()* and *std::cref()* 使用*std::ref()* 和 *std::cref()*

C++11之后,可以使用`<fucntional>`头文件下的std::ref()与std::cref()让调用者来决定来按引用传递

```cpp
template<typename T>
void foo(T x) {
}

std::string s = "hello";
foo(s);				//按值传递
foo(std::cref(s));	//pass s as if by reference
```

std::cref()并不会改变模版参数的处理(handel),实际上它创造了一个std::reference_wrapper<>类型的对象并将参数按值传入其中

std::reference_wrapper<>只支持一种操作:隐式转换回原类型并返回,生成原始对象(也可以用get()).因此只要有被传递对象的有效操作符就可以使用reference warpping

```cpp
void foo2(std::string x) {
}

template<typename T>
void foo1(T x) {
	foo2(x);	//可能将x转换回std::string
}

std::string s = "hello";
foo(std::cref(s));	//pass s as if by reference
```

注意,编译器必须知道隐式转换会原始类型是必要的,因此只有通过泛型函数将参数传递给非泛型参数时std::ref()才可以运行,将T直接输出将会失败,因为此时的类型仍然为std::reference_wrapper<>而这个类型没有针对<<操作符的重载

```cpp
template<typename T>
void foo1(T x) {
    std::cout << x;	//Error
}

std::string s = "hello";
foo(std::cref(s));	//pass s as if by reference
```

同理,以下代码也会失败

```cpp
template<typename T1, typename T2>
bool isLess(T1 a, T2 b) {
    return a < b;
}

std::string s = "hi";
if(isLess(std::cref(s), std::string("world"))); 	//Error!
```

## 7.4 Dealing with String Literals and Raw Arrays 字符串字面量和原始数组的处理

我们已经见到了字符串字面量和原始数组在传入函数的不同结果

* 按值传递,它们将会变成指向元素类型的指针
* 任何形式的引用传递都不会发生decay, 因为我们得到的是原数组的引用

两者都有好有坏,第一种的优劣之前已经提过,第二种的问题在于不同大小的数组将被视为两个类型

```cpp
template<typename T>
bool cmp(T const &arg1, T const &arg2) {
    //....
}

cmp("hi", "world");	//Error
```

如果使用按值传递,这个函数的的调用是可能的

```cpp
template<typename T>
bool cmp(T const arg1, T const arg2) {
    //....
}

cmp("hi", "world");	//OK
```

但是这并不意味着问题没有了,更严重的是,这样可能会把compile-time error变成run-time error

```cpp
template<typename T>
bool cmp(T const arg1, T const arg2) {
    if(arg1 == arg2)	//Oops, 我们似乎在比较两个数组的指针...
    //....
}

cmp("hi", "world");	//OK
```

尽管如此,很多情况下decay是有用的,特别是检查两个对象(都是参数,或者其中一个希望和参数比较)是否具有相同的类型

但是如果你想要使用完美转发,你必须把参数声明为转发引用,这时可以使用std::decay<>()显式decay参数*例子参见Section 7.6中的std::make_pair()*

注意其他type traits有时也会隐式decay,例如std::common_type<>,它产生两个传递的参数的公共类型*详见Section D.5*

### 7.4.1 Special Implementations for String Literials and Raw Arrays 字符串字面量常量和原始数组的特殊实现 

你可能需要根据传入的是数组还是指针来区分实现,当人,这需要参数没有发生decay

有两种方式解决这个问题

1. 声明只对数组有效的模版参数

```cpp
template<typename T, std::size_t N1, std::size_t N2>
void foo(T (&args1)[N1], T (&args2)[N2]) {
    T* pa = arg1;	//decay
    T* pa = arg2;	//decay
    if(compareArrays(pa, N1, pb, N2)) {
    	//...
    }
}
```

注意,您可能需要多种实现来支持不同的原始数组*见Section 5.4*

2. 使用无敌的type traits

```cpp
template<typename T, typename = std::enable_if_t<std::is_array_v<T>>>
void foo(T&& arg1, T&& arg2) {
    //...
}
```

## 7.5 Dealing with Return Values 返回值的处理

对于返回值,我们可以选择选择按值返回或按引用返回,但按引用返回可能是麻烦的来源, 因为我们引用的东西可能超出了我们的控制范围

一些情况下,返回引用是正常的

* 返回容器或字符串元素(operator[]或front())
* 授予类成员写访问的能力
* 返回链式调用的对象(操作符<<,>>,=)

通常返回常引用来授予读访问权

返回引用很可能会罩成空悬问题

```cpp
std::string* s = new std::String("hi");
auto& c = (*s)[0];
delete s;
std::cout << c;	//Error
```

这个例子看上去很简单,但是其他情况就不这么容易发现了

```cpp
auto s = std::make_shared<std::string>("hi");
auto& c = (*s)[0];
s.reset();
std::cout << c;	//Error
```

因此我们应该保证按值返回,但是使用模版参数T并不能保证不是引用

```cpp
template<typename T>
T retR(T&& p) {
    return T{...};	//当传入左值时,T将成为引用
}
```

即使p是按值传递的,我们也可以显式指定T为引用

为了解决这个问题,我们有两种选择

1. 使用std:;remove_reference<>或std::decay<>

2. 令返回值类型为auto,因为auto总是会去除引用

## 7.6 Recommended Template Parameter Declarations 模版参数声明推荐

根据之前的内容进行总结

* 声明为**按值传递**

  简单的传递方式,它可以是传入的参数发生decay,但是对于大型对象不能保证高效.调用者仍然可以使用std::ref()和std::cref()来提供引用,但是要保证这样是有效的

* 声明为**按引用传递**

  这中传递方式一般有很好的效率,特别是在传递

  * 现有对象(lvalue)到左值引用
  * 临时对象(prvalue)或者可移动的对象(xvalue)到右值引用
  * 转发引用

  注意可能的decay,以及转发引用可能将类型推导为引用

**General Recommenditions**

1. 一般情况使用按值传递,对于字符串字面量,一些小参数,临时或可移动对象都能很好地工作

   如果传递的是很大的对象,可以使用std::ref()等避免复制

2. 如果理由充分,也可以

   * 如果想要一个out参数能作为返回,考虑非常量引用,同时可以像*Section 7.2.2*讨论的那样禁止const对象进入
   * 如果提供了一个模版来转发参数,使用完美转发,并在适当的时候std::decay或std::common_type<>来协调不同的字符串字面量和原始数组
   * 如果性能很重要,而且复制很昂贵,考虑常引用

3. 如果你有更好的想法,就使用它

**Don't Be Over-Generic**

在实践中,函数模版通常不会接受任意类型的参数.例如我们想要一种存储某种类型的向量,那么在声明的时候最好不要过于一般化

```cpp
template<typename T>
void foo(std::vector<T> const &v) {

}
```

这么做我们可以保证T不会成为引用,因为vector无法接受引用值.如果我们直接用T,那么按值调用和按引用调用的区别就不那么明显了

**std::make_pair() Example**

std::make_pair<>()展示了决定参数传递机制的缺陷,使用它生成pair<>是很方便的,其声明在不同的标准中有所不同

C++98中,其使用了按引用传递来避免复制

```cpp
template<typename T1, typename T2> 
pair<T1, T2> make_pair(T1 const &a, T2 const &b) {
	return pair<T1, T2>(a, b);
}
```

这在传入字符串字面量的时候会导致严重的问题,于是C++03改为按值传递

```cpp
template<typename T1, typename T2> 
pair<T1, T2> make_pair(T1 a, T2 b) {
	return pair<T1, T2>(a, b);
}
```

在正确性面前,效率并不重要

然而在C++11之后,其必须支持移动语义,因此改为了转发引用

```cpp
template<typename T1, typename T2> 
constexpr pair<typename decay<T1>::type, typename decay<T2>::type>
make_pair(T1&& a, T2&& b) {
	return pair<typename decay<T1>::type, 
    			typename decay<T2>::type>(std::forward<T1>(a), forward<T>(b));
}
```

实际上为了支持std::ref(),该函数还将std::reference_wrapper展开变为完整的引用

## 7.7 Summary

* 在测试模版时,使用不同长度的字符串字面量
* 按值传递的参数会decay,按引用传递的不会
* type traits std::decay<>允许我们显式的decay参数
* 某些情况下,std::ref()和std::cref()允许你通过引用传递参数
* 按值传递很简单,但可能不能获得很好的性能
* 选择按值传递,除非有充分的理由不这样做
* 确保返回值是按值传递的
* 依靠测试而不是直觉评估正确性和性能