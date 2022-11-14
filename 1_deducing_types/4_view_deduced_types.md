# 知道如何查看类型推导
对类型推导结果查看工具的选择和你在软件开发过程中的相关信息有关系。我们要探讨三种可能：编写代码时，在代码编译的时候和在运行的时候得到类型推导的信息。
## `IDE`编辑器
在`IDE`里面的代码编辑器里面当你使用光标悬停在实体之上，常常可以显示出程序实体（例如变量，参数，函数等等）的类型。举个栗子：
```
const int theAnswer = 42;
auto x = theAnswer;
auto y = &theAnswer;
```
一个`IDE`编辑器很可能会展示出`x`的推导类型是`int`，`y`的类型是`const int*`。

对于这样的情况，你的代码必须处在一个差不多可以编译的状态，因为这样可以使得`IDE`接受这种在`IDE`内部运行的一个`C++`编译器（或者至少是一个前端）的信息。如果那个编译器无法能够有足够的能力去感知你的代码并且`parse`你的代码然后去执行类型推导，它就无法展示对应的推导类型了。

对于简单的类型例如`int`，`IDE`里面的信息是正常的。但是我们随后会发现，涉及到更加复杂类型的时候，从`IDE`里面得到的信息并不一定是有帮助的。

## 编译器诊断
一个有效的让编译器展示类型的办法就是故意制造编译问题，编译的错误输出会报告捕捉到的类型相关错误。

首先声明一个类模板，但是并不定义这个模板。
```
template<typename T>
class TD;  // TD == "Type Displayer"
```

尝试实例化这个模板会导致错误信息，因为没有模板的定义实现。想看上述`x`和`y`被推导出来的类型，只要尝试去使用这些类型去实例化`TD`：
```
TD<decltype(x)> xType;  // 编译错误
TD<decltype(y)> yType;  // 同上
```
对上面的代码，编译器输出了诊断信息，其中一部分如下：
```
error: aggregate 'TD<int> xType' has incomplete type and cannot be defined
error: aggregate 'TD<const int *> yType' has incomplete type and cannot be defined
```

## 运行时输出
考虑这样的实现：
```
std::cout << typeid(x).name() << std::endl;
std::cout << typeid(y).name() << std::endl;
```
`GNU`和`Clang`编译器返回`x`的类型是`i`，`y`的类型是`PKi`。这些编译器的输出结果你一旦学会就可以理解它们，`i`意味着`int`，`PK`意味着`point to const`。微软的编译器提供更加直白的输出：`int`对`x`，`int const*`对`y`。

因为这些结果对`x`和`y`而言都是正确的，你可能认为类型输出的问题就此解决了，但是并不能如此轻率。考虑下述一个更加复杂的栗子：
```
template<typename T>
void f(const T& param) {
    std::cout << "T = " << typeid(T).name() << std::endl;
    std::cout << "param = " << typeid(param).name() << std::endl;
}

std::vector<Widget> createVec();

const auto vw = createVec();

if (!vw.empty()) {
    f(&vw[0]);  // 调用f
    ...
}
```

使用`GNU`和`Clang`编译器编译会输出如下结果：
```
T = PK6Widget
param = PK6Widget
```
这些编译器告诉我们`T`和`param`的类型都是`const Widget*`，其中数字`6`代表后面跟着类的名字（`Widget`）的字母字符的长度。

微软的编译器输出：
```
T = class Widget const*
param = class Widget const*
```

三种不同的编译器都产生出了相同的建议性信息，这表明信息是准确的。但是更加仔细的分析，在模板`f`中，`param`的类型是`const T&`。`T`和`param`的类型是一样的难道不会感到很奇怪吗？举个栗子，如果`T`是`int`，`param`的类型应该是`const int&`——根本不是相同的类型。

悲剧的是，`std::type_info::name`的结果并不可靠。在这种情况下，所有的三种编译器报告的`param`的类型都是不正确的。更深入的话，它们本来就是不正确的，因为`std::type_info::name`的特化指定了类型会被当做它们传递给模板函数时按值传递的参数。正如条款1所述，这就意味着如果类型是一个引用，它的引用特性会被忽略，如果在忽略引用之后存在`const`（或者`volatile`），它的`const`特性（或者`volatile`特性）会被忽略。这就是为什么`param`的类型——`const Widget* const&`——被报告成了`const Widget*`。

上述至少说明一个事实——`std::type_info::name`可能在`IDE`中会显示类型失败，但是`Boost TypeIndex`库是被设计成可以成功显示的，举个栗子：
```
#include <boost/type_index.hpp>

template<typename T>
void f(const T& param) {
    using boost::typeindex::type_id_with_cvr;

    // show T
    std::cout << "T = " << type_id_with_cvr<T>().pretty_name() << std::endl;

    // show param's type
    std::cout << "param = " << type_id_with_cvr<decltype(param)>().pretty_name() << std::endl;
    ...
}
```
这个模板函数`boost::typeindex::type_id_with_cvr`接受一个类型参数（我们想知道的类型信息）来正常工作，它不会去除`const`，`volatile`或者引用特性（这也是模板中的`cvr`的意思）。返回的结果是个`boost::typeindex::type_index`对象，其中`pretty_name`成员函数产出一个`std::string`包含一个对人比较友好的类型展示字符串。

通过`f`的这个实现，再次考虑之前使用`typeid`导致推导出现错误的`param`类型信息：
```
std::vector<Widget> createVec();

const auto vw = createVec();

if (!vw.empty()) {
    f(&vw[0]);  // 调用f
    ...
}
```

在`GNU`和`Clang`编译器下面，`Boost.TypeIndex`输出准确的结果如下：
```
T = Widget const*
param = Widget const* const&
```

微软编译器实际上输出的结果是一样的：
```
T = class Widget const*
param = class Widget const* const&
```

## 归纳
* 类型推导的结果常常可以通过`IDE`编辑器，编译器错误输出信息和`Boost TypeIndex`库的结果中得到
* 一些工具的结果不一定有帮助，也不一定准确，所以对`C++`标准的类型推导法则加以理解是很有必要的