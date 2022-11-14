# 理解`decltype`
给定一个变量名或者表达式，`decltype`会告诉你这个变量名或表达式的类型。一般情况下其返回类型也是我们所期望的。

我们从典型情况开始讨论，在这些情况下`decltype`不会有令人惊讶的行为，如下：
```
const int i = 0;  // decltype(i) is const int

bool f(const Widget& w);  // decltype(w) is const Widget&
                          // decltype(f) is bool(const Widget&)

struct Point {
    int x, y;  // decltype(Point::x) is int
};

Widget w;  // decltype(w) is Widget

if (f(w)) ...  // decltype(f(w)) is bool

template<typename T>
class vector {
public:
    ...
    T& operator[](std::size_t index);
    ...
};

vector<int> v;  // decltype(v) is vector<int>
...
if (v[0] == 0)  // decltype(v[0]) is int&
```

在`C++11`中，`decltype`最主要的用处可能就是用来声明一个函数模板，在这个函数模板中返回值得类型取决于参数的类型。举个栗子：
```
template<typename Container, typename Index>  // works, but require refinements
auto authAndAccess(Container&, Index i)
    -> decltype(c[i]) {
    authenticateUser();
    return c[i];
}
```
> 此处有一例外，对于`std::vector<bool>`，`[]`操作返回的不是`bool&`，而是一个全新的对象，这个先按下不表，在后续章节会讨论

将`auto`用在函数名之间和类型推导是没有关系的，此处使用了`C++11`的尾随返回类型技术，即函数的返回值类型在函数参数之后声明（`"->"`后边），我们使用了`c`和`i`定义返回值类型。在传统的方式下，函数名前面声明返回值类型，`c`和`i`是得不到的，因为此时`c`和`i`还没被声明。

`C++11`允许单语句的`lambda`表达式的返回类型被推导，在`C++14`中这种行为被拓展到包含多语句的所有的`lambda`表达式和函数。意味着上述代码在`C++14`中可以这样被改写：忽略尾随返回类型，仅仅保留开头的`auto`
```
template<typename Container, typename Index>  // C++14, but not quite correct
auto authAndAccess(Container& c, Index i) {
    authenticateUser();
    return c[i];
}  // return type deduced from c[i]
```
条款2解释说，对使用`auto`来表明函数返回类型的情况，编译器将使用模板类型推导，但正如我们所讨论的，对绝大部分类型为`T`的容器，`[]`操作返回的类型是`T&`，然而条款1提到，在模板类型推导过程中，初始表达式的引用会被忽略。思考这对下面代码意味着什么：
```
std::deque<int> d;
...
authAndAccess(d, 5) = 10;  // authenticate user, return d[5],
                           // then assign 10 to it, this won't compile!
```
此处，`d[5]`返回的类型是`int&`，但是`authAndAccess`的`auto`返回类型声明将会剥离这个引用，从而得到的返回类型是`int`。`int`作为一个右值成为真正的函数返回类型。上面代码尝试给一个右值`int`赋值为10。这种行为在`C++`中是禁止的。

为了让`authAndAccess`按照我们的预期工作，我们需要为它的返回值使用`decltype`类型推导，即指定`authAndAccess`要返回的类型正是表达式`c[i]`的返回类型。这个功能已经在`C++14`中通过`decltype(auto)`实现：`auto`指定需要推导，`decltype`表明在推导过程中使用`decltype`推导规则。因此，可以重写`anthAndAccess`如下：
```
template<typename Container, typename Index>  // C++14 works, but still requires refinement
decltype(auto) anthAndAccess(Container& c, Index i) {
    authenticateUser();
    return c[i];
}
```

`decltype(auto)`并不仅限使用在函数返回值类型上。当想对一个表达式使用`decltype`推导规则时：
```
Widget w;
const Widget& cw = w;
auto myWidget1 = cw;  // auto type deduction, myWidget1's type is Widget

decltype(auto) myWidget2 = cw;  // decltype type deduction, myWidget2's type is const Widget&
```

继续转到之前讨论的对于`authAndAccess`的改进，再次看下`C++14`版本的`authAndAccess`的声明：
```
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i);
```

这个容器是通过非`const`左值引用传入的，因为通过返回一个容器元素的引用来修改容器是被允许的。但是这也意味着不可能将右值传入这个函数。右值不能和一个左值引用绑定（除非是`const`的左值引用）。

一个右值容器作为一个临时对象，在`authAndAccess`所在语句的最后被销毁，意味着对容器中一个元素的引用在创建它的语句结束的地方将被悬空。然而，这对于给`authAndAccess`一个临时对象是有意义的。一个用户可能仅仅想拷贝一个临时容器中的一个元素，例如：
```
std::deque<std::string> makeStringDeque();
// make copy of 5th element of deque returned from makeStringDeque
auto s = authAndAccess(makeStringDeque(), 5);
```

支持这样的使用意味着我们需要修改`authAndAccess`的声明来接受左值和右值，重载可以解决这个问题，但是我们将有两个容器函数需要维护，很不优雅。避免这种情况的一个方法是使`authAndAccess`有一个既可以绑定左值又可以绑定右值的引用参数，这正是通用引用所做的，因此可以修改如下：
```
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container&& c, Index i);
```
在`C++11`中，可以将`decltype(auto)`按照如下变通实现：
```
template<typename Container, typename Index>
auto authAndAccess(Container&& c, Index i)
    -> decltype(std::forward<Container>(c)[i]) {
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

另外一个容易被你忽略的地方是本章节开头的那一句话：“一般情况下其返回类型也是我们所期望的”。

为了彻底理解`decltype`的行为，你必须使自己对一些特殊情况比较熟悉。对一个变量名使用`decltype`得到这个变量名的声明类型。变量名属于左值表达式，但这并不影响`decltype`的行为。然而，对于一个比变量名更复杂的左值表达式，`decltype`保证返回的类型是左值引用。因此，如果一个非变量名的类型为`T`的左值表达式，`decltype`推导的类型是`T&`。这很少会产生比较不利的影响，因为绝大部分左值表达式的类型有内在的左值引用修饰符。例如，需要返回左值的函数返回的总是左值引用。但是在`C++14`中支持`decltype(auto)`相结合，函数中返回语句的一个细小改变将会影响这个函数的推导类型。
```
decltype(auto) f1() {
    int x = 0;
    ...
    return x;  // decltype(x) is int, so f1 returns int
}

decltype(auto) f2() {
    int x = 0;
    return (x);  // decltype((x)) is int&, so f2 returns int&
}
```

`f2`返回值类型与`f1`不同，它返回的是对一个局部变量的引用，这种类型的代码在一般情况下都会引起业务上灾难性的后果。

# 归纳
* `decltype`几乎总是得到一个变量或表达式的类型而不需要任何修改
* 对于非变量名的类型为`T`的左值表达式，`decltype`总是返回`T&`
* `C++14`支持`decltype(auto)`，它的行为就像`auto`，从初始化操作来推导类型，但是它推导类型时使用`decltype`的规则