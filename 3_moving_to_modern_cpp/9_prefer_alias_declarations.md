# 优先使用声明别名而不是`typedef`
条款18可以说服你使用`std::unique_ptr`也是个好想法，但是我想绝对我们中间没有人喜欢写像这样`std::unique_ptr<std::unordered_map<std::string, std::string>>`的代码多于一次。

为了避免这样的情况，推荐使用一个`typedef`：
```
typedef std::unique_ptr<std::unordered_map<std::string, std::string>> UPtrMapSS;
```
但是`typedef`家族是有如此浓厚的`C++98`气息。它们的确可以在`C++11`下工作，但是`C++11`也提供了声明别名（`alias declaration`）：
```
using UPtrMapSS = std::unique_ptr<std::unordered_map<std::string, std::string>>;
```
考虑到`typedef`和声明别名具有完全一样的意义，推荐其中一个而排斥另外一个的坚实技术原因是容易令人生疑的。这样的质疑也是合理的。

技术原因当然存在，但是在我提到之前，我想说的是，很多人发现使用声明别名可以使涉及到函数指针的类型的声明变的容易理解：
```
// FP等价于一个函数指针，这个函数的参数是一个int类型和std::string常量类型，没有返回值
typedef void (*FP)(int, const std::string&);  // typedef

// 同上
using FP = void (*)(int, const std::string&);  // 声明别名
```
当然，上面任何形式都不是特别让人容易下咽，并且很少有人会花费大量的时间在一个函数指针类型的标识符上，所以这很难当做选择声明别名而不是`typedef`的不可抗拒的原因。

但是，一个不可抗拒的原因是真实存在的：模板。尤其是声明别名有可能是模板化的（这种情况下，它们被称为模板别名（`alias template`）），然而`typedef`这里只能说句“臣妾做不到”。模板别名给`C++11`程序员提供了一个明确的机制来表达在`C++98`中需要黑客式的将`typedef`嵌入在模板化的`struct`中才能完成的东西。举个栗子，给一个使用个性化的分配器`MyAlloc`的链接表定义一个标识符。使用别名模板，这就是小菜一碟：
```
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;  // MyAllocList<T>等同于std::list<T, MyAllocM<T>>

MyAllocList<Widget> lw;  // 客户端代码
```
使用`typedef`，你不得不从草稿图开始去做一个蛋糕：
```
template<typename T>
struct MyAllocList {
    typedef std::list<T, MyAlloc<T>> type;
};  // MyAllocList<T>::type等同于std::list<T, MyAllocM<T>>

MyAllocList<Widget>::type lw;  // 客户端代码
```
如果你想在一个模板中使用`typedef`来完成一个节点类型可以被模板参数指定的链接表的任务，你必须在`typedef`名称使用之前使用`typename`：
```
template<typename T>  // Widget<T>包含一个MyAllocList<T>作为一个数据成员
class Widget {
private:
    typename MyAllocList<T>::type list;
    ...
};
```
此处，`MyAllocList<T>::type`表示一个依赖于模板类型参数`T`的类型，因此`MyAllocList<T>::type`是一个依赖类型（`dependent type`），`C++`中许多令人喜爱的原则中的一个就是在依赖类型的名称之前必须冠以`typename`。

如果`MyAllocList`被定义为一个声明别名，就不需要使用`typename`（就像笨重的`::type`后缀）：
```
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;  // 和之前一样

template<typename T>
class Widget {
private:
    MyAllocList<T> list;  // 没有typename，没有::type
    ...
};
```
对你来说，`MyAllocList<T>`（使用模板别名）看上去依赖于模板参数`T`，正如`MyAllocList<T>::type`（使用内嵌的`typedef`）一样，但是你不是编译器。当编译器处理`Widget`遇到`MyAllocList<T>`（使用模板别名），编译器知道`MyAllocList<T>`是一个类型名称，因为`MyAllocList`是一个模板别名：它必须是一个类型。`MyAllocList<T>`因此是一个非依赖类型（`non-dependent type`），标识符`typename`是不需要和不允许的。

另一方面，当编译器在`Widget`模板中遇到`MyAllocList<T>`（使用内嵌的`typename`）时，编译器并不知道它是一个类型名，因为有可能存在一个特殊化的`MyAllocList`，只是编译器还没有扫描到，在这个特殊化的`MyAllocList`中`MyAllocList<T>::type`表示的并不是一个类型。这听上去挺疯狂的，但是不要因为这种可能性而怪罪于编译器。是人类有可能会写出这样的代码。

例如，一些被误导的人们可能会杂糅出像这样的代码：
```
class Wine {...};

template<>
class MyAllocList<Wine> {  // 当T是Wine时，MyAllocList是特殊化的
private:
    enum class WineType {White, Red, Rose};

    WineType type;  // 在这个类中，type是个数据成员
    ...
};
```
正如你看到的，`MyAllocList<Wine>::type`并不是指一个类型。如果`Widget`被使用`Wine`初始化，`Widget`模板中的`MyAllocList<T>::type`指的是一个数据成员，而不是一个类型。在`Widget`模板中，`MyAllocList<T>::type`是否指的是一个类型忠实的依赖于传入的`T`是什么，这也是编译器坚持要求你在类型前面冠以`typename`的原因。

## 归纳
* `typename`不支持模板化，但是别名声明支持
* 模板别名避免了`::type`后缀，在模板中，`typedef`还经常要求使用`typename`前缀