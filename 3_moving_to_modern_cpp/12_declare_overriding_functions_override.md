# 使用`override`关键字声明覆盖的函数
`C++`中的面向对象的编程都是围绕类，继承和虚函数进行的。其中最基础的一部分就是，派生类中的虚函数会覆盖掉基类中对应的虚函数。但是令人心痛的意识到虚函数重载是如此容易搞错。这部分的语言特性甚至看上去是按照墨菲准则设计的，它不需要被遵从，但是要被膜拜。

因为覆盖“`overriding`”听上去像重载“`overloading`”，但是它们完全没有关系，我们要有一个清晰地认识，虚函数（覆盖的函数）可以通过基类的接口来调用一个派生类的函数：
```
class Base{
public:
    virtual void doWork();  // 基类的虚函数
    ...
};

class Derived: public Base{
public:
    virtual void doWork();  // 覆盖Base::doWork, ("virtual" 是可选的)
    ...
};

std::unique_ptr<Base> upb = std::make_unique<Derived>();  // 产生一个指向派生类的基类指针

...

upb->doWork();  // 通过基类指针调用doWork()，派生类的对应函数被调用
```
如果要使用覆盖的函数，几个条件必须满足：
* 基类中的函数被声明为虚的
* 基类中和派生出的函数必须是完全一样的（除了虚析构函数）
* 基类中和派生出的函数的参数类型必须完全一样
* 基类中和派生出的函数的常量特性必须完全一样
* 基类中和派生出的函数的返回值类型和异常声明必须是兼容的

以上的约束仅仅是`C++98`中要求的部分，`C++11`又增加了一条：
* 函数的引用修饰符必须完全一样。成员函数的引用修饰符是很少被提及的`C++11`的特性，所以你之前没有听说过也不要惊奇。这些修饰符使得将这些函数只能被左值或者右值使用成为可能。成员函数不需要声明为虚就可以使用它们：
```
class Widget {
public:
    ...
    void doWork() &;  // 只有当*this为左值时，这个版本的doWork()函数被调用

    void doWork() &&;  // 只有当*this为右值，这个版本的doWork()函数被调用
};
...
Widget makeWidget();  // 工厂函数，返回右值

Widget w;  // 正常的对象（左值）

...

w.doWork();  // 为左值调用Widget::doWork()（即Widget::doWork &）

makeWidget().doWork();  // 为右值调用Widget::doWork() (即Widget::doWork &&)
```
稍后我们会更多介绍带有引用修饰符的成员函数的情况，但是现在，我们只是简单的提到：如果一个虚函数在基类中有一个引用修饰符，派生类中对应的那个也必须要有完全一样的引用修饰符。如果不完全一样，派生类中的声明的那个函数也会存在，但是它不会覆盖基类中的任何东西。

对覆盖函数的这些要求意味着，一个小的错误会产生一个很大不同的结果。在覆盖函数中出现的错误通常还是合法的，但是它导致的结果并不是你想要的。所以当你犯了某些错误的时候，你并不能依赖于编译器对你的通知。例如，下面的代码是完全合法的，乍一看，看上去也是合理的，但是它不包含任何虚覆盖函数——没有一个派生类的函数绑定到基类的对应函数上。你能找到每种情况里面的问题所在吗？即为什么派生类中的函数没有覆盖基类中同名的函数。
```
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};

class Derived : public Base {
public:
    virtual void mf1();
    virtual void mf2(unsigned int x);
    virtual void mf3() &&;
    void mf4() const;
};
```
需要什么帮助嘛？
* `mf1`在`Base`中声明常成员函数，但是在`Derived`中没有
* `mf2`在`Base`中以`int`为参数，但是在`Derived`中以`unsigned int`为参数
* `mf3`在`Base`中有左值修饰符，但是在`Derived`中是右值修饰符
* `mf4`没有继承`Base`中的虚函数

你可能会想，“在实际中，这些代码都会触发编译警告，因此我不需要过度忧虑。”也许的确是这样，但是也有可能不是这样。经过我的检查，发现在两个编译器上，上边的代码被全然接受而没有发出任何警告，在这两个编译器上所有警告是都会被输出的。（其他的编译器输出了这些问题的警告信息，但是输出的信息也不全。）

因为声明派生类的覆盖函数是如此重要，有如此容易出错，所以`C++11`给你提供了一种可以显式的声明一个派生类的函数是要覆盖对应的基类的函数的：声明它为`override`。把这个规则应用到上面的代码得到下面样子的派生类：
```
class Derived : public Base {
public:
    virtual void mf1() override;
    virtual void mf2(unsigned int x) override;
    virtual void mf3() && override;
    void mf4() const override;
};
```
这当然是无法通过编译的，因为当你用这种方式写代码的时候，编译器会把覆盖函数所有的问题揭露出来。这正是你想要的，所以你应该把所有覆盖函数声明为`override`。

使用`override`，同时又能通过编译的代码如下（假设目的就是`Derived`类中的所有函数都要覆盖`Base`对应的虚函数）：
```
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    virtual void mf4() const;
};

class Derived : public Base {
public:
    virtual void mf1() const override;
    virtual void mf2(int x) override;
    virtual void mf3() & override;
    void mf4() const override;  // 加上“virtual”也可以，但不是必须的
};
```
注意在这个例子中，代码能正常工作的一个基础就是声明`mf4`为`Base`类中的虚函数。绝大部分关于覆盖函数的错误发生在派生类中，但是也有可能在基类中有不正确的代码。

对于派生类中覆盖体都声明为`override`不仅仅可以让编译器在应该要去覆盖基类中函数而没有去覆盖的时候可以警告你。它还可以帮助你预估一下更改基类里的虚函数的标识符可能会引起的后果。如果在派生类中到处使用了`override`，你可以改一下基类中的虚函数的名字，看看这个举动会造成多少损害（即，有多少派生类无法通过编译），然后决定是否可以为了这个改动而承受它带来的问题。如果没有`override`，你会希望此处有一个无所不包的测试单元，因为，正如我们看到的，派生类中那些原本被认为要覆盖基类函数的部分，不会也不需要引发编译器的诊断信息。

关于`override`想说的就这么多，但对于成员函数引用限定（`reference qualifiers`）还有一些内容。我之前承诺我会在后面提供更多的关于它们的资料，现在就是"后面"了。

如果我们想写一个函数只接受左值实参，我们声明一个`non-const`左值引用形参：
```
void doSomething(Widget& w);  // 只接受左值Widget对象
```
如果我们想写一个函数只接受右值实参，我们声明一个右值引用形参：
```
void doSomething(Widget&& w);  // 只接受右值Widget对象
```
成员函数的引用限定可以很容易的区分一个成员函数被哪个对象（即`*this`调用）。它和在成员函数声明尾部添加一个`const`很相似，暗示了调用这个成员函数的对象（即`*this`）是`const`的。

对成员函数添加引用限定不常见，但也是可以见到的。举个栗子，假设我们的`Widget`类有一个`std::vector`数据成员，我们提供一个访问函数让客户端可以直接访问它：
```
class Widget {
public:
    using DataType = std::vector<double>;
    ...
    DataType& data() { return values; }

private:
    DataType values;
};
```
这是最具封装性的设计，只给外界保留一线光。但先把这个放一边，思考一下下面的客户端代码：
```
Widget w;
...
auto vals1 = w.date();  // 拷贝w.values到vals1
```
`Widget::data`函数的返回值是一个左值引用（准确的说是`std::vector<double>&`）, 因为左值引用是左值，所以`vals1`是从左值初始化的。因此`vals1`由`w.values`拷贝构造而得，就像注释说的那样。

现在假设我们有一个创建`Widgets`的工厂函数，
```
Widget makeWidget();
```
我们想用`makeWidget`返回的`Widget`里的`std::vector`初始化一个变量：
```
auto vals2 = makeWidget().data();  // 拷贝Widget里面的值到vals2
```
再说一次，`Widgets::data`返回的是左值引用，还有，左值引用是左值。所以，我们的对象（`vals2`）得从`Widget`里的`values`拷贝构造。这一次，`Widget`是`makeWidget`返回的临时对象（即右值），所以将其中的`std::vector`进行拷贝纯属浪费。最好是移动，但是因为`data`返回左值引用，`C++`的规则要求编译器不得不生成一个拷贝。（这其中有一些优化空间，被称作“`as if rule`”，但是你依赖编译器使用这个优化规则就有点傻。）（译注：“`as if rule`”简单来说就是在不影响程序的“外在表现”情况下做一些改变）

我们需要的是指明当`data`被右值`Widget`对象调用的时候结果也应该是一个右值。现在就可以使用引用限定，为左值`Widget`和右值`Widget`写一个`data`的重载函数来达成这一目的：
```
class Widget {
public:
    using DataType = std::vector<double>;
    ...
    DataType& data() & { return values; }  // 对于左值Widgets，返回左值

    DataType data() && { return std::move(values); }  // 对于右值Widgets，返回右值

private:
    DataType values;
};
```
注意`data`重载的返回类型是不同的，左值引用重载版本返回一个左值引用（即一个左值），右值引用重载返回一个临时对象（即一个右值）。这意味着现在客户端的行为和我们的期望相符了：
```
auto vals1 = w.data();  // 调用左值重载版本的Widget::data，拷贝构造vals1

auto vals2 = makeWidget().data();  // 调用右值重载版本的Widget::data，移动构造vals2
```
这真的很棒，但别被这结尾的暖光照耀分心以致忘记了该条款的中心。这个条款的中心是只要你在派生类声明想要重写基类虚函数的函数，就加上`override`。

## 归纳
* 为重写函数加上`override`
* 成员函数引用限定让我们可以区别对待左值对象和右值对象（即`*this`)