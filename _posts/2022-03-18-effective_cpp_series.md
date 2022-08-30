---
title: "Effective 三部曲"
tags: [cpp, notes]
---

这是《Effective C++》、《More Effective C++》以及《Modern Effective C++》的读书笔记

<!--more-->

## Modern Effective C++

当处理一个有右值引用的参数时，形参本身是个左值
```
class Widget {
public：
    Widget(Widget&& rhs); // rhs是一个左值，尽管他有一个右值引用类型
};
```
### 模板类型推导
> 传入左值，T是左值引用，传入右值，T非引用。使用引用折叠推导出参数类型。auto同理。


* ParamType 是普通引用或者是指针，直接比对

```
int x = 27;         // x是一个int
const int cx = x;   // cx是一个const int
const int& rx = x;  // rx是const int的引用

template<typename T> void f(T& param); 
f(x);   // T是int，param的类型时int&
f(cx);  // T是const int，param的类型是const int&
f(rx);  // T是const int，param的类型时const int&
const char name[] = "J. P. Briggs";
f(name);    //T是const char [13]，param的类型时const char(&)[13]

template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept {
    return N; 
}
```

* ParamType 是通用的引用，左值变引用，右值和上一种方法一样

```
template<typename T> void f(T&& param); 
f(x);   // x是左值，所以T是int&，param的类型也是int&
f(cx);  // cx是左值，所以T是const int&，param的类型也是const int&
f(rx);  // rx是左值，所以T是const int&，param的类型也是const int&
f(27);  // 27是右值，所以T是int，所以param的类型是int&&
```

* ParamType 既不是指针也不是引用，忽略CV

```
template<typename T> void f(T param); 
f(x);   // T和param的类型都是int
f(cx);  // T和param的类型也都是int
f(rx);  // T和param的类型还都是int
const char* const ptr = "Fun with pointers";
f(ptr); // T和param的类型还都是const char* 
const char name[] = "J. P. Briggs";
f(name); // T和param的类型还都是const char* 
```

### auto 类型推导
和模板类型推导规则基本一样，花括号初始化的行为不同
```
auto x = { 11, 23, 9 }; // x的类型是 std::initializer_list<int>
template<typename T> void f(T param); 
f({ 11, 23, 9 });       // 编译出错，没办法推导T的类型
template<typename T> void f(std::initializer_list<T> initList);
f({ 11, 23, 9 });       // 编译通过
```
C++14中auto表示推导的函数返回值、lambda参数声明里面使用auto的情况，复用的是模板类型推导，而不是auto类型推导
```
auto createInitList() {
    return { 1, 2, 3 };     // 编译错误
}
auto resetV = [&v](const auto& newValue) {}
resetV({ 1, 2, 3 });         // 编译错误
```
### decltype类型推导
```
template<typename Container, typename Index> 
auto authAndAccess(Container &c, Index i) {  return c[i]; } 
std::deque<int> d; 
authAndAccess(d, 5) = 10;    // 编译错误。返回d[5]，一个int，右值

template<typename Container, typename Index> 
decltype(auto) authAndAccess2(Container &c, Index i) {  return c[i]; }
authAndAccess2(d, 5) = 10;  // 返回int&
```

如果能确定返回类型，可以直接用auto写
```
auto& f(const int& i) {
    return i;
}
```
泛型编程里一般返回类型未知，decltype(auto)就更好用，比如从容器的一个元素推导返回类型
```
template<typename Cont, typename N>
decltype(auto) f(Cont&& c, N n) {
    return std::forward<Cont>(c)[n];
}
```
对于非变量名的类型为 T 的左值表达式， decltype 总是返回 T&
```
decltype(auto) f1() {
    int x = 0; 
    return x;   // decltype(x) is int, so f1 returns int
}
decltype(auto) f2()c{
    int x = 0;
    return (x); // decltype((x)) is int&, so f2 return int&
}
```
### 优先使用auto
std::unorder_map 的 key 部分实际上是 const 类型的
```
std::unordered_map<std::string, int> m;
for (const std::pair<std::string, int>& p : m) {}   // 会产生临时对象
for (const auto& p : m) {}                          // 不会产生临时对象
```
如果不使用auto，你要取p的地址，将得到一个指向临时对象的指针，这个临时对象在每次循环结束时将被销毁。

### 使用deleted函数
deleted函数有一个优势是任何函数都可以标记为deleted，而只有成员函数可被标记为private。deleted函数还能禁止一些模板的实例化。
```
template<typename T> void processPointer(T* ptr);
template<> void processPointer<void>(void*) = delete;

struct Widget {
    template<typename T> void processPointer(T* ptr) { … }
};
template<> void Widget::processPointer<void>(void*) = delete; 
```

### 使用noexcept
移动构造函数、swap函数，最好加上noexcept

在C++98，允许内存释放（memory deallocation）函数（即operator delete和operator delete[]）和析构函数抛出异常是糟糕的代码设计，C++11将这种作风升级为语言规则。默认情况下，内存释放函数和析构函数——不管是用户定义的还是编译器生成的——都是隐式noexcept。因此它们不需要声明noexcept。

### 特殊成员函数的生成
特殊成员函数：默认构造函数，析构函数，拷贝构造函数，拷贝赋值运算符、移动构造函数和移动赋值运算符

声明一个拷贝操作不会限制编译器生成另一个。但是声明一个移动操作，编译器就不再生成另一个。声明了拷贝操作，编译器就不会生成移动操作。反之亦然。声明了析构操作，移动操作不会自动生成。

注意，成员函数模版不会阻止编译器生成特殊成员函数。

### unique_ptr
当使用默认删除器时，可以假设unique_ptr对象和原始指针大小相同。当自定义函数对象形式的删除器，unique_ptr的大小取决于函数对象中存储的状态多少，无状态函数对象（比如不捕获变量的lambda表达式）对大小没有影响。所以自定义删除器尽量使用lambda

### shared_ptr
![](https://raw.githubusercontent.com/CnTransGroup/EffectiveModernCppChinese/master/4.SmartPointers/media/item19_fig1.png)
unique_ptr删除器类型是智能指针类型的一部分，std::shared_ptr则不是。因此`vector<shared_ptr<Widget>>`可以放不同删除器的shared_ptr。不同删除器的shared_ptr也能互相赋值

shared_from_this查找当前对象控制块，然后创建一个新的shared_ptr关联这个控制块。设计的依据是当前对象已经存在一个关联的控制块。如果没有shared_ptr指向当前对象，行为是未定义,通常抛出异常

shared_ptr不能处理数组

### weak_ptr
在调用weak_ptr::expired和解引用操作之间，另一个线程可能对指向这对象的shared_ptr重新赋值或者析构。因此会使用weak_ptr::lock，它返回shared_ptr。如果weak_ptr过期这个shared_ptr为空。或者以weak_ptr为实参构造shared_ptr。如果weak_ptr过期，会抛出一个异常

### 优先使用make_unique、make_shared而非new
```
processWidget(std::shared_ptr<Widget>(new Widget), computePriority());
processWidget(std::make_shared<Widget>(), computePriority());
```
如果computePriority抛出异常，第一个processWidget可能会资源泄露。

make_*函数比调用构造函数更快，因为只需要分配一次内存。

不过make_*函数不能自定义删除器，也不能直接使用花括号初始化。

std::make_shared的话，shared_ptr的控制块与指向的对象放在同一块内存中，这导致只有控制块的内存也被销毁，对象占用的内存才被释放。而只要weak_ptrs引用一个控制块，该控制块就必须继续存在。

### 使用Pimpl时，在实现文件中定义特殊成员函数
```
// widget.h
class Widget() {                  
public:
    Widget();
    struct Impl;              
    std::unique_ptr<Impl> pImpl;
};
// widget.cpp
#include "widget.h"           

struct Widget::Impl { ... };
Widget::Widget() : pImpl(std::make_unique<Impl>())  {}
```
编译会出错，应该在cpp文件中加入析构函数的定义`Widget::~Widget() = default;`。注意我们加了析构函数的定义后，编译器不会帮我们生成其他特殊函数，如果需要的话，我们需要再次在cpp文件中定义。

上面讨论的是unique_ptr，shared_ptr不需要。unique_ptr删除器的类型是这个智能指针的一部分，因此特殊成员函数被调用时，必须已经是一个完成类型。shared_ptr删除器的类型不是该智能指针的一部分，特殊成员函数被使用的时候，指向的对象不必是一个完成类型。
```
auto loggingDel = [](Widget *pw) { ... };
std::unique_ptr<Widget, decltype(loggingDel)> upw(new Widget, loggingDel);
std::shared_ptr<Widget> spw(new Widget, loggingDel);     
```
### 理解move和forward
move和forward只是执行类型转换。
```
class Annotation {
public:
    explicit Annotation(const std::string text)
    ：value(std::move(text))  { … }        

    std::string value;
};
```
这里`text`是拷贝到`Annotation::value`而不是移动。因为text的move结果是`const std::string&&`，调用string的`string(const string& rhs)`构造函数。

不要在你希望能移动对象的时候，声明他们为const。对const对象的移动请求会悄无声息的被转化为拷贝操作。

### 区分通用引用和右值引用
```
template <typename T>
void f(std::vector<T>&& param);     // 右值引用
emplate <typename T>
void f(const T&& param);            // 右值引用
```

### 对右值引用使用std::move，对通用引用使用std::forward
对于形参，避免在右值引用上使用std::forward，通用引用上使用std::move。函数返回值同理。

```
Matrix operator+(Matrix&& lhs, const Matrix& rhs) {
    lhs += rhs;
    return std::move(lhs);	 
}
Matrix operator+(Matrix&& lhs, const Matrix& rhs) {
    lhs += rhs;
    return lhs;	 
}
```
注意lhs是左值。在move返回的版本，lhs直接移动到返回值的内存位置。不用move的话，编译器拷贝对象到返回值的内存空间。

以上讨论的是形参，对于局部变量不适用，因为会阻碍返回值优化，我们直接按值返回即可。返回值优化要求：
1. 局部对象与函数返回值的类型相同
2. 局部对象就是要返回的东西

满足上面要求而编译器不做优化的情况下（比如不同控制流返回不同类型对象，或者局部变量是形参），编译器也会把返回的对象视为右值。

*不满足优化要求的局部变量可以使用move返回？*

### 避免在通用引用上重载
```
template<typename T>
void logAndAdd(T&& name) { ... }

void logAndAdd(int idx) { ... }
```
如果传入short类型，会导致通用引用版本被精确匹配。

编译器会为类生成构造函数，可能在你意料外重载通用引用
```
class Person {
public:
    template<typename T>           
    explicit Person(T&& n)  { ... }

    Person(const Person& rhs);      //拷贝构造函数（编译器生成）
    Person(Person&& rhs);           //移动构造函数（编译器生成）
};
```
这会导致调用的构造函数可能不是你想要的。再引入继承关系，派生类的表现会更加复杂：
```
class SpecialPerson: public Person {
public:
    SpecialPerson(const SpecialPerson& rhs) 
    : Person(rhs)  { … }

    SpecialPerson(SpecialPerson&& rhs)     
    : Person(std::move(rhs))  { … }
};
```
SpecialPerson总是调用基类的通用引用版本的构造函数。

因此要避免使用通用引用实现构造函数，因为对于non-const左值，它们比拷贝构造函数而更匹配，而且会劫持派生类对于基类的拷贝和移动构造函数的调用。

### 通用引用重载的替代方法

* 不重载
* 不使用通用引用，替换为const T&
* 传值
* tag dispatch，但是不能解决类构造函数的问题
* 约束使用通用引用的模板

```
template<typename T>                        
void logAndAddImpl(T&& name, std::false_type) { ... }
template<typename T>                        
void logAndAddImpl(T&& name, std::true_type) { ... }
template<typename T>
void logAndAdd(T&& name) {
    logAndAddImpl(std::forward<T>(name),
            std::is_integral<typename std::remove_reference<T>::type>());
}

class Person {
public:
    template<typename T, typename = 
        std::enable_if_t<!std::is_base_of<Person, std::decay_t<T>>::value>>
    explicit Person(T&& n)  { … }
    
    Person(const Person& rhs);      //拷贝构造函数（编译器生成）
    Person(Person&& rhs);           //移动构造函数（编译器生成）
};
```


### 引用折叠
如果任一引用为左值引用，则结果为左值引用。否则结果为右值引用。

引用折叠发生在四种情况下：模板实例化，auto类型推导，typedef与别名声明的创建和使用，decltype。以typedef为例，其他三种的折叠方式见开头几节：
```
template<typename T>
struct Widget {  typedef T&& RvalueRefToT;  };

Widget<int&> w;
```
此时T会被推导为int&，得到`typedef int& && RvalueRefToT;`，折叠后是`typedef int& RvalueRefToT;`

再举forward的例子，实现方式及使用方式如下：
```
template<typename T>                        
T&& forward(remove_reference_t<T>& param) {
  return static_cast<T&&>(param);
}

template<typename T>
void f(T&& fParam) {
    someFunc(std::forward<T>(fParam)); 
}
```
如果f传入左值类型`int`，则f的T是`int&`，调用`std::forward<int&>(fParam)`，在forward中，调用`static_cast<int& &&>(param)`，折叠后变成`static_cast<int&>(param)`

如果f传入右值类型`int`，则f的T是`int`，调用`std::forward<int>(fParam)`，在forward中，调用`static_cast<int&&>(param)`


### 移动操作可能没那么快
一些容器不存在开销小的移动操作，比如`array`需要把容器中的元素都移动到新对象中，线性时间。对于其他数据存储在堆的容器，如`vector`，只需要修改对象指向堆具体内容的指针，常量时间。

`string`提供了常数时间的移动操作和线性时间的复制操作，但如果采用`SSO`的实现方式，数据存储在对象的缓冲区中，移动操作就不比复制操作快

### 完美转发失败的情况
```
template<typename... Ts>
void fwd(Ts&&... params) { f(std::forward<Ts>(params)...); }
```
如果f(expression)与fwd(expression)执行不同的操作，则完美转发失败。比如发生下面情况：

* 不能推导出fwd的一个或者多个形参类型
* 推导错了fwd的一个或者多个形参类型

```
void f(const std::vector<int>& v);
fwd({ 1, 2, 3 });       // ERROR
auto il = { 1, 2, 3 }; 
fwd(il);                // OK
```
`fwd({ 1, 2, 3 })`无法推导出`initializer_list`，但是`auto`可以，因此`fwd(il)`编译通过。

传递0或者NULL作为空指针给模板时，类型推导会实参推导为一个整型类型而不是指针类型。

fwd传入仅有声明的整型static const数据成员作为实参，会链接出错

传入重载函数的名称和模板名称作为实参，需要指定签名
```
void f(int (*pf)(int));
int processVal(int value);
int processVal(int value, int priority);
template<typename T> T workOnVal(T param) { … }

fwd(processVal);    // ERROR
fwd(workOnVal);     // ERROR

using ProcessFuncType = int (*)(int);
fwd(static_cast<ProcessFuncType>(workOnVal));   // OK
```

pwd传入位域作为实参会出错，因为non-const引用不应该绑定到位域，位域无法直接寻址

### 避免使用默认捕获模式
按引用捕获可能会造成悬空引用，需要避免lambda创建的闭包生命周期超过了局部变量或者形参的生命周期。

按值捕获可能捕获到一个悬空指针。
```
std::vector<std::function<bool(int)>> filters;
class Widget {
public:
    void addFilter() const {
        filters.emplace_back([=](int value) { return value % divisor == 0; });
    }	
    int divisor;       
};
```
捕获只能应用于lambda被创建时所在作用域里的non-static局部变量（包括形参），这里不能捕捉divisor，而是隐式捕捉this指针。因此有可能存在悬空指针：
```
{
    auto pw = std::make_unique<Widget>(); 
    pw->addFilter();
}
```
因此应该改成这样：
```
void Widget::addFilter() const {
    auto divisorCopy = divisor;     // C++11
    filters.emplace_back(
        [=](int value) { return value % divisorCopy == 0; }	
    );
}
void Widget::addFilter() const {
    filters.emplace_back(                   // C++14
        [divisor = divisor](int value) { return value % divisor == 0; }
    );
}
```
对象定义在全局空间或者命名空间，或者在类、函数、文件中声明为static。这些对象也能在lambda里使用，但它们不能被捕获。但默认按值捕获可能会因此误导你，让你以为捕获了这些变量。
```
void addDivisorFilter() {
    static auto divisor = 1;
    filters.emplace_back(
        [=](int value)                          //什么也没捕获到！
        { return value % divisor == 0; }        //引用上面的static
    );
    ++divisor;                                  
}
```
这里lambda没有捕获任何东西，而是引用了static变量divisor，调用addDivisorFilter后divisor都会递增，通过这个函数添加到filters的所有lambda都展示新的行为


### 使用初始化捕获来移动对象到闭包中
C++11缺少移动捕获，C++14使用初始化捕获补救。
```
class Widget;
auto pw = std::make_unique<Widget>();
auto func = [pw = std::move(pw)]{ ... };
```
编译器不支持C++14怎么办？使用可调用对象或者使用bind：
```
auto func = std::bind(
                [](const std::unique_ptr<Widget>& pw){ ... },
                std::make_unique<Widget>()
            );
```
对于每个左值实参，bind对象中的对应对象都是复制构造的。对于每个右值，它都是移动构造的。但是不建议使用bind。


### 对auto&&形参使用decltype以std::forward
```
auto f = [](auto&& x)
         { return func(normalize(std::forward<???>(x))); };
```
`???`应该是什么？在decltype(x)中，如果x绑定左值，decltype(x)就能产生左值引用。x绑定右值，decltype(x)就会产生右值引用。由于引用折叠，forward的模板参数是右值引用时仍然可以工作。因此上述代码应该改成这样：

```
auto f = [](auto&& x) {
            return func(normalize(std::forward<decltype(x)>(x)));
        };
```
### 使用lambda替代bind
```
auto funcB = bind(func, std::chrono::steady_clock::now()+1h, _1);
```
`now()`将在调用bind的时候调用，而不是在funcB调用被调用，在大多数情况下这可能不是我们想要的。需要再用一个bind来包装now()绕过这个问题。

如果有多个func重载函数，bind不能确定选择哪一个，需要将func强制转换为函数指针
```
using Func1ParamType = void(*)(Time t);
auto funcB = bind(static_cast<Func1ParamType>(func),
              std::bind(std::plus<>(),
                        steady_clock::now(),
                        1h), 
              _1);
```
编译器不太可能通过函数指针内联函数，因此lambda能生成比bind更快的代码。在C++14中，通常可以省略标准运算符模板的模板类型实参。

注意，bind总是按值捕获来拷贝实参，但是调用者可以使用std::ref来存储实参。对std::bind对象进行调用，实参通过引用传递。

在C++14中，可以完全使用lambda而抛弃bind。C++11中有两个地方lambda无法做到：移动捕获、绑定带有模板化函数调用运算符的对象

### 并发
*TODO*
### 传值还是传引用
考虑三种定义函数方式
```
struct Widget {
    void addName(const std::string& newName)   
    { names.push_back(newName); }
    void addName(std::string&& newName)        
    { names.push_back(std::move(newName)); }  
    std::vector<std::string> names;
};

struct Widget {
    template<typename T>                            
    void addName(T&& newName) 
    { names.push_back(std::forward<T>(newName)); }
    std::vector<std::string> names;
};

struct Widget {
    void addName(std::string newName) 
    { names.push_back(std::move(newName)); }
    std::vector<std::string> names;
};

Widget w; 
std::string name("Bart");
w.addName(name);          
w.addName(name + "Jenne");  
```

* 重载方式两份代码维护麻烦。左值一次拷贝，右值一次移动
* 通用引用方式模板定义必须放在头文件、可能实例化过多函数、而且有些实参类型不能通过通用引用传递。左值一次拷贝，右值一次移动。
* 传值方式，左值实参，一次拷贝一次移动，右值实参两次移动。

移动成本低且总是被拷贝的可拷贝形参，可以考虑按值传递。

对于不可拷贝形参，如unique_ptr，没必要这么麻烦，直接使用重载版本中的接受右值引用的函数。如果使用传值方式，会先移动构造形参，再移动赋值到数据成员。

使用传值方式如果每次addName不是都会插入元素，那么做的无用功也会有临时对象的开销，不如使用引用方式传递。

传值可能会带来隐藏的内存分配开销。

按值传递会引起切片问题，不适合基类形参类型。

### 置入还是插入
插入函数接受对象去插入，而置入函数接受对象的构造函数接受的实参去插入。这种差异允许置入函数避免插入函数所必需的临时对象的创建和销毁。

```
std::vector<std::string> vs;     
vs.push_back("xyzzy");       
```
这里调用两次构造函数，一次析构函数：
1. 通过字面量`xyzzy`创建string临时对象
2. vector调用移动构造函数，在容器内部创建一个对象
3. string临时对象销毁

```
vs.emplace_back("xyzzy");
```

这里没有临时变量产生

置入不一定会优于插入，但下列条件满足，我们就使用置入：

* 值是通过构造函数添加到容器，而不是直接赋值。因为赋值操作总是需要有个源对象，不可避免会有临时对象创建，这样置入就没有优势
* 传递的实参类型与容器的初始化类型不同。置入优于插入是因为实参不是容器元素类型时，不需要临时对象。如果实参类型与容器元素类型相同，就没有理由期望置入更快
* 容器不拒绝重复项作为新值。如果拒绝重复元素，那么置入创建的元素可能会被销毁，这抵消了没有临时对象节省的开销

有时候，临时对象值得被创建，比如资源管理

```
std::list<std::shared_ptr<Widget>> ptrs;
void killWidget(Widget* pWidget);
ptrs.push_back({new Widget, killWidget});
ptrs.emplace_back(new Widget, killWidget);
```
push_back的形参是shared_ptr的引用，因此一定会生成临时std::shared_ptr。这样如果push_back的内存分配过程抛出异常，shared_ptr已经构造好了，能够调用自定义删除器，不会资源泄漏。如果使用emplace_back，则会资源泄露。

资源管理类是以资源被立即传递给资源管理对象的构造函数为条件的。而在置入函数中，完美转发推迟了资源管理对象的创建。

实际上，我们更应该使用独立语句来创建资源管理对象。
```
std::shared_ptr<Widget> spw(new Widget, killWidget);
ptrs.push_back(std::move(spw)); 
ptrs.emplace_back(std::move(spw));
```

置入操作使用直接初始化，有可能错误地调用explicit的构造函数。插入函数使用拷贝初始化，不能用explicit的构造函数。使用置入函数要确保传递了正确的实参。
```
std::vector<std::regex> regexes;
regexes.emplace_back(nullptr);  // UB 
regexes.push_back(nullptr);     // 编译出错
```

## More Effective C++
### pointer和reference的区别
* 引用不能为空，指针可能为空。指针使用前判断是否为空指针
* 引用指向最初的对象，指针可以重新赋值
* operator[]应该返回一个引用

### 使用C++转型操作符
dynamic_cast用来将指向基类的指针或引用，转型为指向派生类的指针或引用。如果转型失败，会返回null指针（转型对象是指针）或抛出异常（转型对象是引用）。

reinterpret_cast的转换结果与平台相关，不具可移植性。最常用的用途是转换函数指针类型。
```
typedef void (*FuncPtr)();
FuncPtr funcPtrArray[10];
int doSomtthing();
funcPtrArray[0] = reinterpret_cast<FuncPtr>(&doSomtthing);
```
### 不以多态方式处理数组
```
class Base { ... };
class Derive : public Base { ... };
void func(Base array[]) { ... }

Derive deriveArray[10];
func(deriveArray);  // Error
```
取数组的元素是按照基类的大小来对指针进行计算，然而派生类一般都比基类大，会导致计算错误

*想不到为什么会这么用*

### 非必要不提供默认构造函数
不提供默认构造函数，产生一个对应类型的数组会报错，这时可以采用placement new。可能会用不了一些模板容器，需要容器作者谨慎地设计。作为基类的话，要求派生类知道如何构造基类。

*辩证地看待*

### 避免类型转换函数
使用explicit。避免定义类型转换函数。

### 区别i++/++i的重载
*了解即可*

### 不重载&&、||和，操作符
*常识*

### 区别operator new和new operator
new operator是关键字，执行两个操作：分配内存、构造对象。其中分配内存调用的就是operator new。

### 用析构函数避免资源泄漏
*RAII*
### 在构造函数内防止资源泄漏
*使用RAII保护资源*
### 禁止异常流出析构函数之外
两种情况下会调用析构函数：对象正常被销毁、对象被异常处理机制销毁。如果异常离开析构函数，而此时也处于另一个异常中，C++会调用terminate函数结束程序。禁止异常流出析构函数能避免terminate函数在异常传播过程的栈展开被调用；协助确保析构函数完成应该完成的所有工作。
### 了解抛出异常、传递参数、调用虚函数的差异
不论被捕捉的异常是以传值还是传引用方式传递（不可能传指针，会造成类型不吻合），都会发生复制行为。如果以传值方式，甚至会被复制两次

不会表现出多态，比如抛出指向派生类的基类引用，实际上还是抛出基类对象。

```
catch (Base& b) {
    throw;
}
catch (Base& b) {
    throw b;
}
```
第一个语句块不会进行复制，如果最初抛出的是派生类的引用，则第一个语句块传播派生类的异常。第二个语句会复制，总是抛出基类。

catch捕捉的类型只会做两种转换：捕捉基类可以处理派生类异常、捕捉无型指针可以处理有型指针
```
catch (const void*) { ... } // 可捕捉任何指针类型的异常 
```
catch遵循最先吻合的策略处理异常

### 以传引用方式捕捉异常
可以抛出指针，但是要防止catch捕捉到指向不存在对象的指针。

catch-by-value除了有复制两次的问题，也会有派生类被切割的问题。抛出派生类，捕捉到的是基类，会调用基类的虚函数

catch-by-reference只复制一下，没有派生类被切割的问题。抛出派生类，捕捉到的是派生类

### 明智运用exception specifications
不将模板与exception specifications混合使用，因为不知道模板参数类型可能抛出什么异常。

如果需要调用回调函数，可以为回调函数指针类型的声明加上exception specifications。这样就只能注册不抛出异常的回调函数
```
typedef void (*callBackPtr)() throw();
```
### 了解异常处理的成本
*避免使用异常*

### 虚函数表和虚函数指针
有虚函数的类会有一个虚函数指针，指向虚函数表。虚函数表中包含的是指向该类虚函数的指针。如果Derive继承Base，那么Derive会重新定义某些继承的虚函数，修改虚函数表中指针的指向，并加上新的虚函数指针。

虚函数表位置在哪？暴力方法是在每一个需要虚函数表的目标文件都产生虚函数表副本，再由链接器去除重复的副本。另一种方法是将虚函数表放在第一个non-inline、non-pure虚函数定义的目标文件中。

调用虚函数本身不构成性能瓶颈，而在与虚函数无法inline。inline意味着在编译期就将函数本体替换调用动作，这与virtual语义相反。

### 将类的构造器和非成员函数虚化
类的构造器虚化，可以方便地生成不同的派生类
```
class Base {
    virtual Base* clone() = 0;
};
class DeriveA : public Base {
    virtual DeriveA* clone() {  return new DeriveA(*this);  }
};
class DeriveB : public Base {
    virtual DeriveB* clone() {  return new DeriveB(*this);  }
};
```
派生类重新定义虚函数，不一定得使用和基类相同的返回类型

非成员函数虚化，就是非成员函数调用虚函数

## Effective C++
### 不在构造和析构过程中调用virtual函数
基类构造期间调用虚函数，总是调用本身的虚函数。因为基类比派生类更早构造。
### 令operator=返回*this的引用
为了连锁赋值
### operator=中记得处理自我赋值
也要注意异常安全
### 复制对象不要忘记每个成员
派生类复制时记得也要复制基类
```
Derive::Derive(const Derive& rhs) : Base(rhs) { ... }
Derive& Derive::operator=(const Derive& rhs) {
    Base::operator=(rhs);
    return *this;
}
```
### 以独立语句将newed对象放入智能指针
```
func(shared_ptr<Widget>(new Widget), priority());
```
这是错误的，参数对应的三个操作顺序不一定，priority函数可能抛出异常，导致new操作的指针还没放入shared_ptr，造成资源泄露

### 用non-member、non-friend替换成员函数
*防止降低封装性*

### 若所有参数都需类型转换，使用non-member函数
比如说`operator*`，让它能满足 2*Widget 和 Widget*2

### 考虑写出一个不抛异常的swap函数
std::swap执行拷贝来交换对象，对于用pimpl手法实现的对象，默认的swap操作会缺乏效率。我们不能改变std命名空间内的东西，但可以为空间内的模板制造特化版本。
```
namespace std {
    template <> 
    void swap<Widget> (Widget& lhs, Widget& rhs) {
        ...
    }
}
```
那如果Widget是模板类呢？
```
namespace std {
    template <typename T> 
    void swap<Widget<T>> (Widget<T>& lhs, Widget<T>& rhs) { ... }
    template <typename T> 
    void swap(Widget<T>& lhs, Widget<T>& rhs) { ... }
}
```
第一种对模板函数偏特化，这是不行的。第二种使用重载，添加新的函数到std命名空间，也是不行的。因此，我们不在std命名空间提供重载函数就可以了。

### 处理模板化基类内的名称
派生类不会在模板化基类内寻找继承而来的名称
```
template <typename T>
struct Base {
    void BaseFunc();
};

template <typename T>
struct Derive : Base<T> {
    void DeriveFunc() {
        BaseFunc();         // Error
    }
};
```
解决方法：
1. 调用模板基类函数前加上`this->`
2. 调用模板基类函数前加上`Base<T>::`
3. 派生类使用 `using Base<T>::BaseFunc;`

第二种方法不太可以，如果BaseFunc是虚函数，会失去virtual绑定效果

### 将参数无关的代码抽离templates
为了防止模板实例化导致代码膨胀

### 需要类型转换时为模板定义非成员
需要类型转换则使用non-member函数，但对于模板类来说不行，因为模板实参推导过程中不会考虑隐式类型转换。这时候需要把non-member函数作为模板类的友元函数，这样当模板类实例化，对应的友元模板函数也实例化，匹配参数就没有模板实参推导，可以进行隐式类型转换
```
template <typename T> 
struct Rational {
    friend const Rational operator*(const Rational& lhs, const Rational& rhs) {
        ... 
    }
};
```
必须inline定义友元函数，否则就只是实例化友元模板函数的声明，没有实例化其定义，链接会失败