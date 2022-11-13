# 理解模板类型推导
声明一个模板函数如下：
```
template<typename T>
void f(ParamType param);
```
一个可能的调用如下：
```
f(expr);  // call f with expr
```
在编译期间，编译器会通过`expr`来推导出两个类型：一个是`T`的，另外一个是`ParamType`。一般来说，这俩类型是不同的，因为`ParamType`通常包含一些类型的装饰，比如`const`或是引用特性。

举个栗子：
```
template<typename T>
void f(const T& param);  // ParamType是const T&
```
如果这样调用的话，
```
int x = 0;
f(x);  // 使用int调用f
```
`T`被推导成`int`，但`ParamType`被推导成`const int&`。

一般会很自然的期望`T`的类型和传递给它的参数类型一致，但很多情况下并不如此，`T`不仅和`expr`类型独立，还和`ParamType`类型独立，下面分别以三种栗子论述下。
### `ParamType`是个非通用的引用或者是一个指针
[通用引用介绍wiki](https://zhuanlan.zhihu.com/p/386312987)

类型的推导过程如下：
* 如果`expr`的类型是个引用，忽略引用的部分
* 再利用`expr`的类型和`ParamType`的对比去判断`T`的类型

举一个栗子，如果这个是我们的模板：
```
template<typename T>
void f(T& param);  // param是一个引用类型
```
我们有这样的代码变量声明：
```
int x = 27;  // x是一个int
const int cx = x;  // cx是一个const int
const int& rx = x;  // rx是const int的引用
```
`param`和`T`在不同的调用下面的类型推导如下：
```
f(x);  // T是int，param是int&

f(cx);  // T是const int，param是const int&

f(rx);  // T是const int，param是const int&
```
> 在第二和第三部分的调用中，当调用者传递一个`const`对象给一个引用参数，`T`被推导成`const int`，`param`被推导成`const int&`，这样，参数变成了`const`引用。对象的`const`特性是`T`类型推导的一部分。

这些栗子展示了左值引用参数的处理方式，在右值引用上也是如此，不做赘述。

如果把`f`的参数类型从`T&`变成`const T&`，情况会是这样的，不会发生太多变化：
```
template<typename T>
void f(const T& param);  // param是const引用

int x = 27;  // 和之前一样
const int cx = x;  // 和之前一样
const int& rx = x;  // 和之前一样

f(x);  // T是int，param是const int&

f(cx);  // T是int，param是const int&

f(rx);  // T是int，param是const int&
```

如果`param`是一个指针（或者是指向`const`的指针）而不是引用，情况也是类似：
```
template<typename T>
void f(T* param);  // param是一个指针

int x = 27;  // 和之前一样
const int* px = &x;  // px是一个指向const int x的指针

f(&x);  // T是int，param是int*

f(px);  // T是const int，param是const int*
```

### `ParamType`是个通用引用
类型的推导过程如下：
* 如果`expr`是左值，`T`和`ParamType`都会被推导成左值引用；这是模板类型`T`被推导成一个引用的唯一情况，另外，尽管`ParamType`利用右值引用的语法来进行推导，但最终被推导出来的类型是左值引用
* 如果`expr`是一个右值，那么就执行之前的法则

举个栗子：
```
template<typename T>
void f(T&& param);  // param现在是一个通用引用

int x = 27;  // 和之前一样
const int cx = x;  // 和之前一样
const int& rx = x;  // 和之前一样

f(x);  // x是左值，所以T是int&，param是int&

f(cx);  // cx是左值，所以T是const int&，param是const int&

f(rx);  // rx是左值，所以T是const int&，param是const int&

f(27);  // 27是右值，所以T是int，param是int&&
```
> 当`ParamType`使用了通用引用，左值参数和右值参数的类型推导大不相同，在非通用的类型推导上绝不会发生。

### `ParamType`既不是指针也不是引用
> 当`ParamType`既不是指针也不是引用时，我们把它处理成`pass-by-value`

基于这个事实可以从`expr`给出推导的法则：
* 和之前一样，如果`expr`是个引用，忽略其引用的部分
* 如果忽略了`expr`的引用特性后，`expr`是个`const`的，也要忽略掉`const`，如果是`volatile`，照样忽略掉

这样的话：
```
int x = 27;  // 和之前一样
const int cx = x;  // 和之前一样
const int& rx = x;  // 和之前一样

f(x);  // T和param都是int

f(cx);  // T和param都还是int

f(rx);  // T和param都还是int
```
但是考虑到`expr`是一个`const`的指针指向一个`const`对象，而且`expr`被通过按值传递给`param`：
```
template<typename T>
void f(T param);  // param按值传递

const char* const ptr = "Fun with pointers";  // ptr是一个const指针，指向一个const对象

f(ptr);  // 给参数传递的是一个const char* const类型
```
> 按照按值传递的类型推导法则，`ptr`的`const`特性会被忽略，这样`param`推导出来的类型就是`const char*`，也就是一个可以被修改的指针，指向一个`const`字符串。

## 数组参数
数组参数声明会被当做指针参数，传递给模板函数的按值传递的数组参数会退化成指针类型，如下：
```
const char name[] = "J. P. Briggs";  // name的类型是const char[13]

template<typename T>
void f(T param);  // 按值传递的参数

f(name);  // T和param都是const char*
```

但是如果将模板`f`的参数修改成引用
```
template<typename T>
void f(T& param);  // 引用参数的模板

f(name);  // T是const char[13]，param是const char(&)[13]
```
> 其中`T`推导出来的实际类型就是数组，且类型推导还包括了数组的长度

有趣的是，声明数组的引用可以创建出一个推导一个数组包含元素长度的模板：
```
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept {
    return N;
}
```
定义为`constexpr`说明函数可以在编译的时候得到其返回值。这就使得创建一个和一个数组长度相同的一个数组，其长度可以从括号初始化：
```
int keyVals[] = {1, 3, 7, 9, 11, 22, 35};  // keyVals有七个元素

std::array<int, arraySize(keyVals)> mappedVals;  // mappedVals的长度也是7
```
> 由于`arraySize`被声明成`noexcept`，这会帮助编译器生成更加优化的代码

## 函数参数
和之前讨论的数组推导类似，函数可以退化成函数指针：
```
void someFunc(int, double);  // someFunc是一个函数，类型是void(int, double)

template<typename T>
void f1(T param);  // 参数按值传递

template<typename T>
void f2(T& param);  // 参数按照引用传递

f1(someFunc);  // param被推导成函数指针，类型是void(*)(int, double)

f2(someFunc);  // param被推导成函数指针引用，类型是void(&)(int, double)
```

## 归纳
* 在模板类型推导的时候，有引用特性的参数的引用特性会被忽略
* 在推导通用引用参数的时候，左值会被特殊处理
* 在推导按值传递参数的时候，`const`和`/`或`volatile`参数会被视为非`const`和非`volatile`
* 在模板类型推导的时候，参数如果是数组或者函数名称，它们会退化成指针，除非参数是引用类型