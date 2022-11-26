# 对于独占资源使用`std::unique_ptr`
当你需要一个智能指针时，`std::unique_ptr`通常是最合适的。可以合理假设，默认情况下，`std::unique_ptr`大小等同于原始指针，而且对于大多数操作（包括解引用），他们执行的指令完全相同。这意味着你甚至可以在内存和时间都比较紧张的情况下使用它。如果原始指针够小够快，那么`std::unique_ptr`一样可以。

`std::unique_ptr`体现了专有所有权（`exclusive ownership`）语义。一个`non-null std::unique_ptr`始终拥有其指向的内容。移动一个`std::unique_ptr`将所有权从源指针转移到目的指针。（源指针被设为`null`。）拷贝一个`std::unique_ptr`是不允许的，因为如果你能拷贝一个`std::unique_ptr`，你会得到指向相同内容的两个`std::unique_ptr`，每个都认为自己拥有（并且应当最后销毁）资源，销毁时就会出现重复销毁。因此，`std::unique_ptr`是一种只可移动类型（`move-only type`）。当析构时，一个`non-null std::unique_ptr`销毁它指向的资源。默认情况下，资源析构通过对`std::unique_ptr`里原始指针调用`delete`来实现。

`std::unique_ptr`的常见用法是作为继承层次结构中对象的工厂函数返回类型。假设我们有一个投资类型（比如股票、债券、房地产等）的继承结构，使用基类`Investment`。
```
class Investment {...};
class Stock : public Investment {...};
class Bond : public Investment {...};
class RealEstate : public Investment {...};
```
这种继承关系的工厂函数在堆上分配一个对象然后返回指针，调用方在不需要的时候有责任销毁对象。这使用场景完美匹配`std::unique_ptr`，因为调用者对工厂返回的资源负责（即对该资源的专有所有权），并且`std::unique_ptr`在自己被销毁时会自动销毁指向的内容。`Investment`继承关系的工厂函数可以这样声明：
```
template<typename... Ts>
std::unique_ptr<Investment>
makeInvestment(Ts&&... params);  // 返回指向对象的std::unique_ptr，对象使用给定实参创建
```
调用者应该在单独的作用域中使用返回的`std::unique_ptr`智能指针：
```
{
    ...
    auto pInvestment = makeInvestment( arguments );  // pInvestment是std::unique_ptr<Investment>类型
    ...
}  // 销毁 *pInvestment
```
但是也可以在所有权转移的场景中使用它，比如将工厂返回的`std::unique_ptr`移入容器中，然后将容器元素移入一个对象的数据成员中，然后对象过后被销毁。发生这种情况时，这个对象的`std::unique_ptr`数据成员也被销毁，并且智能指针数据成员的析构将导致从工厂返回的资源被销毁。如果所有权链由于异常或者其他非典型控制流出现中断（比如提前从函数`return`或者循环中的`break`），则拥有托管资源的`std::unique_ptr`将保证指向内容的析构函数被调用，销毁对应资源。（这个规则也有些例外。大多数情况发生于不正常的程序终止。如果一个异常传播到线程的基本函数（比如程序初始线程的`main`函数）外，或者违反`noexcept`说明（见条款14），局部变量可能不会被销毁；如果`std::abort`或者退出函数（如`std::_Exit`，`std::exit`，或`std::quick_exit`）被调用，局部变量一定没被销毁。）

默认情况下，销毁将通过`delete`进行，但是在构造过程中，`std::unique_ptr`对象可以被设置为使用（对资源的）自定义删除器：当资源需要销毁时可调用的任意函数（或者函数对象，包括`lambda`表达式）。如果通过`makeInvestment`创建的对象不应仅仅被`delete`，而应该先写一条日志，`makeInvestment`可以以如下方式实现。
```
auto delInvmt = [](Investment* pInvestment) {  // 自定义删除器，（lambda表达式）
    makeLogEntry(pInvestment);
    delete pInvestment;
};

template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)> makeInvestment(Ts&&... params) {
    std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);

    if (/* 一个Stock对象应被创建 */) {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    } else if (/* 一个Bond对象应被创建 */) {
        pInv.reset(new Bond(std::forward<Ts>(params)...));
    } else if (/* 一个RealEstate对象应被创建 */) {
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }
    return pInv;
}
```
稍后，我将解释其工作原理，但首先请考虑如果你是调用者，情况如何。假设你存储`makeInvestment`调用结果到`auto`变量中，那么你将在愉快中忽略在删除过程中需要特殊处理的事实。当然，你确实幸福，因为使用了`unique_ptr`意味着你不需要关心什么时候资源应被释放，不需要考虑在资源释放时的路径，以及确保只释放一次，`std::unique_ptr`自动解决了这些问题。从使用者角度，`makeInvestment`接口很棒。

这个实现确实相当棒，如果你理解了：
* `delInvmt`是从`makeInvestment`返回的对象的自定义的删除器。所有的自定义的删除行为接受要销毁对象的原始指针，然后执行所有必要行为实现销毁操作。在上面情况中，操作包括调用`makeLogEntry`然后应用`delete`。使用`lambda`创建`delInvmt`是方便的，而且，正如稍后看到的，比编写常规的函数更有效。

* 当使用自定义删除器时，删除器类型必须作为第二个类型实参传给`std::unique_ptr`。在上面情况中，就是`delInvmt`的类型，这就是为什么`makeInvestment`返回类型是`std::unique_ptr<Investment, decltype(delInvmt)>`。（对于`decltype`，更多信息查看条款3）

* `makeInvestment`的基本策略是创建一个空的`std::unique_ptr`，然后指向一个合适类型的对象，然后返回。为了将自定义删除器`delInvmt`与`pInv`关联，我们把`delInvmt`作为`pInv`构造函数的第二个实参。

* 尝试将原始指针（比如`new`创建）赋值给`std::unique_ptr`不能通过编译，因为是一种从原始指针到智能指针的隐式转换。这种隐式转换会出问题，所以`C++11`的智能指针禁止这个行为。这就是通过`reset`来让`pInv`接管通过`new`创建的对象的所有权的原因。

* 使用`new`时，我们使用`std::forward`把传给`makeInvestment`的实参完美转发出去（查看条款25）。这使调用者提供的所有信息可用于正在创建的对象的构造函数。

* 自定义删除器的一个形参，类型是`Investment*`，不管在`makeInvestment`内部创建的对象的真实类型（如`Stock`，`Bond`，或`RealEstate`）是什么，它最终在`lambda`表达式中，作为`Investment*`对象被删除。这意味着我们通过基类指针删除派生类实例，为此，基类`Investment`必须有虚析构函数：
```
class Investment {
public:
    ...
    virtual ~Investment();  // 关键设计部分！
    ...
};  
```
在`C++14`中，函数的返回类型推导存在（参阅条款3），意味着`makeInvestment`可以以更简单，更封装的方式实现：
```
template<typename... Ts>
auto makeInvestment(Ts&&... params)  // C++14
{
    auto delInvmt = [](Investment* pInvestment) {  // 现在在makeInvestment里
        makeLogEntry(pInvestment);
        delete pInvestment; 
    };

    std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);  // 同之前一样
    if ( ... ) {  // 同之前一样
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    } else if ( ... ) {  // 同之前一样
        pInv.reset(new Bond(std::forward<Ts>(params)...));   
    } else if ( ... ) {  // 同之前一样
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));   
    }   
    return pInv;  // 同之前一样
}
```
我之前说过，当使用默认删除器时（如`delete`），你可以合理假设`std::unique_ptr`对象和原始指针大小相同。当自定义删除器时，情况可能不再如此。函数指针形式的删除器，通常会使`std::unique_ptr`的从一个字（`word`）大小增加到两个。对于函数对象形式的删除器来说，变化的大小取决于函数对象中存储的状态多少，无状态函数（`stateless function`）对象（比如不捕获变量的`lambda`表达式）对大小没有影响，这意味当自定义删除器可以实现为函数或者`lambda`时，尽量使用`lambda`：
```
auto delInvmt1 = [](Investment* pInvestment) {  // 无状态lambda的自定义删除器
    makeLogEntry(pInvestment);
    delete pInvestment;
};

template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt1)> makeInvestment(Ts&&... args);  // 返回类型大小是Investment*的大小

void delInvmt2(Investment* pInvestment) {  // 函数形式的自定义删除器
    makeLogEntry(pInvestment);
    delete pInvestment;
}

template<typename... Ts>
std::unique_ptr<Investment, void(*)(Investment*)> makeInvestment(Ts&&... params);  // 返回类型大小是Investment*的指针加上至少一个函数指针的大小
```
具有很多状态的自定义删除器会产生大尺寸`std::unique_ptr`对象。如果你发现自定义删除器使得你的`std::unique_ptr`变得过大，你需要审视修改你的设计。

工厂函数不是`std::unique_ptr`的唯一常见用法。作为实现`Pimpl Idiom`（译注：`pointer to implementation`，一种隐藏实际实现而减弱编译依赖性的设计思想，《Effective C++》条款31对此有过叙述）的一种机制，它更为流行。代码并不复杂，但是在某些情况下并不直观，所以这安排在条款22的专门主题中。

`std::unique_ptr`有两种形式，一种用于单个对象（`std::unique_ptr<T>`），一种用于数组（`std::unique_ptr<T[]>`）。结果就是，指向哪种形式没有歧义。`std::unique_ptr`的`API`设计会自动匹配你的用法，比如`operator[]`就是数组对象，解引用操作符（`operator*`和`operator->`）就是单个对象专有。

你应该对数组的`std::unique_ptr`的存在兴趣泛泛，因为`std::array`，`std::vector`，`std::string`这些更好用的数据容器应该取代原始数组。`std::unique_ptr<T[]>`有用的唯一情况是你使用类似`C`的`API`返回一个指向堆数组的原始指针，而你想接管这个数组的所有权。

`std::unique_ptr`是`C++11`中表示专有所有权的方法，但是其最吸引人的功能之一是它可以轻松高效的转换为`std::shared_ptr`：
```
std::shared_ptr<Investment> sp = makeInvestment( arguments );  // 将std::unique_ptr隐式转换为std::shared_ptr
```
这就是`std::unique_ptr`非常适合用作工厂函数返回类型的原因的关键部分。 工厂函数无法知道调用者是否要对它们返回的对象使用专有所有权语义，或者共享所有权（即`std::shared_ptr`）是否更合适。 通过返回`std::unique_ptr`，工厂为调用者提供了最有效的智能指针，但它们并不妨碍调用者用其更灵活的兄弟替换它。（有关`std::shared_ptr`的信息，请转到条款19。)

## 归纳
* `std::unique_ptr`是轻量级、快速的、只可移动（`move-only`）的管理专有所有权语义资源的智能指针
* 默认情况，资源销毁通过`delete`实现，但是支持自定义删除器。有状态的删除器和函数指针会增加`std::unique_ptr`对象的大小
* 将`std::unique_ptr`转化为`std::shared_ptr`非常简单