---
title: "DCR框架中Message类的分析"
tags: [code, cpp]
---

DCR是实验室内部使用的分布式计算框架，本文讨论其中自动注册工厂模式的实现方法。

<!--more-->

假设有多个线程分别运行着不同的组件，它们之间通过消息`Message`传输数据。现在消息种类有很多，为了降低耦合度，你可能更倾向于使用消息工厂类`MessageFactory`，而不是手工显性地创建消息对象。这样只需要调用`MessageFactory::build`函数，带上你想创建消息类的标识符，就能获得相对应的消息对象：

```
struct Message { ... };

struct MessageA : public Message { ... };

struct MessageFactory {
    Message* build(const string &name) {
        if (name == "MessageA")
            return new MessageA;
    }
};
```

这样的话存在一个问题，如果需要创建新的消息类`MessageB`，那你不得不找到`MessageFactory::build`函数的位置，添上新的条件判断语句，这也会导致`MessageFactory`所在的编译单元得重新编译一遍。如果你的代码是以库的形式发布出去，此时用户可能会创建新的消息类，那么目前的`MessageFactory::build`函数就没办法继续工作了。

这时候你可能会想到用`map`来替代`build`函数中的一长串条件判断语句，同时要求用户创建新的消息类后，必须调用`MessageFactory::registerMessage`进行注册。紧接着，你会写出这样的代码：
```
struct MessageFactory {
    void registerMessage(const string &name) {
        builders_ = ?;
    }

    Message* build(const string &name) {
        return builders_[name];
    }

    std::map<string, ?> builders_;
};
```
示例代码中`?`的部分应该是什么呢？`registerMessage`如何获得用户新创建的消息类？`builders_`的值应该是某种能够创建新对象的东西？DCR框架采用的是**模板**+**多态**的解决方法。

为了能够创建新的对象，DCR使用多态为`builders_`的值中加入了一层抽象：
```
struct MessageFactory {
    ...

    struct AbstractMessageBuilder {
        virtual Message* build() = 0;
    };    
    
    Message* build(const string &name) {
        return builders_[name]->build();
    }

    std::map<string, AbstractMessageBuilder*> builders_;
};
```
不同的消息类只需创建各自的`AbstractMessageBuilder`子类并加入`builders_`中，`MessageFactory::build`便能够通过调用虚函数`AbstractMessageBuilder::build`，返回指向不同消息类的`Message`指针。而这也是`MessageFactory::registerMessage`的实现方式。

刚刚我们讲到，`MessageFactory::registerMessage`代码中需要创建`AbstractMessageBuilder`子类，这样问题来了，如果不同的消息类都调用了`MessageFactory::registerMessage`，如何创建不同的`AbstractMessageBuilder`子类呢？

结合之前DCR是使用模板来获得用户新创建的消息类，此时我们意识到，可以使用模板特化来创建不同的`AbstractMessageBuilder`子类：
```

struct MessageFactory {
    ...
    template<typename MessageType>
    strcut MessageBuilder : public AbstractMessageBuilder {
        virtual Message* build() {
            return new MessageType;
        }
    };

    template<typename MessageType>
    void registerMessage(const string &name) {
        MessageBuilder<MessageType> *builder = new MessageBuilder<MessageType>();
        builders_[name] = builder;
    }
    ...
};
```
好了，现在万事大吉，用户只需要在使用`MessageFactory::build`前调用`MessageFactory::registerMessage`就可以正常使用这个工厂类了：
```
// DCR代码
MessageFactory message_factory;

// 用户代码
class UserMessage : public Message { ... };

int main() {
    message_factory.registerMessage<UserMessage>("UserMessage");
    ...
    Message* message_ptr = message_factory.build("UserMessage");

}
```

看起来还行，但是稍显不足，用户可能忘记调用`registerMessage`函数。要是`registerMessage`这个函数能够自动调用就好了，最好是在用户定义新的消息类时，能够插入一些代码来完成这部分工作。你可能会这么写：
```
// DCR代码
#define DCR_REGISTER_MESSAGE(MessageType) \
message_factory.registerMessage<MessageType>(#MessageType);

// 用户代码
struct UserMessage : public Message { 
    UserMessage() {
        DCR_REGISTER_MESSAGE(UserMessage);
    }
 };

```
这样要求用户在其新消息类型的构造函数中，必须使用`DCR_REGISTER_MESSAGE`宏。但是仔细一想，这是不对的。如果用户不实例化`UserMessage`，那岂不是没有注册消息？我们希望新的消息类型是无论有无实例化，无论有几个对象，都只注册一次。

仔细想想，`static`成员变量好像能够派上用场，因为同个类的所有对象都共享相同的成员变量。我们只需将`DCR_REGISTER_MESSAGE`宏的内容放入`static`成员变量的构造函数中不就可以了，接着再继续用宏进行简化代码:

```
// DCR代码
template<typename MessageType>
struct DoRegister {
    DoRegister(string name) {
        message_factory->registerMessage<MessageType>(name);
    }
};
#define DCR_MESSAGE_DECLARE_STATIC(MessageType)  \
static DoRegister<MessageType> do_register_;

#define DCR_REGISTER_MESSAGE(MessageType) \
DoRegister<MessageType> MessageType::do_register_(#MessageType);

// 用户代码
struct UserMessage : public Message { 
    DCR_MESSAGE_DECLARE_STATIC(UserMessage);
    ...
 };

DCR_REGISTER_MESSAGE(UserMessage);
```

这是一种可行的解决方法。而DCR的实现方法比较繁琐，它不是把`static`成员变量放在用户定义的新消息类中，而是使用模板类`RegisterMessageFactory`，用户每次定义新的消息类`UserMessage`都会额外实例化一个含有`static`成员变量的`RegisterMessageFactory<UserMessage>`类，精简后如下：
```
// DCR代码
template<typename MessageType>
struct RegisterMessageFactory; 

#define DCR_REGISTER_MESSAGE(MessageType)                                   \
template<>                                                                  \
struct RegisterMessageFactory<MessageType> {                                \
    struct DoRegister {                                                     \
        DoRegister(const string &name) {                                    \
            message_factory->registerMessage<MessageType>(name);            \
        }                                                                   \
    };                                                                      \
    static const DoRegister do_register_;                                   \
};                                                                          \
const RegisterMessageFactory<MessageType>::DoRegister                       \
    RegisterMessageFactory<MessageType>::do_register_(#MessageType);  


// 用户代码
struct UserMessage : public Message { ... };

DCR_REGISTER_MESSAGE(UserMessage);

```
这样子，用户定义完新的消息类后，只需要使用`DCR_REGISTER_MESSAGE`宏就可以将新消息类注册到工厂类`MessageFactory`中了。这就完了吗？还没有！

我们知道，不同编译单元内的`non-local static`变量的初始化顺序是不确定的。DCR框架中的消息工厂对象`message_factory`，与用户代码中`RegisterMessageFactory`模板类的静态成员变量`do_register_`是位于不同的编译单元，这意味着有可能`do_register_`先初始化，调用了还未初始化的`message_factory`。

因此我们可以使用单例模式，把`message_factory`作为局部静态变量：
```
MessageFactory* globalMessageFactory() {
    static MessageFactory message_factory;
    return &message_factory;
}
```
`do_register_`使用`globalMessageFactory()`来获得`message_factory`，确保`message_factory`先于`do_register_`被初始化：
```
#define DCR_REGISTER_MESSAGE(MessageType)                                   \
template<>                                                                  \
struct RegisterMessageFactory<MessageType> {                                \
    struct DoRegister {                                                     \
        DoRegister(const string &name) {                                    \
            globalMessageFactory()->registerMessage<MessageType>(name);     \
        }                                                                   \
    };                                                                      \
    static const DoRegister do_register_;                                   \
};                                                                          \
const RegisterMessageFactory<MessageType>::DoRegister                       \
    RegisterMessageFactory<MessageType>::do_register_(#MessageType);  

```

到此，DCR框架中的`Message`类就剖析完毕。想更进一步？如果你觉得模板+宏的写法不够新时代，想要把宏都用template和constexpr换掉的话。我们可以利用C++的新特性和GCC的扩展，写出下面代码：
```
template <typename T>
constexpr auto getTypeName() noexcept {
    std::string name = __PRETTY_FUNCTION__, prefix, suffix;
    prefix = "constexpr auto getTypeName() [with T = ";
    suffix = "]";
    name = name.substr(prefix.size());
    name = name.substr(0, name.size() - suffix.size());
    return name;
}

template <typename MessageType>
struct RegisterMessage : public Message {
    struct DoRegister {
        DoRegister() {
            globalMessageFactory()->registerMessage<MessageType>(
                getTypeName<MessageType>()
            );
        }
    };
    RegisterMessage() {  do_register_;  }
           
    static inline DoRegister do_register_{};
};

```
这样的话，用户定义新的消息类也很简单：
```
// 用户代码
struct UserMessage : public RegisterMessage<UserMessage> { 
    UserMessage() : RegisterMessage<UserMessage>() {}
    ... 
};

```

值得注意的是，模板类中的静态成员变量是惰性构造，因此需要手动触发`do_register_`的初始化。至此，修改后的DCR框架代码如下：
```
template <typename T>
constexpr auto getTypeName() noexcept {
    std::string name = __PRETTY_FUNCTION__, prefix, suffix;
    prefix = "constexpr auto getTypeName() [with T = ";
    suffix = "]";
    name = name.substr(prefix.size());
    name = name.substr(0, name.size() - suffix.size());
    return name;
}

struct Message {};

struct MessageFactory {
    struct AbstractMessageBuilder {
        virtual Message* build() = 0;
    };    
    
    template<typename MessageType>
    struct MessageBuilder : public AbstractMessageBuilder {
        virtual Message* build() {
            return new MessageType;
        }
    };

    template<typename MessageType>
    void registerMessage(const string &name) {
        MessageBuilder<MessageType> *builder = new MessageBuilder<MessageType>();
        builders_[name] = builder;
    }

    Message* build(const string &name) {
        return builders_[name]->build();
    }

    std::map<string, AbstractMessageBuilder*> builders_;
};


MessageFactory* globalMessageFactory() {
    static MessageFactory message_factory;
    return &message_factory;
}


template <typename MessageType>
struct RegisterMessage : public Message {
    struct DoRegister {
        DoRegister() {
            globalMessageFactory()->registerMessage<MessageType>(
                getTypeName<MessageType>()
            );
        }
    };
    RegisterMessage() {  
        do_register_; 
    }
           
    static inline DoRegister do_register_{};
};
```

到这里，`Message`类已经被剖析得清清楚楚了。此外，DCR框架中的`MessageHandler`也是消息机制的一个重要组成部分，不过原理大致相同，也是**模板**+**多态**，可以试着自己分析一下。

