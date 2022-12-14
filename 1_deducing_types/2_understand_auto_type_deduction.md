# 理解`auto`类型推导
除了一个例外，`auto`类型推导就是模板类型推导，模板类型推导和`auto`类型推导是有一个直接的映射。
```
template<typename T>
void f(ParamType param);

f(expr);  // 使用一些表达式来当做调用f的参数
```
在调用`f`的地方，编译器使用`expr`来推导`T`和`ParamType`的类型。当一个变量被声明为`auto`，`auto`相当于模板中的`T`，而对变量做的相关的类型限定就像`ParamType`。
```
auto x = 27;

template<typename T>
void func_for_x(T param);  // 推导x的类型的概念上的模板

func_for_x(27);  // 概念上的调用，param的类型就是x的类型，都是int

const auto cx = x;

template<typename T>
void func_for_cx(const T param);  // 推导cx的类型的概念上的模板

func_for_cx(x);  // 概念上的调用，param的类型就是cx的类型，都是const int

const auto& rx = x;

template<typename T>
void func_for_rx(const T param);  // 推导rx的类型的概念上的模板

func_for_rx(x);  // 概念上的调用，param的类型就是rx的类型，都是const int&
```
条款1把模板类型推导划分成三个部分，基于在通用的函数模板的`ParamType`特性和`param`的类型声明。在一个用`auto`声明的变量上，类型声明代替了`ParamType`的作用，所以也有三种情况：
* 情况1：类型声明是一个指针或者是一个引用，但不是一个通用引用
* 情况2：类型声明是一个通用引用
* 情况3：类型声明既不是一个指针也不是一个引用
```
auto x = 27;  // 情况3，x为int

const auto cx = x;  // 情况3，cx为const int

const auto& rx = x;  // 情况1，rx为const int&

auto&& uref1 = x;  // 情况2，x是int且是左值，所以uref1类型是int&

auto&& uref2 = cx;  // 情况2，cx是const int且是左值，所以uref2类型是const int&

auto&& uref3 = 27;  // 情况2，27是int且是右值，所以uref3类型是int&&
```

条款1讲解了在非引用类型声明里，数组和函数名称如何退化成指针。在`auto`类型推导上也是一样：
```
const char name[] = "R. N. Briggs";  // name类型是const char[13]

auto arr1 = name;  // arr1的类型是const char*

auto& arr2 = name;  // arr2的类型是const char(&)[13]

void someFunc(int, double);  // someFunc的类型是void (*)(int, double)

auto func1 = someFunc;  // func1的类型是void (*)(int, double)

auto& func2 = someFunc;  // func2的类型是void (&)(int, double)
```
如上所示，`auto`类型推导和模板类型推导工作是很类似的，除了有一种情况是不一样的。在`C++98`中，声明一个用27初始化的`int`，我们有两种选择：
```
int x1 = 27;
int x2(27);
```
在`C++11`中，可以通过统一初始化添加下面的代码：
```
int x3 = {27};
int x4{27};
```
综上四种语法，都会生成一种结果：一个拥有27数值的`int`。

如果上述`int`换成`auto`，可能会带来意向不到的结果。
```
auto x1 = 27;  // 类型是int，值是27

auto x2(27);  // 同上

auto x3 = {27};  // 类型是std::initializer_list<int>，值是{27}
auto x4{27};  // 同上
```
所有声明都可以编译，但是它们和被替换的对应语句意义并不一样。头两个的确是一样的。然而后面两个却声明了一个类型为`std::initializer_list<int>`的变量，这个变量里面包含了一个单一的元素27！

这和`auto`的一种特殊类型推导有关系。当使用一对花括号来初始化一个`auto`类型的变量的时候，推导的类型是`std::initializer_list`。如果这种类型无法被推导（比如在花括号中的变量拥有不同的类型），代码就会编译错误。
```
auto x5 = {1, 2, 3.0};  // 错误！不能将T推导成std::initializer_list<T>
```
> 上述代码实际上是有两种类型推导，一种是`auto x5`类型被推导，`x5`必须被推导成`std::initializer_list`。但是`std::initializer_list`也是一个模板，对一些`T`实例化成`std::initializer_list<T>`，这就意味着`T`的类型必须被推导出来，而这个过程却失败了，因为花括号里面的数值并不是单一类型的

对待花括号初始化的行为是`auto`唯一和模板类型推导不一样的地方。当`auto`声明变量被使用一对花括号初始化，推导的类型是`std::initializer_list`的一个实例。但是如果相同的初始化传递给相同的模板，类型推导就会失败，代码不能编译。
```
auto x = {11, 23, 9};  // x的类型是std::initializer_list<int>

template<typename T>
void f(T param);  // 和x声明等价的模板

f({11, 23, 9});  // 错误的！没办法推导T的类型
```
但是，如果明确了模板的`param`的类型是一个不知道`T`类型的`std::initializer_list<T>`：
```
template<typename T>
void f(std::initializer_list<t> initList);

f({11, 23, 9});  // 正确的！T被推导成int，initList的类型是std::initializer_list<int>
```

`C++14`允许`auto`表示推导的函数返回值，而且`C++14`的`lambda`可能会在参数声明里面使用`auto`。但是，这里面使用是复用了模板的类型推导，而不是`auto`的类型推导，所以以下用法是不能通过编译的。
```
auto createInitList() {
    return {1, 2, 3};  // 编译错误：不能推导出{1, 2, 3}的类型
}

...

std::vector<int> v;

auto resetV = [&v](const auto& newValue) { v = newValue; }  // C++14
resetV({1, 2, 3});  // 编译错误，不能推导出{1, 2, 3}的类型
```

## 归纳
* `auto`类型推导通常和模板类型推导类似，但是`auto`类型推导假定花括号初始化列表的类型是`std::initializer_list`，但是模板类型推导却不是这样
* `auto`在函数返回值或者`lambda`参数里面执行模板的类型推导，而不是通常意义下的`auto`类型推导