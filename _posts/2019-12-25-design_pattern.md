---
title: "设计模式"
tags: [notes]
---

something about design patterns...

<!--more-->

## 简单工厂模式
```
class Product {
    virtual void fun();
};
class Product1 : public Product { ... };
class Product2 : public Product { ... };
class Factory {
    Product* createProduct(auto xx) {
        switch (xx) { ... }
    }
};
```

## 工厂方法模式
> 把简单工厂的内部逻辑判断移到了客户端代码来进行

工厂模式包括简单工厂、工厂方法、抽象工厂这3种细分模式。其中，简单工厂和工厂方法比较常用，抽象工厂的应用场景比较特殊，所以很少用到，不是我们学习的重点。

工厂模式用来创建不同但是相关类型的对象（继承同一父类或者接口的一组子类），由给定的参数来决定创建哪种类型的对象。实际上，如果创建对象的逻辑并不复杂，那我们直接通过new来创建对象就可以了，不需要使用工厂模式。当创建逻辑比较复杂，是一个“大工程”的时候，我们就考虑使用工厂模式，封装对象的创建过程，将对象的创建和使用相分离。

当每个对象的创建逻辑都比较简单的时候，我推荐使用简单工厂模式，将多个对象的创建逻辑放到一个工厂类中。当每个对象的创建逻辑都比较复杂的时候，为了避免设计一个过于庞大的工厂类，我们推荐使用工厂方法模式，将创建逻辑拆分得更细，每个对象的创建逻辑独立到各自的工厂类中。

详细点说，工厂模式的作用有下面4个，这也是判断要不要使用工厂模式最本质的参考标准。

* 封装变化：创建逻辑有可能变化，封装成工厂类之后，创建逻辑的变更对调用者透明。
* 代码复用：创建代码抽离到独立的工厂类之后可以复用。
* 隔离复杂性：封装复杂的创建逻辑，调用者无需了解如何创建对象。
* 控制复杂度：将创建代码抽离出来，让原本的函数或类职责更单一，代码更简洁。

除此之外，我们还讲了工厂模式一个非常经典的应用场景：依赖注入框架，比如Spring IOC、Google Guice，它用来集中创建、组装、管理对象，跟具体业务代码解耦，让程序员聚焦在业务代码的开发上。DI框架已经成为了我们平时开发的必备框架，在专栏中，我还带你实现了一个简单的DI框架，你可以再回过头去看看。

```
class Product {
    virtual void fun();
};
class Product1 : public Product { ... };
class Product2 : public Product { ... };

class Factory {
    virtual Product* createProduct() = 0;
};
class Factory1 : public Factory { 
    Product* createProduct() { return new Product1(); }
};
class Factory2 : public Factory { ... };

Factory* f = new Factory1();
Product* product1 = f->createProduct();
```
## 抽象工厂模式
工厂模式有了多种 Product。
```
class ProductA {
    virtual void fun();
};
class ProductA1 : public ProductA { ... };

class ProductB {
    virtual void fun();
};
class ProductB1 : public ProductB { ... };

class Factory {
    virtual ProductA* createProductA() = 0;
    virtual ProductB* createProductB() = 0;
};
class Factory1 : public Factory { 
    Product* createProductA() { return new ProductA1(); }
    Product* createProductB() { return new ProductB1(); }
};
class Factory2 : public Factory { ... };
```

## 策略模式
> 定义一系列的算法，把它们一个个封装起来，并且使它们可相互替换。

普通的多态
```
class Strategy {
    virtual void algorithmInterface() = 0;
};
class Strategy1 : public Strategy {
    void algorithmInterface() { ... }
};
class Strategy2 : public Strategy { ... };

class Context {
    Strategy* s;
    void contextInterface()  { s->algorithmInterface(); }
};
```

## 装饰模式
> 动态地给一个对象添加一些额外的职责。将核心职责和装饰功能分开。

装饰器模式主要解决继承关系过于复杂的问题，通过组合来替代继承，给原始类添加增强功能。这也是判断是否该用装饰器模式的一个重要的依据。除此之外，装饰器模式还有一个特点，那就是可以对原始类嵌套使用多个装饰器。为了满足这样的需求，在设计的时候，装饰器类需要跟原始类继承相同的抽象类或者接口。
```
class Component {
    virtual void operation();
};
class ConcreteComponent { ... };

class Decorator : public Component {
    Component c;
    void operation() {  c.operation();  };
};
ConcreteDecorator1 : public Decorator {
    void operation() {  
        ...
        Decorator::operation();   // 我觉得c.operation();也行，因为构建子类时把父类赋予子类的c了
        ...
    };
};
ConcreteDecorator2 : public Decorator { ... };

ConcreteComponent a;
ConcreteDecorator1 b(a);
ConcreteDecorator2 c(b);
c.operation();
```

## 代理模式
代理模式在不改变原始类接口的条件下，为原始类定义一个代理类，主要目的是控制访问，而非加强功能，这是它跟装饰器模式最大的不同。一般情况下，我们让代理类和原始类实现同样的接口。但是，如果原始类并没有定义接口，并且原始类代码并不是我们开发维护的。在这种情况下，我们可以通过让代理类继承原始类的方法来实现代理模式。

静态代理需要针对每个类都创建一个代理类，并且每个代理类中的代码都有点像模板式的“重复”代码，增加了维护成本和开发成本。对于静态代理存在的问题，我们可以通过动态代理来解决。我们不事先为每个原始类编写代理类，而是在运行的时候动态地创建原始类对应的代理类，然后在系统中用代理类替换掉原始类。

代理模式常用在业务系统中开发一些非功能性需求，比如：监控、统计、鉴权、限流、事务、幂等、日志。我们将这些附加功能与业务功能解耦，放到代理类统一处理，让程序员只需要关注业务方面的开发。除此之外，代理模式还可以用在RPC、缓存等应用场景中。
```
class Subject {
    virtual void request() = 0;
};
class RealSubject : public Subject {
    void request() { ... }
};
class Proxy : public Subject {
    Subject s;
    void request() { 
        s.request();
    }
};
```

## 原型模式
用原型实例指定创建对象的种类，并且通过拷贝这些原型创建新的对象。

> 就是实现拷贝构造函数和赋值运算符而已。

如果对象的创建成本比较大，而同一个类的不同对象之间差别不大（大部分字段都相同），在这种情况下，我们可以利用对已有对象（原型）进行复制（或者叫拷贝）的方式，来创建新对象，以达到节省创建时间的目的。这种基于原型来创建对象的方式就叫作原型模式。

原型模式有两种实现方法，深拷贝和浅拷贝。浅拷贝只会复制对象中基本数据类型数据和引用对象的内存地址，不会递归地复制引用对象，以及引用对象的引用对象……而深拷贝得到的是一份完完全全独立的对象。所以，深拷贝比起浅拷贝来说，更加耗时，更加耗内存空间。

如果要拷贝的对象是不可变对象，浅拷贝共享不可变对象是没问题的，但对于可变对象来说，浅拷贝得到的对象和原始对象会共享部分数据，就有可能出现数据被修改的风险，也就变得复杂多了。操作非常耗时的情况下，我们比较推荐使用浅拷贝，否则，没有充分的理由，不要为了一点点的性能提升而使用浅拷贝。
```
class Prototype {
    virtual Prototype* clone() = 0;
};
class ConcretePrototype : public Prototype { ... };
```

## 访问者模式
> 目的是把处理从数据结构分离出来。适用于有比较稳定的数据结构、易于变化的算法。

结构稳定，算法多变，所以转变思路，从 **一个结构对应多个算法** 变成 **一个算法对应多个结构**。
```
class MutableBase {  // Visitor, 易于变化的算法
    virtual void visitImmutable1(Immutable1*) = 0;
    virtual void visitImmutable2(Immutable2*) = 0;
};
class Mutable1 : public MutableBase {
    void visitImmutable1(Immutable1*);
    void visitImmutable2(Immutable2*);
};
class Mutable2 : public MutableBase { ... };

class ImmutableBase {  // Element, 比较稳定的数据结构
    virtual void Accept(MutableBase*) = 0;
};
class Immutable1 : public ImmutableBase {
    void Accept(MutableBase* p) {
        p->visitImmutable1(this);
    }
};
class Immutable2 : public ImmutableBase { ... };

vector<ImmutableBase> v{...};  // 用法
v[xx].Accept(xx);
```
### 非侵入式多基类访问器
> https://zhuanlan.zhihu.com/p/112003741

## 组合模式
> 组合模式使得用户对单个对象和组合对象的使用具有一致性。

感觉是很普通的多态

```
class Component
{
    virtual void fun();
};
class Leaf : public Component { ... };  // 重载处理
class Composite : public Component { 
    vector<Component> v;
    ...                                 // 重载处理
};
```

## 模板方法模式
> 把不变行为搬移到基类，去除子类的重复代码。

在基类写好逻辑框架，其中变化的部分用虚函数替代。子类重载虚函数。
```
class Base {
    void codeTemplate() {    // 基类写好逻辑框架
        ...         
        fun1();
        fun2();
        ...
    }
    virtual void fun1();    // 变化的部分
    virtual void fun1();
};
class Derive1 : public Base { 
    virtual void fun1();    // 子类重载虚函数
    virtual void fun2();
};
class Derive2 : public Base { ... };
```
## 状态模式
> 目的是消除庞大的条件分支语句。通过定义新的子类可以很容易地增加新的状态和转换

定义一个Context类和多个State的子类。Context类里有State变量和设置State的函数。State子类进行判断及更改Context类状态。
```
class Context {
    State state;
    void setState(State) { ... }
    void request() {  state.handle(this);  }
};

class State {
    virtual void handle(Context*) = 0;
};
class State1 : public State {
    void handle(Context* c) {
        ...                     // 对Context的某一变量进行判断，如true则执行相应动作
        c->setState(State2);     // 如false，则切换状态再进行判断，以此类推
        c->request();
    }    
};
class State2 : public State { ... };
```
## 观察者模式
> 定义了一种一对多的依赖关系，让多个观测者同时监听某一个主题对象。这个主体对象在状态改变时，会通知所有观察者。

个人观点，在A类与B类互相需要对方的数据时使用，一对多的通知不是重点。把A类与B类都抽象出基类。
```
class Subject {
    list<Observer> l;
    virtual void notify() = 0;
};
class Subject1 : public Subject {
    auto data;      // Observer类想知道的数据
    void notify() {  for_each(l.begin(), l.end(), [&](Observer& o){  o.update(); }); }
};

class Observer {
    virtual void update() = 0;
};
class observer : public Observer {
    Subject* s;  
    void update() { ... }  // update时读取s.data获得数据
};
```
C++实现委托
TODO
## 备忘录模式
> 在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态。这样以后就可将该对象恢复到原先保存的状态。

把要保存的细节封装在Memento类里，感觉Caretaker不必要。
```
class Originator {
    ...                             // 需要保存的数据
    Memento save() { ... }          // 数据在返回的Memento中
    void recovery(Memento) { ... }  
};
class Memento { ... };              // 需要保存的数据
class Caretaker {
    Memento m;
    ...                             // Memento相应的set、get函数
};
```
## 中介者模式
> 用一个中介对象来封装一系列的对象交互。中介者使各对象不需要显式地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。

```
class Mediator {
    virtual void send() = 0; 
};
class Mediator1 : public Mediator {
    void send(Colleague* c) { ... }  // 判断c的类别，进行不同处理
};

class Colleague {
    Mediator* m;
};
class Colleague1 : public Colleague {
    void send() {  m->send(this);  }
};
class Colleague2 : public Colleague { ... };
```
## 迭代器模式
> 提供一种方法顺序访问一个聚合对象中各个元素，而又不需要暴露该对象的内部表示。

```
class Iterator {
    virtual auto first() = 0;
    ...  // next()、isDone()、...
};
class ConcreteIterator : public Iterator {
    Aggregate* a;
    ConcreteIterator(Aggregate*);
    auto first() {  return a[0];  }
    ... // next()、isDone()、...
};

class Aggregate {
    virtual Iterator createIterator() = 0;
};
class ConcreteAggregate : public Aggregate {
    Iterator createIterator() {   return ConcreteIterator(this);  }
};
```
## 解释器模式
感觉就是普通的多态

## 命令模式
> 将一个请求封装为一个对象，从而使你可用不同的请求对客户进行参数化；对请求排队或记录请求日志，以及支持可撤销的操作。

个人看法是为了把注册和执行分开，感觉并不是很实用。
```
class Receiver {
    void action() { ... }
};

class Command {
    Receiver* r;
    virtual void execute() = 0;
};
class Command1 : public Command {
    void execute() {  r->action();  }
};
class Command1 : public Command { ... };

class Invoker {
    auto container;     // auto可以是Command*、vector<Command>...
    void invoke() {  container.execute(); };
};

Receiver r; ... Command1 c1; ... Invoker i; ... // 注册
...
i.invoke();                                     // 执行
```
从时序的角度来看，在一般编程任务中，如果想执行一个动作，先将某个对象、该对象的某个成员函数、该函数的引数组装成一个调用：`window.Resize(0,0,200,100);`。“启动这样一个调用”的时刻和“收集这个调用所需元素”的时刻，在概念上是无法区分的。

## 职责链模式
> 使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，直到有一个对象处理它为止

感觉和状态模式差不多，都是把 switch 语句给分解成多个类。不同点在于状态模式的分支跳跃由作者写在子类，职责链的分支跳跃由使用者指定。
```
class Handler {
    Handler successor;
    void setSuccessor(Handle*) { ... }
    virtual void handleRequest() = 0;
};
class Handler1 : public Handler {
    void handleRequest(auto r) {
        ...                             // 进行判断，true则处理请求
        successor.handleRequest(r);     // false则进行下一个判断
    };
};
class Handler2 : public Handler { ... };

Handler1 h1; Handler2 h2;   
h1.setSuccessor(&h2);       // 分支跳跃由使用者指定
h1.handleRequest(xx);
```
## 享元模式
> 运用共享技术有效地支持大量细粒度的对象。

目的是减少存储开销，用较少的共享对象取代很多组对象。
```
class FlyWeight { ... };
class ConcreteFlyWeight : public FlyWeight { ... };

class FlyWeightFactory {
    auto container;                     // 可以用哈希表或vector实现，目的是找出已经存在的对象再利用
    FlyWeight* getFlyWeight() { ... };  // 如果container存在对象则返回，否则实例化再返回
};
```
## 外观模式
> 将子系统中的一组接口提供一个一致的界面，外观模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。

就是用一个类把很多其他的类给包装起来
```
class SubSystem1 {  void fun1();  };
class SubSystem2 {  void fun2();  };
class Facade {
    SubSystem1 s1; SubSystem2 s2;
    void fun() {  
        s1.fun1(); 
        s2.fun2(); 
    }
};
```
## 桥接模式
> 感觉只是 组合关系(has-a)+多态

桥接模式的代码实现非常简单，但是理解起来稍微有点难度，并且应用场景也比较局限，所以，相对来说，桥接模式在实际的项目中并没有那么常用，你只需要简单了解，见到能认识就可以了，并不是我们学习的重点。

桥接模式有两种理解方式。第一种理解方式是“将抽象和实现解耦，让它们能独立开发”。这种理解方式比较特别，应用场景也不多。另一种理解方式更加简单，等同于“组合优于继承”设计原则，这种理解方式更加通用，应用场景比较多。不管是哪种理解方式，它们的代码结构都是相同的，都是一种类之间的组合关系。

对于第一种理解方式，弄懂定义中“抽象”和“实现”两个概念，是理解它的关键。定义中的“抽象”，指的并非“抽象类”或“接口”，而是被抽象出来的一套“类库”，它只包含骨架代码，真正的业务逻辑需要委派给定义中的“实现”来完成。而定义中的“实现”，也并非“接口的实现类”，而是的一套独立的“类库”。“抽象”和“实现”独立开发，通过对象之间的组合关系组装在一起。
```
class Implementor {
    virtual void operation() = 0;
};
class Implementor1 : public Implementor {
    void operation() { ... }
};

class Abstraction {
    Implementor* i;
    virtual void operation() = 0;
};
class Abstraction1 : public Abstraction {
    void operation() {  i->operation();  }
};
```
## 单例模式
单例模式用来创建全局唯一的对象。一个类只允许创建一个对象（或者叫实例），那这个类就是一个单例类，这种设计模式就叫作单例模式。单例有几种经典的实现方式，它们分别是：饿汉式、懒汉式、双重检测、静态内部类、枚举。

尽管单例是一个很常用的设计模式，在实际的开发中，我们也确实经常用到它，但是，有些人认为单例是一种反模式（anti-pattern），并不推荐使用，主要的理由有以下几点：

* 单例对OOP特性的支持不友好
* 单例会隐藏类之间的依赖关系
* 单例对代码的扩展性不友好
* 单例对代码的可测试性不友好
* 单例不支持有参数的构造函数

那有什么替代单例的解决方案呢？如果要完全解决这些问题，我们可能要从根上寻找其他方式来实现全局唯一类。比如，通过工厂模式、IOC容器来保证全局唯一性。

有人把单例当作反模式，主张杜绝在项目中使用。我个人觉得这有点极端。模式本身没有对错，关键看你怎么用。如果单例类并没有后续扩展的需求，并且不依赖外部系统，那设计成单例类就没有太大问题。对于一些全局类，我们在其他地方new的话，还要在类之间传来传去，不如直接做成单例类，使用起来简洁方便。

除此之外，我们还讲到了进程唯一单例、线程唯一单例、集群唯一单例、多例等扩展知识点，这一部分在实际的开发中并不会被用到，但是可以扩展你的思路、锻炼你的逻辑思维。这里我就不带你回顾了，你可以自己回忆一下。
```
class Singleton {
    static Singleton* s = nullptr;
    Singleton() = delete;
    static Singleton* getInstance() { ... }  // 如果s为nullptr则创建
};
```
## 建造者模式
> 和外观模式差不多，都是有一个类把复杂的东西进行封装。 

建造者模式用来创建复杂对象，可以通过设置不同的可选参数，“定制化”地创建不同的对象。建造者模式的原理和实现比较简单，重点是掌握应用场景，避免过度使用。

如果一个类中有很多属性，为了避免构造函数的参数列表过长，影响代码的可读性和易用性，我们可以通过构造函数配合set()方法来解决。但是，如果存在下面情况中的任意一种，我们就要考虑使用建造者模式了。

* 我们把类的必填属性放到构造函数中，强制创建对象的时候就设置。如果必填的属性有很多，把这些必填属性都放到构造函数中设置，那构造函数就又会出现参数列表很长的问题。如果我们把必填属性通过set()方法设置，那校验这些必填属性是否已经填写的逻辑就无处安放了。
* 如果类的属性之间有一定的依赖关系或者约束条件，我们继续使用构造函数配合set()方法的设计思路，那这些依赖关系或约束条件的校验逻辑就无处安放了。
* 如果我们希望创建不可变对象，也就是说，对象在创建好之后，就不能再修改内部的属性值，要实现这个功能，我们就不能在类中暴露set()方法。构造函数配合set()方法来设置属性值的方式就不适用了。

```
class Builder {
    virtual void fun1() {}
    virtual void fun2() {}
    ...
};
class ConcreteBuilder : public Builder {
    ...     // 重载虚函数
};
class Director {
    Builder b;
    void create() {  b.fun1(); b.fun2();  }
};
```

## 适配器模式
### 对象适配器
> 感觉就是中间层加一个类进行封装

代理模式、装饰器模式提供的都是跟原始类相同的接口，而适配器提供跟原始类不同的接口。适配器模式是用来做适配的，它将不兼容的接口转换为可兼容的接口，让原本由于接口不兼容而不能一起工作的类可以一起工作。适配器模式有两种实现方式：类适配器和对象适配器。其中，类适配器使用继承关系来实现，对象适配器使用组合关系来实现。

适配器模式是一种事后的补救策略，用来补救设计上的缺陷。应用这种模式算是“无奈之举”。如果在设计初期，我们就能规避接口不兼容的问题，那这种模式就无用武之地了。在实际的开发中，什么情况下才会出现接口不兼容呢？我总结下了下面这5种场景：

* 封装有缺陷的接口设计
* 统一多个类的接口设计
* 替换依赖的外部系统
* 兼容老版本接口
* 适配不同格式的数据

```
class Adaptee {
    void specificRequest() { ... }
};
class Target {
    virtual void request();
};

class Adapter : public Target {
    Adaptee a;
    void request() { 
        a.specificRequest();
    }
};
```
### 类适配器
TODO

## Policy based design 
> https://blog.csdn.net/u011173875/article/details/17490807

```
template <class CPU, class DISPLAY, class BATTERY,
          template<class C> class NUMCorePolicy = QuadCorePolicy,
          template<class D> class DisplayPolicy = LCDDisplayPolicy,
          template<class B> class BatteryPolicy = ReplacableBatteryPolicy >
class SmartPhone : public NUMCorePolicy<CPU>
                 , public DisplayPolicy<DISPLAY>
                 , public BatteryPolicy<BATTERY>
{ ... };
```
```
template <typename OutputPolicy, typename LanguagePolicy>
class HelloWorld : private OutputPolicy, private LanguagePolicy
{
    using OutputPolicy::print;
    using LanguagePolicy::message;
public:
    void run() const {
        print(message());
    }
};
 
class OutputPolicyWriteToCout
{
protected:
    template<typename MessageType>
    void print(MessageType const &message) const {
        std::cout << message << std::endl;
    }
};
 
class LanguagePolicyEnglish
{
protected:
    std::string message() const  {
        return "Hello, World!";
    }
};

 
int main() {
    typedef HelloWorld<OutputPolicyWriteToCout, LanguagePolicyEnglish> HelloWorldEnglish;
 
    HelloWorldEnglish hello_world;
    hello_world.run(); 
}
```


## 奇异递归模板模式 (Curiously Recurring Template Pattern, CRTP)
> https://xr1s.me/2018/05/10/brief-introduction-to-crtp/、https://zhuanlan.zhihu.com/p/54945314

所谓的 CRTP ，就是基类作为模板类，派生类在继承基类的时候，传入自己作为模板参数。CRTP 真正的用武之地，是在模板类需要访问派生的类的成员（变量或函数）的时候，它可以引用它的派生类，也就可以访问派生类的成员。

```
template <typename Derive>
struct Quack {
  void quack() {
    std::cout << static_cast<Derive*>(this)->name << "quack!\n";
  }
};

template <typename Derive>
struct MuteQuack {
  void quack() {
    std::cout << static_cast<Derive*>(this)->name << " did not say any thing!\n";
  }
};

struct MallardDuck: Quack<MallardDuck> {
  const char *name;
  MallardDuck(const char *name): name{name} {
  }
};
```

鸭子叫的多态例子中，编译期就可以确定下来，运行时不需要再修改，不需要在运行时构造函数中绑定行为，那么 CRTP 就可以派上用场。

子类也可以是模版，如下：

``` 
template <typename Derive>
struct Fly {
  void fly() {
    std::cout << static_cast<Derive *>(this)->name << " flies!";
  }
};

// A same class Quack implementation as above.

template <template <typename> class QuackBehavior, template <typename> class FlyBehavior>
struct Duck
    : QuackBehavior<Duck<QuackBehavior, FlyBehavior>>
    , FlyBehavior<Duck<QuackBehavior, FlyBehavior>> {
  const char *name;
  Duck(const char *name): name{name} {
  }
};

int main() {
  Duck<Quack, Fly> xris{"Xris"};
  xris.quack();
  xris.fly();
}
```
固定的几个模板参数就显得很笨拙，使用可变模板参数：
```
template <template <typename ...> class ...Impl>
struct Duck: public Impl<Duck<Impl...>>... {
  const char *name;
  Duck(const char *name)
      : name{name}, Impl<Duck<Impl...>>{*this}... {
  }
};
```