# 当`auto`推导出非预期类型时应当使用显式的类型初始化
条款5解释了使用`auto`关键字声明变量比直接显式声明类型提供了一系列的技术优势，但是有时候`auto`的类型推导会和你想的南辕北辙。举个栗子，假设我有一个函数接受一个`Widget`返回一个`std::vector<bool>`，其中每个`bool`表征`Widget`是否接受一个特定的特性：
```
std::vector<bool> features(const Widget& w);
```

进一步的，假设第五个`bit`表示`Widget`是否有高优先级。我们可以这样写代码：
```
Widget w;
...
bool highPriority = features(w)[5];
...
processWidget(w, highPriority);  // 配合优先级处理w
```
这份代码没有任何问题。它工作正常。但是如果我们做一个看似不会出错的修改，把`highPriority`的显式类型换成`auto`：
```
auto highPriority = features(w)[5];
```
所有的代码还是可以编译，但是它的行为变得不可预测：
```
processWidget(w, highPriority);
```

在使用`auto`的代码中，`highPriority`的类型已经不是`bool`了。尽管`std::vector<bool>`从概念上说是`bool`的容器，对`std::vector<bool>`的`operator[]`运算符并不一定是返回容器中的元素的引用（`std::vector::operator[]`对所有类型都返回引用，就是除了`bool`）。事实上，它返回的是一个`std::vector<bool>::reference`对象（是一个在`std::vector<bool>`中内嵌的`class`）。

`std::vector<bool>::reference`存在是因为`std::vector<bool>`是对`bool`数据封装的模板特化，一个`bit`对应一个`bool`。这就给`std::vector::operator[]`带来了问题，因为`std::vector<T>`的`operator[]`应该返回一个`T&`，但是`C++`禁止`bits`的引用。没办法返回一个`bool&`，`std::vector<T>`的`operator[]`于是就返回了一个行为上和`bool&`相似的对象。想要这种行为成功，`std::vector<bool>::reference`对象必须能在`bool&`能处的语境中使用。在`std::vector<bool>::reference`对象的特性中，是它隐式的转换成`bool`才使得这种操作得以成功。

再次阅读原先的代码：
```
bool highPriority = features(w)[5];  // 直接显式指明highPriority的类型
```
这里，`features`返回了一个`std::vector<bool>`对象，`operator[]`被调用。`operator[]`返回一个`std::vector<bool>::reference`对象，然后隐式的转换成`highPriority`需要用来初始化的第五个`bit`的数值来结束`highPriority`的赋值，这也是我们所预期的。

和使用`auto`的`highPriority`声明进行对比：
```
auto highPriority = features(w)[5];  // 推导highPriority的类型
```

这次，`features`返回一个`std::vector<bool>`对象，而且，`operator[]`再次被调用。`operator[]`继续返回一个`std::vector<bool>::reference`对象，但是现在有一个变化，因为`auto`推导`highPriority`的类型。`highPriority`根本没有`features`返回的`std::vector<bool>`的第五个`bit`的数值。

调用`features`会返回一个临时的`std::vector<bool>`对象。这个对象是没有名字的，我们暂且把它叫做`temp`，`operator[]`是在`temp`上调用的，`std::vector<bool>::reference`返回一个由`temp`管理的包含一个指向包含`bits`的数据结构的指针，在`word`上面加上偏移定位到第五个`bit`。`highPriority`也是一个`std::vector<bool>::reference`对象的一份拷贝，所以`highPriority`也在`temp`中包含一个指向`word`的指针，加上偏移定位到第五个`bit`。在这个声明的结尾，`temp`被销毁，因为它是个临时对象。因此，`highPriority`包含一个野指针，这也就是调用`processWidget`会造成未定义行为的原因：
```
processWidget(w, highPriority);  // 未定义的行为，highPriority包含野指针
```
`std::vector<bool>::reference`是代理类应用的一个栗子：一个类的存在就是为了模拟和增加其他类型的行为。代理类被广泛使用。`std::vector<bool>::reference`的存在就是为了提供一个对`std::vector<bool>`的`operator[]`的错觉，让它返回一个对`bit`的引用，而且标准库的智能指针类型（参考第四章）也是一些托管资源的代理类，使得它们的资源管理类似于原始指针。代理类的功能是良好确定的。

一些代理类被设计用来隔离用户，这就是`std::shared_ptr`和`std::unique_ptr`的情况。另外一些代理类是为了一些或多或少的不可见性。`std::vector<bool>::reference`就是这样一个“不可见”的代理，和它类似的是`std::bitset`，对应的是`std::bitset::reference`。

作为一个通用的法则，“不可见”的代理类不能和`auto`愉快的玩耍。这种类常常它的生命周期不会被设计成超过一个单个的语句，所以创造这样的类型的变量是会违反库的设计假定。因此需要避免使用下面的代码形式：
```
auto someVar = expression of "invisible" proxy class type;
```

代理类通过函数签名可以明确，这儿还是`std::vector<bool>::operator[]`的栗子：
```
namespace std {
    template<class Allocator>
    class vector<bool, Allocator> {
        public:
        ...
        class reference { ... };
        reference operator[](size_type n);
        ...
    };
}
```
假设你知道对`std::vector<T>`的`operator[]`常常返回一个`T&`，在这个节选的部分代码里面，非常规的`operator[]`的返回类型一般就表征了代理类的使用。

一旦`auto`被决定作为推导代理类的类型而不是它被代理的类型，它就不需要涉及到关于`auto`，`auto`自己本身没有问题。问题在于`auto`推导的类型不是所想让它推导出来的类型。解决方案就是强制一个不同的类型推导。这种方法叫做显式的类型初始化原则。

显式的类型初始化原则涉及到使用`auto`声明一个变量，但是转换初始化表达式到`auto`想要的类型。下面就是一个强制`highPriority`类型到`bool`的栗子：
```
auto highPriority = static_cast<bool>(features(w)[5]);
```

这里，`features(w)[5]`还是返回一个`std::vector<bool>::reference`的对象，强制类型转换改变了表达式的类型成为`bool`，然后`auto`才推导其作为`highPriority`的类型。在运行的时候，从`std::vector<bool>::operator[]`返回的`std::vector<bool>::reference`对象支持执行转换到`bool`的行为，作为转换的一部分，从`features`返回的仍然存活的指向`std::vector<bool>`的指针被间接引用。这样就在运行的开始避免了未定义行为。索引5然后放置在`bits`指针的偏移上，然后暴露的`bool`就作为了`highPriority`的初始化数值。

## 归纳
* 在初始化表达式中，“不可见”代理类会导致`auto`推导出“不符合预期”的类型
* 显式的类型初始化原则会强制`auto`推导出你想要的类型数据