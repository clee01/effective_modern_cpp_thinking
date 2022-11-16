# 优先使用`auto`而非显式类型声明
使用显式类型声明有如下潜在的问题，举个栗子：
```
int x;
```
容易写出上面这样的代码——忘记初始化`x`，因此它的值是无法确定的。也许它会被初始化为`0`。

再举个栗子：
```
template<typename It>
void dwim(It b, It e) {
    while (b != e) {
        typename std::iterator_traits<It>::value_type currValue = *b;
        ...
    }
}
```
需要显式指明`typename std::iterator_traits<It>::value_type`来表示被迭代器指向的值得类型，这并不是使用`C++`编程本该有的愉悦体验。

由于`C++11`，得益于`auto`，这些问题都消失了，`auto`变量从它们的初始化推导出其类型，所以它们必须被初始化。
```
int x1;  // 未初始化，且能通过编译
auto x2;  // 不能通过编译
auto x3 = 0;  // 能通过编译，运行良好

template<typename It>
void dwim(It b, It e) {
    while (b != e) {
        auto currValue = *b;
        ...
    }
}
```

由于`auto`使用类型推导，它还可以表示那些仅仅被编译器知晓的类型：
```
auto derefUPLess =   // comparison func.
    [](const std::unique_ptr<Widget>& p1,
       const std::unique_ptr<Widget>& p2) {  // for Widgets pointed by std::unique_ptrs
    return *p1 < *p2;
}
```

在`C++14`中，模板被进一步丢弃，因为使用`lambda`表达式的参数可以包含`auto`：
```
auto derefLess =   // C++14 comparison func.
    [](const auto& p1,
       const auto& p2) {  // for values pointed
    return *p1 < *p2;
```
也许你在想，我们不需要使用`auto`去声明一个持有封装体的变量，因为我们可以使用一个`std::function`对象。

`std::function`是`C++11`标准库的一个模板，它可以使函数指针普通化。鉴于函数指针只能指向一个函数，然而，`std::function`对象可以应用任何可以被调用的对象，就像函数。声明一个名为`func`的`std::function`对象，它可以引用有如下特点的可调用对象：
```
bool(const std::unique_ptr<Widget>&,
     const std::unique_ptr<Widget>&)  // C++11 signature for std::unique_ptr<Widget> comparsion func
```
你可以这么写：
```
std::function<bool(const std::unique_ptr<Widget>&,
                   const std::unique_ptr<Widget>&)> func;
```
因为`lambda`表达式得到一个可调用对象，封装体可以存储在`std::function`对象里面。这意味着，我们可以声明不使用`auto`的`C++11`版本的`derefUPLess`如下：
```
std::function<bool(const std::unique_ptr<Widget>&,
                   const std::unique_ptr<Widget>&)>
    derefUPLess = [](const std::unique_ptr<Widget>& p1,
                     const std::unique_ptr<Widget>& p2) {
                        return *p1 < *p2;
                    }
```

使用`std::function`和使用`auto`并不一样。一个使用`auto`声明持有一个封装的变量和封装体有同样的类型，也仅使用和封装同样大小的内存。持有一个封装体的被`std::function`声明的变量的类型是`std::function`模板的一个实例，并且对任何类型只有一个固定大小。这个内存可能不能满足封装体的需求。出现这种情况时，`std::function`将会开辟堆空间来存储这个封装体。导致的结果就是`std::function`对象一般会比`auto`声明的对象使用更多的内存。由于实现细节中，约束`inline`的使用和提供间接函数的调用，通过`std::function`对象来调用一个封装体比通过`auto`对象要慢。换言之，`std::function`方法通常体积比`auto`大，且慢，还有可能导致内存不足的异常。

`auto`的优点除了可以避免未初始化的变量，变量声明引起的歧义，直接持有封装体的能力。还有一个就是可以避免“类型截断”的问题。举个栗子：
```
std::vector<int> v;
...
unsigned sz = v.size();
```
`v.size()`定义的返回类型是`std::vector<int>::size_type`，但是很少有开发者对此十分清楚。`std::vector<int>::size_type`被指定为一个非符号的整数类型，因此很多程序员认为`unsigned`类型是足够的，然后写出了上面的代码。这将导致一些有趣的后果。比如说在32位`windows`系统上，`unsigned`和`std::vector<int>::size_type`有同样的大小，但是在64位的`windows`上，`unsigned`是32位的，而`std::vector<int>::size_type`是64位的。这意味着上面的代码在32位`windows`系统上工作良好，但是在64位`windows`系统上有时可能不正确，当应用程序从32位移植到64位上，这就比较浪费时间了。使用`auto`可以保证你不必被上面的东西所困扰：
```
auto sz = v.size();  // sz's typs is std::vector<int>::size_type
```
再看如下的代码：
```
std::unordered_map<std::string, int> m;
for (const std::pair<std::string, int>& p : m) {
    ...  // do something with p
}
```
这看上去完美合理。但是有一个问题，意识到`std::unordered_map`的`key`部分是`const`类型的，在哈希表中`std::pair`的类型不是`std::pair<std::string, int>`，而是`std::pair<const std::string, int>`。但是这不是循环体外变量`p`的声明类型。后果就是，编译器竭尽全力找到一种方式，把`std::pair<const std::string, int>`对象转化为`std::pair<std::string, int>`对象。这个过程将通过复制`m`的一个元素到一个临时对象，然后将这个临时对象和`p`绑定完成。在每个循环结束的时候这个临时对象将被销毁。最终这个代码的行为将会令人吃惊，因为你本来想简单的将引用`p`和`m`的每个元素绑定的。当然这种无意的类型不匹配还是可以通过`auto`解决：
```
for (const auot& p : m) {
    ...  // as before
}
```

## 归纳
* `auto`变量一定要被初始化，并且对由于类型不匹配引起的兼容和效率问题有免疫力，可以简单化代码重构，一般会比显式的声明类型敲击更少的键盘
* `auto`类型的变量也受限于条款2和条款6中描述的陷阱