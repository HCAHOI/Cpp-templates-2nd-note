# 1 Function Templates 函数模版

## 1.1 A First Look at Function Templates 初识模版函数

### 1.1.1 Defining the Template 定义模版

定义了简单的模版函数

```cpp
template<typename T>
T max(T a, T b) {
	return b < a ? a : b;
}
```

这里typename可以用class代替,但是由于并不只有类(class)可以作为template parameter,所以统一使用typename.

### 1.1.2 Using the Template 使用模版

展示max()的实例化(instantition)

### 1.1.3 Two-Phase Translation 二阶段翻译

编译器的编译分为两个阶段:

1. definition time:此时不进行实例化,而是检查模版代码本身的正确性,此时进行的检查:
   * 语法正确性,比如是否缺少分号
   * 是否使用了不依赖于模版参数的未知类型或函数名称
   * 检查不依赖于模版参数的static_assert
2. instantition time:此时将再次检查代码,确保代码全部是有效的

*更多讨论见第九章*

## 1.2 Template Argument Deduction 模版参数推导

简单的模版类型推导,唯一值得注意的

```cpp
template<typename T>
T max(T a, T b) {
	return b < a ? a : b;
}
...
int x = 42;
int const &y = x;
max(y, y);	//这里的T推导为int const&
```

```cpp
template<typename T>
T max(T const &a, T const &b) {
	return b < a ? a : b;
}
//...
int x = 42;
int const &y = x;
max(y, y);	//这里的T推导为int
```

## 1.3 Multiple Template Parameters 多模版参数

告诉我们template parameter可以不止一个

```cpp
template<typename T1, typename T2>
T1 max(T1 a, T2 b) {
	return b < a ? a : b;
}
```

### 1.3.1 Template Parameters for Return Types 返回类型的模版参数

```cpp
template<typename RT, typename T1, typename T2>
RT max(T1 a, T2 b) {
	return b < a ? a : b;
}
//...
::max<double>(4.2, 7);	//返回值将会是double类型
```

注意上面的::,调用模版函数会默认在起头加上::,所以也可以不加

### 1.3.2 Deducing the Return Type 返回类型推导

```cpp
template<typename T1, typename T2>
auto max(T1 a, T2 b) {
	return b < a ? a : b;
}
```

**trailing return type**

```cpp
template<typename T1, typename T2>
auto max(T1 a, T2 b) -> decltype(b<a?a:b) {
	return b < a ? a : b;
}
```

这样将返回由decltype对括号内表达式进行推导得到的类型.

**std::decay**

可以去除引用,const与volatile

```cpp
#include <type_traits>

template<typename T1, typename T2>
auto max(T1 a, T2 b) -> typename std::decay<decltype(b<a?a:b)>::type {
	return b < a ? a : b;
}
```

注意这里的typename,这是因为对于程序员自创的class,type成员不一定表示类型,所以需要加上typename告诉编译器这是一个类型.

在C++14以后,已经可以用std::decay_t代替

```cpp
template<typename T1, typename T2>
auto max(T1 a, T2 b) -> std::decay_t<decltype(b<a?a:b)> {
	return b < a ? a : b;
}
```

### 1.3.3 Return Type as Common Type 作为通用类型返回

std::common_type是一个type traits,具有std::decay的功能,具体原理这里还没说,见D.5

```cpp
#include <type_traits>

template<typename T1, typename T2>
std::common_type_t<T1, T2> max(T1 a, T2 b) {
	return b < a ? a : b;
}
//...
max(4.2, 7);	//返回值将会是double类型
max(4, 7.2);	//返回值还会是double类型
```

## 1.4 Default Template Arguments 缺省模版参数

像指定函数形参默认值一样,template parameter也可以指定默认值

```cpp
#include <type_traits>

template<typename T1, typename T2, 
typename RT = std::decay_t<decltype(true : T1() ? T2())>>
RT max(T1 a, T2 b) {
	return b < a ? a : b;
}
//...
::max<double>(4.2, 7);
```

注意这里的括号,使得T1与T2是自建类型的时候程序也可以正确推导.

```cpp
template<typename RT = long, typename T1, typename T2>
RT max(T1 a, T2 b) {
	return b < a ? a : b;
}
//...
::max(4, 7);	//返回值将会是long类型
::max<int>(4, 7);	//返回值将会是int类型
```

## 1.5 Overloading Function Templates 重载函数模版

如果你还不了解重载函数匹配的基本规则,转到附录C

总体原则是---转换越少越匹配,但是但是不能有两个及以上重载函数同时匹配

确保在模版函数前能看到所有重载版本.

```cpp
int max(int a, int b) {
	return b < a ? a : b;
}

template<typename T>
T max(T a, T b) {
	return b < a ? a : b;
}

//...

int main() {
	max(7, 42);			//1
	max(7.0, 42.0);		//2
	max('a' 'b');		//2
	max<>(7, 42);		//2
	max<double>(7, 42);	//2
	max('a', 42.7);		//1
}
```

```cpp
template<typename T1, typename T2>
auto max(T1 a, T2 b) {
	return b < a ? a : b;
}

template<typename RT, typename T1, typename T2>
RT max(T1 a, T2 b) {
	return b < a ? a : b;
}

int main() {
    auto a = max(4, 7.2);				//1
    auto b = max<double, int>(4, 7.2);	//2
    auto c = max<int>(4, 7.2);			//Error!
}
```

## 1.6 But, Shouldn't We...? 但是...我们不应该?...

### 1.6.1 Pass by Value or by Reference? 按值返回还是按引用返回?

按照《Effective C++》中的观点,应该无脑选择按值返回,这里简要描述本书中的观点

* 语法简单
* 编译器可以进行更好的优化 (For example, Return Value Optimization)

* 可以使用移动语义
* 有的时候根本不需要进行移动或拷贝(这不就是说Return Value Opeimization嘛)

* 容易出现内存管理问题
* 如果一定要传引用,使用std::ref()和std::cref() (见Section 7.3)

### 1.6.2 Why Not inline? 为什么不用inline?

编译器比你聪明多了,人家会自己加的,不用你操心了.

### 1.6.3 Why Not constexpr? 为什么不用constexpr?

能用就用.

## 1.7 Summary

(懒得写了)