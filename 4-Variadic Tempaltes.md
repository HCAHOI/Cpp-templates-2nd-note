# 4 Variadic Tempaltes 可变参数模版

从C++11开始，模版可以接受可变数量的模版参数

## 4.1 Variadic Tempaltes 可变参数模版

### 4.1.1 Variadic Template by Example 可变参数模版实例

```cpp
#include <iostream>
void print() {
}	//退出递归
template<typename T, typename ...Types>
void print(T firstArg, Types... args) {
    std::cout << firstArg << '\n';
    print(args...);
}

int main() {
    print(7.5, "hello", std::string("hello world"));
    return 0;
}
```

原理很简单

### 4.1.2 Overloading Variadic and Nonvariadic Templates 重载可变参数和非可变参数模版

由于没有trailing parameter pack的模版函数更受编译器欢迎,所以我们可以这么写

```cpp
#include <iostream>

template<typename T>
void print(T arg) {
    std::cout << arg << '\n';
}

template<typename T, typename... Types>
void print(T firstArg, Types... args) {
	print(firstArg);
    print(args...)
}
```

*前往SectionC.3.1见更多关于重载函数的细节*

### 4.1.3 Operator *sizeof...*

通过`sizeof...`运算符我们可以得到template parameter pack内的元素数量,有的同学可能会想这样实现print()

```cpp
#include <iostream>

//不需要空print()啦

template<typename T, typename ...Types>
void print(T firstArg, Types... args) {
    std::cout << firstArg << '\n';
	if(sizeof...(args) > 0) {
        print(args...);
    }	//Error!
}
```

这是不可以的,因为在编译时,if的两种可能都会被实例化,具体选择哪条路径是运行时决定的,当向print()传入空参数时会发生错误.

C++17以后加入了编译时if使得类似操作变为可能,*见Section8.5*

## 4.2 Fold Expressions 折叠表达式

C++17之后我们可以用二元运算符对parameter pack中的所有元素进行运算,也能指定计算初始值

比如,我们可以轻松计算出所有元素的和

```cpp
template<typename... T>
auto sum(T... args) {
    return (... + args)
}
```

如果parameter pack为空,将会返回ill-formed expression, 对于 && 操作符为1, 对于||操作符为0, 对于逗号运算符为void()

| **Fold Expression**              | **Evaluatio**n                                               |
| -------------------------------- | ------------------------------------------------------------ |
| ( ... *op* *pack* )              | ((( *pack1* *op* *pack2* ) *op* *pack3* ) ... *op* *packN* ) |
| (  *pack* *op* ...)              | ( *pack1* *op* ( *pack2*  *op* ( *packN-1* *op* *packN* )))  |
| ( *init* *op* ... *op* *pack* )  | (((( *init* *op* *pack1* ) *op* *pack2* ) *op* *pack3* ) ... *op* *packN* ) |
| (  *pack* *op* ... *op* *init* ) | ( *pack1* *op* ( *pack2*  *op* ( *packN-1* *op* ( *packN* *op* *init* ))) |

**`.*`与`->*`运算符**

```cpp
#include <iostream>

struct Foo
{
	int x;
	int y;
};

void test(Foo & ref, int Foo::* ptr_to_member)
{
	std::cout << ref.*ptr_to_member << std::endl;
}

void test(Foo * ptr, int Foo::* ptr_to_member)
{
	std::cout << ptr->*ptr_to_member << std::endl;
}

int main()
{
	Foo f;
	f.x = 3;
	f.y = 4;
	
	test(f, &Foo::x);	//获得x,y成员相对基址的偏移
	test(f, &Foo::y);
	test(&f, &Foo::x);
	test(&f, &Foo::y);

	return 0;
}
```

通过折叠表达式,我们可以简化print()函数

```cpp
template<typename... Types>
void print(Types... args) {
    (std::cout << ... << args) << '\n'; 
}
```

但是这样所有参数会输出在一起,不能在中间加空格,考虑重载运算符

```cpp
template<typename T>
class AddSpace {
private:
    T const &ref;
public:
    explicit AddSpace(T const &ref) noexcept : ref(ref) {};
    friend std::ostream& operator<<(std::ostream &os, AddSpace<T> const &s) {
        os << s.ref << ' ';
    }
};

template<typename... Types>
void print(Types... args) {
    (std::cout << ... << AddSpace(args)) << '\n';
}
```

*前往Section12.4.6见更多关于折叠表达式的细节*

## 4.3 Application of Variadic Templates 可变参数模版的应用

可变参数模版在泛型库的编写中起到重要的作用,例如C++标准库

* 传递参数给被shared_ptr控制的堆内对象

* 传递参数给std::thread

  ```cpp
  std::thread t(foo, 42, "hello");	//在单独线程中运行foo(42, "hello");
  ```

* 向vector的构造函数传递参数一般来说,这些参数的传递依靠完美转发(perfect forward)进行

```cpp
namespace {
    template<typename T, typename... Types> shared_ptr<T>
    make_shared(Types&&... args);
    
    class thread {
    public:
        template<typename F, typename... Types>
        explicit thread(F f, Types&&... args);
    };
}
```

注意,可变参数在类型推导上和不可变参数遵循一样的原则.

## 4.4 Variadic Class Template and Variadic Expressions 可变参数类模版和可变参数表达式

 除了上面的用法, parameter pack也可以用于表达式,类模版,using declarations 和 deduction guides.

*全部用法在Section12.4.2有完整的列表*

### 4.4.1 Variadic Expressions 可变表达式

我们可以针对parameter pack进行运算

如果我们想到输出每个参数的double

```cpp
template<typename... T>
void twice(T const& ... args) {
    print(args + args...);
}

twice(7.5, std::string("hello"));
```

相当于调用

```cpp
print(7.5 + 7.5, 
	std::string("hello") + std::string("hello");
);
```

如果输出每个元素+1

```cpp
template<typename... T>
void addOne(T const &... args) {
    print(args + 1...);		//Error,看起来是点了三个小数点的1
    print(args + 1 ...);	//OK
    print((args + 1)...);	//OK
}
```

在C++17之后,利用折叠表达式,我们可以判定parameter pack中所有元素的类型是否相同

```cpp
template<typename T1, typename... TN>
constexpr bool 
isHomogeneous(T1, TN...) {
    std::cout << (std::is_same_v<T1, TN> && ...)
}
```

### 4.4.2 Variadic Indices 可变参数索引

目录的翻译是可变参数指数,但是我觉得可变索引更贴合一点

```cpp
template<typename C, typename... Idx>
void printElems(C const& container, Idx... idx) {
    print(container[idx]...);
}

std::vector<std::string> v{"h", "e", "l", "l", "o"};
printElems(v, 2, 0, 3);
```

也可以使用非类型模版参数

```cpp
template<typename C, std::size_t... idx>
void printElems(C const& container) {
	print(container[idx]...);
}
```

### 4.4.3 Variadic Class Templates 可变类模版

可变参数也可以用于类模版,典型的例子就是std::tuple和std::variant

```cpp
template<typename... Elems>
class Tuple;

Tuple<int, double, std::string> t;	//存储int,double和string
```

*也会在Chapter25介绍*

```cpp
template<typename... Elems>
class Variant;

Variant<int, double, std::string> v;//存储int,double或string
```

*也会在Chapter26介绍*

当然非类型模版参数在类模版仍然可用

我们都知道std::array和std::tuple使用编译时指定索引的`std::get<[index]>([container])`取用元素,我们可定义一个类作为索引列表

```cpp
template<std::size_t...>
class Indices {  
};

template<typename T, std::size_t... Idx>
void printByIdx(T t, Indices<Idx...>) {
    print(std::get<Idx>(t))
}
```

是不是看不大懂捏,这是一种元编程,所以现在看不懂是很正常的

*关于元编程(meta programming)的更多知识将在Section8.1与Chapter23讨论*

*元编程已经是一种黑魔法,不要走火入魔捏*

### 4.4.4 Variadic Deduction Guides 可变参数Deduction Guides

也许可以翻译成可变参数推导指示

C++标准库在std::array中进行了以下操作

```cpp
namespace std{
	template<typename T, typename... U> array(T, U...)
    -> array<enable_if_t<(is_same_v<T, U> && ...), T>, 
    1 + sizeof...(U)>
}

std::array a = {42, 42, 42};	//C++17以后
```

### 4.4.5 Variadic Base Classes and *using* 可变参数基类和*using*

```cpp
class Custormer{
    //基类
};

class CustomerEq{
    bool operator() (Custormer const &lhs, Custormer const &rhs);
}

class CustomerHash{
    size_t operator() (Custormer const &c);
}

template<typename... Bases>
struct Overloader {
    using Bases::operetor()...;	//OK since C++17
}

int main() {
    using CustomerOp = Overloader<CustomerEq, CustomerHash>;
    
    std::unordered_map<Customer, CustomerEq, CustomerHash> m1;
    std::unordered_map<Customer, CustomerOp, CustomerOp> m2;	//same effect
}
```

*前往Section26.4见这种奇妙技术的引用*

## 4.5 Summary

* 通过使用parameters pack, 模版可以通过可变参数定义
* 为了处理这些参数,需要一个递归函数和一个重载非可变参数函数作为终点
* 操作符`sizeof...`可以获得parameter pack内的参数数量
* 可变参数模版的一个典型应用是转发参数到另一个函数
* 通过使用折叠表达式, 可以对parameter pack中的所有元素使用同一种操作符