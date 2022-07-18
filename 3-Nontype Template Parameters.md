# Nontype Template Parameters 非类型模版参数

对于函数模版和类模版,模版参数不一定是类型,也可以是普通值,使用方法相同

## 3.1 Nontype Class Template Parameters 非类型类模版参数

```cpp
template<typename T, std::size_t Maxsize>
class Stack {
private:
    std::array<T, Maxsize> arr;
    //...
};

Stack s<int, 40> s1;
```

也可以指定缺省值

```cpp
template<typename T = float, std::size_t Maxsize = 10>
class Stack {
private:
    std::array<T, Maxsize> arr;
    //...
};
//缺省值应该是符合直觉的,但是float和100似乎都不怎么符合直觉
Stack s2;
```

## 3.2 Nontype Function Template Parameters 非类型函数模版参数

**std::all_of**

```cpp
template<_InputIterator, _Predicate> 
constexpr inline bool 
all_of(_InputIterator __first, _InputIterator __last, _Predicate __pred);
```

对`__first`到`__end`范围内的元素利用`__pred`进行测试,当全部测试结果为true时返回true,否则返回false

```cpp
std::vector<int> v1{1, 2, 3, 4, 5};
std::vector<int> v2{1, 3, 5};

std::cout << std::all_of(std::cbegin(v1), std::cend(v1), [](auto val) {
    return val % 2;
}) << '\n';	//0
std::cout << std::all_of(std::cbegin(v2), std::cend(v2), [](auto val) {
    return val % 2;
}) << '\n';	//1
```

**std::for_each**

```cpp
template<typename _InputIterator, typename _Function>
_Function
for_each(_InputIterator __first, _InputIterator __last, _Function __f);
```

对`__first`到`__end`范围内的元素执行`__f`

```cpp
std::vector<int> v1{1, 2, 3, 4, 5};

std::for_each(std::cbegin(v1), std::cend(v1), [](auto val) {
   return 2 * val; 
});	//v1:2, 4, 6, 8, 10
```

**std::transform**

```cpp
template<typename _InputIterator, typename _OutputIterator, typename _UnaryOperation>
transform(_InputIterator __first, _InputIterator __last,
	      _OutputIterator __result, _UnaryOperation __unary_op);
```

对`__first`到`__end`范围内的元素执行`__unary_op`,结果写在`__result`



对于模版参数也可使用类型推导

```cpp
template<auto Val, typename T = decltype(Val)>
```

也可以

```cpp
template<typename T, T val= T{}>
```

## 3.3 Restrictions for Nontype Template Parameters 非类型模版参数的限制

非类型模版参数的类型只能是常整数(包括enum), 对象,函数或成员的指针, 对象或函数的左值引用或者std::nullptr_t



C++17 : 当向引用或指针传递模版参数时,传入对象不能是字符串字面量, 临时对象,数据成员或其他子对象

C++14 : 同C++17,而且对象必须具有外部或内部链接

C++11 : 同C++17,而且对象必须具有外部链接

因此

```cpp
template<char const *name>
class Myclass {
    //...
};

Myclass<"hello"> x;	//Error!不能是字符串字面量
```

```cpp
extern char const s03[] = "hi";	//外部链接
char const s11[] = "hi";		//内部链接

int main() {
    Myclass<s03> m03;	//OK in all version
    Myclass<s11> m11;	//Ok since C++11
    
    static char const s17[] = "hi";	//没有链接
    Myclass<s17> m17;	//OK since C++17
}
```

*前往Section12.3.3见更多细节*

*前往Section17.2见这部分未来会发生的变化*

**Avoiding Invalid Expressions**

模版参数可以是任意在编译期可以出结果的表达式, 但是如果使用了>运算符,要把整个式子用括号包起来,避免编译器将其识别为模版参数的右括号

## 3.4 Template Parameter Type *auto*

在C++17之后,我们可以将非类型模版参数的类型设置为auto,以获得更大的自由度

```cpp
template<typename T, auto Maxsize>
class Stack {
public:
	using size_type = decltype(Maxsize)	//得到Maxsize的类型,然而decltype和auto的推导规则还是有所不同
    //...
};
```

C++14之后,函数返回值类型可以用auto代替

```cpp
Stack<int, 20u> s1;			//Maxsize为unsigned int
Stack<std::string, 20> s2;	//Maxsize为int

//利用type traits进行确认
template<typename T, auto Mansize>
auto Stack::size() const {
    return numElems;	//Stack内部表示当前元素数量的成员
}

auto size1 = s1.size();	//unsigned int
auto size2 = s2.size();	//int

std::cout << std::is_same_v<decltype(size1), decltype(size2)>;
```

此外`template<decltype(auto) N>`使用auto的规则对N的类型进行推导,使得N可以为引用类型 *细节见Section15.10.1*