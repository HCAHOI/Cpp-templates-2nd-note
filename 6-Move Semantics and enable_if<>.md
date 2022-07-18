# 6 Move Semantics and *enable_if<>* 移动语义与*enable_if<>*

提示:在此之前我已经通过《Effective Modern C++》对左值右值、移动语义、完美转发和引用折叠有了一定的认知

## 6.1 Perfect Forwarding 完美转发

如果我们想要对得到的参数进行传递，使得：

* 对于可修改的对象,要使其仍然可以被修改
* 对于常量要保持其常量性
* 对于可移动(或者说可"窃取",因为其即将过期), 应该作为可移动对象转发

那么我们可能需要创造出一批重载函数

```cpp
foo(X& x);
foo(X const& x);
foo(X&& x);
```

注意:对于可移动对象的代码和其他代码不同,它需要std::move(),因为移动语义是不传递的,即使第三个foo()参数被标为&&,但直接将`X x`的x直接传入会传入第二个foo()

标准完美转发模版

```cpp
template<typename T>
void foo(T&& x) {
    g(std::forward<T>(x));	//x被转发给g()
}
```

注意std::move()没有模版参数,这是因为std::forward()是完美转发,而std::move()为强制转发为右值

注意X&&是右值引用,而T&&是转发引用语义,这是不同的!

注意只有纯纯的T&&才是转发引用语义,不能加奇奇怪怪的东西比如`typename T::iterator&&`是错误的

完美转发也可以转发可变模版参数

*更多关于完美转发的细节见Section 15.6.3*

## 6.2 Special Member Function Templates 特殊的成员函数模版

我们之前提到过,特殊成员函数也可以使用模版,但有时会发生令人惊讶的行为

简单来说

```cpp
class Person{
private:shi
    std::string name;
public:
    Person() = default;
    
    template<typename STR>
    explicit Person(STR&& n) : name(std::forward<STR>(n)) {};
    
    explicit Person(Person const& n) : name(n.name) {};
    explicit Person(Person&& n) : name(std::move(n.name)) {};
};

std::string s = "name"
Person p1(s);
Person p2(p1)	//Error!
```

这是因为想要调用复制构造函数需要将参数提升为const,按照重载规则(*见*Section 16.2.4),此时复制构造函数的优先级低于完美转发构造函数,而name无法通过Person构造,于是报错.

解决方案可以提供非常量构造函数,但是这不合适(),实际上我们可以使用std::enable_if<>禁用成员模版

## 6.3 Disable Templates with *enable_if<>* 利用*enable_if<>*禁用模版

C++11以后,C++标准库提供了std::enable_if<>使得我们能够在编译期根据一定条件决定是否禁用模版函数

```cpp
template<typename T>
typename std::enable_if<(sizeof(T) > 4)>::type	//不要忘记括号哦
foo() {
}
```

当条件不成立时,foo()的定义将会被忽略,成立后foo()将变为

```cpp
void foo() {
}
```

std::enable_if是一种type traits,它会在编译期计算作为第一个模版参数给定的表达式,行为如下:

* 如果结果为true,它的类型成员`type`将产生一个类型
  * 如果指定了第二个模版参数,`type`就是其指定的类型
  * 否则该类型为void
* 如果为false,则不会定义`type`,根据非常重要的SFINAE(subtitution failure is not a error)原理*详见Section 8.4*,这会导致std::enable_if<>所在的函数模版被忽略

事实上,在C++14后我们已经可以使用

```cpp
template<typename T>
std::enable_if_t<(sizeof(T) > 4)>	//不要忘记括号哦
foo() {
}
```

注意enable_if表达式是很笨拙的,通常的方式是,使用带有默认值的附加模版函数参数

```cpp
template<typename T,
        typename = std::enable_if_t<(sizeof(T) > 4)>>
void foo() {
}
```

这将会生成

```cpp
template<typename T,
        typename = void>
void foo() {
}
```

其中的`std::enable_if_t<(sizeof(T) > 4)>`可以用using简化

*std::enable_if<>的实现见Section 20.3*

## 6.4 Using *enable_if<>* 使用*enable_if<>*

现在我们可以使用std::enable_if<>解决6.2节遇到的问题了

具体方法是在传入类型是std::string或可以转换为std::string时使用转发构造函数

```cpp
template<typename STR,
        typename = std::enable_if_t<std::is_convertible_v<STR, std::string>>>
explicit Person(STR&& n) : name(std::forward<STR>(n)) {};
```

这里我们使用std::is_convertible要求STR必须可以隐式转换为std::string,我们也可使用std::is_constructible启用显式转换用于初始化,这时我们要调换两个模版参数的位置

```cpp
template<typename STR,
        typename = std::enable_if_t<std::is_constructible_v<std::string, STR>>>
explicit Person(STR&& n) : name(std::forward<STR>(n)) {};
```

*std::is_convertible<>的细节见Section D.3.2*

*std::is_constructible<>的细节见Section D.3.3*

*std::enable_if<>在可变参数模版上的应用见Section D.6*

**Disabling Special Member Functions**

(如何阻止有模版特殊成员函数时缺省特殊成员函数的生成 : delete)

## 6.5 Using Concepts to Simplify *enable_if<>* Expressions 使用concepts简化*enable_if<>*的使用

我们会觉得std::enable_if<>很难理解,因为我们向模版参数中添加了一个额外的参数并"滥用"了它.

原则上,我们只需要一种语言特性,它允许我们指定函数的需求或约束,如果没有满足,函数就会被忽略

这就是大家期待已久的特性concepts的应用,书中作者遗憾地告诉我们C++17并没有将concepts列入标准.但是令人高兴的是,concepts已经成为了C++20标准的一部分!🥳🥳🥳

我们可以简单地这样写

```cpp
template<typename STR>
requires std::is_convertible_v<STR, std::string>
explicit Person(STR&& n) : name(std::forward<STR>(n)) {};
```

我们可以使用concept简化require

```cpp
//global/namespace scope
template<typename T>
concept ConvertibleToString = std::is_convertible_v<T, std::string>;

//inside class
template<typename STR>
requires ConvertibleToString<STR>
explicit Person(STR&& n) : name(std::forward<STR>(n)) {};
```

甚至可以

```cpp
template<ConvertibleToString STR>
explicit Person(STR&& n) : name(std::forward<STR>(n)) {};
```

*关于concepts的更多信息见附录E*

## 6.6 Summary

* 在模版中,通过将参数声明为转发引用(万能引用), 并在转发调用中使用std::forward<>(),我们可以完美转发参数
* 当使用完美转发成员函数模版时,可能比预定义的特殊成员函数更加匹配
* 使用std::enable_if<>,可以在编译时当条件为false时禁用模版
* 使用std::enable_if<>,可以避免模版特殊成员函数比预定义特殊成员函数匹配更好的问题
* 通过删除预定义特殊成员函数(以及使用std::enable_if<>),可以对特殊成员函数进行模版化
* concepts允许我们对模版函数进行限制时使用更符合直觉的语法