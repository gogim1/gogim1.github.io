---
title: "effective三部曲"
tags: [cpp, note]
---



<!--more-->

### More Effective C++
#### pointer和reference的区别
* 引用不能为空，指针可能为空。指针使用前判断是否为空指针
* 引用指向最初的对象，指针可以重新赋值
* operator[]应该返回一个引用

#### 使用C++转型操作符
dynamic_cast用来将指向基类的指针或引用，转型为指向派生类的指针或引用。如果转型失败，会返回null指针（转型对象是指针）或抛出异常（转型对象是引用）。

reinterpret_cast的转换结果与平台相关，不具可移植性。最常用的用途是转换函数指针类型。
```
typedef void (*FuncPtr)();
FuncPtr funcPtrArray[10];
int doSomtthing();
funcPtrArray[0] = reinterpret_cast<FuncPtr>(&doSomtthing);
```
#### 不以多态方式处理数组
```
class Base { ... };
class Derive : public Base { ... };
void func(Base array[]) { ... }

Derive deriveArray[10];
func(deriveArray);  // Error
```
取数组的元素是按照基类的大小来对指针进行计算，然而派生类一般都比基类大，会导致计算错误

*想不到为什么会这么用*

#### 非必要不提供默认构造函数
不提供默认构造函数，产生一个对应类型的数组会报错，这时可以采用placement new。可能会用不了一些模板容器，需要容器作者谨慎地设计。作为基类的话，要求派生类知道如何构造基类。

*辩证地看待*

#### 避免类型转换函数
使用explicit。避免定义类型转换函数。

#### 区别i++/++i的重载
*了解即可*

#### 不重载&&、||和，操作符
*常识*

#### 区别operator new和new operator
new operator是关键字，执行两个操作：分配内存、构造对象。其中分配内存调用的就是operator new。

#### 用析构函数避免资源泄漏
*RAII*
#### 在构造函数内防止资源泄漏
*使用RAII保护资源*
#### 禁止异常流出析构函数之外
两种情况下会调用析构函数：对象正常被销毁、对象被异常处理机制销毁。如果异常离开析构函数，而此时也处于另一个异常中，C++会调用terminate函数结束程序。禁止异常流出析构函数能避免terminate函数在异常传播过程的栈展开被调用；协助确保析构函数完成应该完成的所有工作。
#### 了解抛出异常、传递参数、调用虚函数的差异
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

#### 以传引用方式捕捉异常
可以抛出指针，但是要防止catch捕捉到指向不存在对象的指针。

catch-by-value除了有复制两次的问题，也会有派生类被切割的问题。抛出派生类，捕捉到的是基类，会调用基类的虚函数

catch-by-reference只复制一下，没有派生类被切割的问题。抛出派生类，捕捉到的是派生类

#### 明智运用exception specifications
不将模板与exception specifications混合使用，因为不知道模板参数类型可能抛出什么异常。

如果需要调用回调函数，可以为回调函数指针类型的声明加上exception specifications。这样就只能注册不抛出异常的回调函数
```
typedef void (*callBackPtr)() throw();
```
#### 了解异常处理的成本
*避免使用异常*

#### 虚函数表和虚函数指针
有虚函数的类会有一个虚函数指针，指向虚函数表。虚函数表中包含的是指向该类虚函数的指针。如果Derive继承Base，那么Derive会重新定义某些继承的虚函数，修改虚函数表中指针的指向，并加上新的虚函数指针。

虚函数表位置在哪？暴力方法是在每一个需要虚函数表的目标文件都产生虚函数表副本，再由链接器去除重复的副本。另一种方法是将虚函数表放在第一个non-inline、non-pure虚函数定义的目标文件中。

调用虚函数本身不构成性能瓶颈，而在与虚函数无法inline。inline意味着在编译期就将函数本体替换调用动作，这与virtual语义相反。

#### 将类的构造器和非成员函数虚化
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

### Effective C++
#### 不在构造和析构过程中调用virtual函数
基类构造期间调用虚函数，总是调用本身的虚函数。因为基类比派生类更早构造。
#### 令operator=返回*this的引用
为了连锁赋值
#### operator=中记得处理自我赋值
也要注意异常安全
#### 复制对象不要忘记每个成员
派生类复制时记得也要复制基类
```
Derive::Derive(const Derive& rhs) : Base(rhs) { ... }
Derive& Derive::operator=(const Derive& rhs) {
    Base::operator=(rhs);
    return *this;
}
```
#### 以独立语句将newed对象放入智能指针
```
func(shared_ptr<Widget>(new Widget), priority());
```
这是错误的，参数对应的三个操作顺序不一定，priority函数可能抛出异常，导致new操作的指针还没放入shared_ptr，造成资源泄露

#### 用non-member、non-friend替换成员函数
*防止降低封装性*

#### 若所有参数都需类型转换，使用non-member函数
比如说`operator*`，让它能满足 2*Widget 和 Widget*2

#### 考虑写出一个不抛异常的swap函数
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

#### 处理模板化基类内的名称
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

#### 将参数无关的代码抽离templates
为了防止模板实例化导致代码膨胀

#### 需要类型转换时为模板定义非成员
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