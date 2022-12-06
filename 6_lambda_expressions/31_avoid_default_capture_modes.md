# 避免使用默认捕获模式
`C++11`中有两种默认的捕获模式：按引用捕获和按值捕获。但默认按引用捕获模式可能会带来悬空引用的问题，而默认按值捕获模式可能会诱骗你让你以为能解决悬空引用的问题（实际上并没有），还会让你以为你的闭包是独立的（事实上也不是独立的）。

这就是本条款的一个总结。如果你偏向技术，渴望了解更多内容，就让我们从按引用捕获的危害谈起吧。

按引用捕获会导致闭包中包含了对某个局部变量或者形参的引用，变量或形参只在定义`lambda`的作用域中可用。如果该`lambda`创建的闭包生命周期超过了局部变量或者形参的生命周期，那么闭包中的引用将会变成悬空引用。举个栗子，假如我们有元素是过滤函数（`filtering function`）的一个容器，该函数接受一个`int`，并返回一个`bool`，该`bool`的结果表示传入的值是否满足过滤条件：
```
using FilterContainer =
    std::vector<std::function<bool(int)>>;

FilterContainer filters;  // 过滤函数
```
我们可以添加一个过滤器，用来过滤掉`5`的倍数：
```
filters.emplace_back([](int value) {
    return value % 5 == 0;
});
```
然而我们可能需要的是能够在运行期计算除数（`divisor`），即不能将`5`硬编码到`lambda`中。因此添加的过滤器逻辑将会是如下这样：
```
void addDivisorFilter() {
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();

    auto divisor = computeDivisor(calc1, calc2);

    filters.emplace_back([&](int value) {  // 危险！对divisor的引用将会悬空！
        return value % divisor == 0;
    });
}
```
这个代码实现是一个定时炸弹。`lambda`对局部变量`divisor`进行了引用，但该变量的生命周期会在`addDivisorFilter`返回时结束，刚好就是在语句`filters.emplace_back`返回之后。因此添加到`filters`的函数添加完，该函数就死亡了。使用这个过滤器会导致未定义行为，这是由它被创建那一刻起就决定了的。

现在，同样的问题也会出现在`divisor`的显式按引用捕获。
```
filters.emplace_back([&divisor](int value) {  // 危险！对divisor的引用将会悬空！
    return value % divisor == 0;
});
```
但通过显式的捕获，能更容易看到`lambda`的可行性依赖于变量`divisor`的生命周期。另外，写下“`divisor`”这个名字能够提醒我们要注意确保`divisor`的生命周期至少跟`lambda`闭包一样长。比起“`[&]`”传达的意思，显式捕获能让人更容易想起“确保没有悬空变量”。

如果你知道一个闭包将会被马上使用（例如被传入到一个`STL`算法中）并且不会被拷贝，那么在它的`lambda`被创建的环境中，将不会有持有的引用比局部变量和形参活得长的风险。在这种情况下，你可能会争论说，没有悬空引用的危险，就不需要避免使用默认的引用捕获模式。例如，我们的过滤`lambda`只会用做`C++11`中`std::all_of`的一个实参，返回满足条件的所有元素：
```
template<typename C>
void workWithContainer(const C& container) {
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();
    auto divisor = computeDivisor(calc1, calc2);

    using ContElemT = typename C::value_type;
    using std::begin;
    using std::end;

    if (std::all_of(
        begin(container), end(container), [&](const ContElemT& value) {  // 如果容器内所有值都为除数的倍数
            return value % divisor == 0;
        }
    )) {
        ...  // 它们...
    } else {
        ...  // 至少有一个不是的话...
    }
}
```
的确如此，这是安全的做法，但这种安全是不确定的。如果发现`lambda`在其它上下文中很有用（例如作为一个函数被添加在`filters`容器中），然后拷贝粘贴到一个`divisor`变量已经死亡，但闭包生命周期还没结束的上下文中，你又回到了悬空的使用上了。同时，在该捕获语句中，也没有特别提醒了你注意分析`divisor`的生命周期。

从长期来看，显式列出`lambda`依赖的局部变量和形参，是更加符合软件工程规范的做法。

额外提一下，`C++14`支持了在`lambda`中使用`auto`来声明变量，上面的代码在`C++14`中可以进一步简化，`ContElemT`的别名可以去掉，`if`条件可以修改为：
```
if (std::all_of(  // C++14
    begin(container), end(container), [&](const auto& value) {
        return value % divisor == 0;
    }
))
```
一个解决问题的方法是，`divisor`默认按值捕获进去，也就是说可以按照以下方式来添加`lambda`到`filters`：
```
filters.emplace_back([=](int value) {  现在divisor不会悬空了
    return value % divisor == 0;
});
```
这足以满足本实例的要求，但在通常情况下，按值捕获并不能完全解决悬空引用的问题。这里的问题是如果你按值捕获的是一个指针，你将该指针拷贝到`lambda`对应的闭包里，但这样并不能避免`lambda`外`delete`这个指针的行为，从而导致你的副本指针变成悬空指针。

也许你要抗议说：“这不可能发生。看过了第4章，我对智能指针的使用非常热衷。只有那些失败的`C++98`的程序员才会用裸指针和`delete`语句。”这也许是正确的，但却是不相关的，因为事实上你的确会使用裸指针，也的确存在被你`delete`的可能性。只不过在现代的`C++`编程风格中，不容易在源代码中显露出来而已。

假设在一个`Widget`类，可以实现向过滤器的容器添加条目：
```
class Widget {
public:
    ...
    void addFilter() const;  // 向filters添加条目

private:
    int divisor;  // 在widget的过滤器中使用
};
```
这是`Widget::addFilter`的定义：
```
void Widget::addFilter() const {
    filters.emplace_back([=](int value) {
        return value % divisor == 0;
    });
}
```
这个做法看起来是安全的代码。`lambda`依赖于`divisor`，但默认的按值捕获确保`divisor`被拷贝进了`lambda`对应的所有闭包中，对吗？

错误，完全错误。

捕获只能应用于`lambda`被创建时所在作用域里的`non-static`局部变量（包括形参）。在`Widget::addFilter`的视线里，`divisor`并不是一个局部变量，而是`Widget`类的一个成员变量。它不能被捕获。而如果默认捕获模式被删除，代码就不能编译了：
```
void Widget::addFilter() const {
    filters.emplace_back([](int value) {  // 错误！divisor不可用
        return value % divisor == 0;
    });
}
```
另外，如果尝试去显式地捕获`divisor`变量（或者按引用或者按值——这不重要），也一样会编译失败，因为`divisor`不是一个局部变量或者形参。
```
void Widget::addFilter() const {
    filters.emplace_back([divisor](int value) {  // 错误！没有名为divisor局部变量可捕获
        return value % divisor == 0;
    });
}
```
所以如果默认按值捕获不能捕获`divisor`，而不用默认按值捕获代码就不能编译，这是怎么一回事呢？

解释就是这里隐式使用了一个原始指针：`this`。每一个`non-static`成员函数都有一个`this`指针，每次你使用一个类内的数据成员时都会使用到这个指针。例如，在任何`Widget`成员函数中，编译器会在内部将`divisor`替换成`this->divisor`。在默认按值捕获的`Widget::addFilter`版本中，
```
void Widget::addFilter() const {
    filters.emplace_back([=](int value) {
        return value % divisor == 0;
    });
}
```
真正被捕获的是`Widget`的`this`指针，而不是`divisor`。编译器会将上面的代码看成以下的写法：
```
void Widget::addFilter() const {
    auto currentObjectPtr = this;

    filters.emplace_back([currentObjectPtr](int value) {
        return value % currentObjectPtr->divisor == 0;
    });
}
```
明白了这个就相当于明白了`lambda`闭包的生命周期与`Widget`对象的关系，闭包内含有`Widget`的`this`指针的拷贝。特别是考虑以下的代码，参考第4章的内容，只使用智能指针：
```
using FilterContainer =
    std::vector<std::function<bool(int)>>;

FilterContainer filters;

void doSomeWork() {
    auto pw = std::make_unique<Widget>();

    pw->addFilter();
    ...
}  // 销毁Widget；filters现在持有悬空指针！
```
当调用`doSomeWork`时，就会创建一个过滤器，其生命周期依赖于由`std::make_unique`产生的`Widget`对象，即一个含有指向`Widget`的指针——`Widget`的`this`指针——的过滤器。这个过滤器被添加到`filters`中，但当`doSomeWork`结束时，`Widget`会由管理它的`std::unique_ptr`来销毁（见条款18）。从这时起，`filter`会含有一个存着悬空指针的条目。

这个特定的问题可以通过给你想捕获的数据成员做一个局部副本，然后捕获这个副本去解决：
```
void Widget::addFilter() const {
    auto divisorCopy = divisor;  // 拷贝数据成员

    filters.emplace_back([divisorCopy](int value) {  // 拷贝数据成员
        return value % divisorCopy == 0;  // 使用副本
    });
}
```
事实上如果采用这种方法，默认的按值捕获也是可行的。
```
void Widget::addFilter() const {
    auto divisorCopy = divisor;  // 拷贝数据成员

    filters.emplace_back([=](int value) {  // 拷贝数据成员
        return value % divisorCopy == 0;  // 使用副本
    });
}
```
但为什么要冒险呢？当一开始你认为你捕获的是`divisor`的时候，默认捕获模式就是造成可能意外地捕获`this`的元凶。

在`C++14`中，一个更好的捕获成员变量的方式时使用通用的`lambda`捕获：
```
void Widget::addFilter() const {  // C++14
    filters.emplace_back([divisor = divisor](int value) {  // 拷贝divisor到闭包
        return value % divisor == 0;  // 使用这个副本
    });
}
```
这种通用的`lambda`捕获并没有默认的捕获模式，因此在`C++14`中，本条款的建议——避免使用默认捕获模式——仍然是成立的。

使用默认的按值捕获还有另外的一个缺点，它们预示了相关的闭包是独立的并且不受外部数据变化的影响。一般来说，这是不对的。`lambda`可能会依赖局部变量和形参（它们可能被捕获），还有静态存储生命周期（`static storage duration`）的对象。这些对象定义在全局空间或者命名空间，或者在类、函数、文件中声明为`static`。这些对象也能在`lambda`里使用，但它们不能被捕获。但默认按值捕获可能会因此误导你，让你以为捕获了这些变量。参考下面版本的`addDivisorFilter`函数：
```
void addDivisorFilter() {
    static auto calc1 = computeSomeValue1();  // 现在是static
    static auto calc2 = computeSomeValue2();  // 现在是static
    static auto divisor = computeDivisor(calc1, calc2);  // 现在是static

    filters.emplace_back([=](int value) {  // 什么也没捕获到！
        return value % divisor == 0;  // 引用上面的static
    });

    ++divisor;  // 调整divisor
}
```

## 归纳
* 默认的按引用捕获可能会导致悬空引用
* 默认的按值捕获对于悬空指针很敏感（尤其是`this`指针），并且它会误导人产生`lambda`是独立的想法