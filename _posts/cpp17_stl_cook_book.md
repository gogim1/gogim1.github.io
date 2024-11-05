# C++17 STL Cook Book

标签（空格分隔）： 待整理

---

> https://chenxiaowei.gitbook.io/c-17-stl-cook-book/

## 第三章 迭代器
迭代器的几种类型：双向、前向，随机访问等等。
### 建立可迭代区域
range for的细节：
``` 
auto __begin = std::begin(range);
auto __end = std::end(range);
for (; __begin != __end; ++__begin) {
    auto x = *__begin;
    code_block
}
```
自己的迭代器要能运用于 **range for**，至少需要实现 **operator!=**、**operator++**、**operator*** 。 
### 让自己的迭代器与STL的迭代器兼容
* difference_type: it1- it2结果的类型
* value_type: 迭代器解引用的数据的类型(这里需要注意void类型)
* pointer: 指向元素指针的类型
* reference: 引用元素的类型
* iterator_category: 迭代器属于哪种类型，如 std::forward_iterator_tag

C++17之前， 通过继承自 std::iterator 提供成员 typedef
```
class iterator: public std::iterator<std::input_iterator_tag, ..., ..., ..., ...>
```
C++17中弃用，不推荐。stackoverflow 的做法是自己在类里面 typedef 或自己实现 iterator。
> https://stackoverflow.com/questions/37031805/preparation-for-stditerator-being-deprecated
https://stackoverflow.com/questions/43268146/why-is-stditerator-deprecated

本书的做法是打开 std 命名空间，特化 iterator_traits：
```
namespace std {
    template <>
    struct iterator_traits<num_iterator> {
        using iterator_category = std::forward_iterator_tag;
        using value_type = int;
    };
}
```
### 使用迭代适配器填充通用数据结构
> 大多数情况下，我们想要用数据来填充任何容器，不过数据源和容器却没有通用的接口。这种情况下，我们就需要人工的去编写算法，将相应的数据推入容器中。不过，这会分散我们解决问题的注意力。

> 不同数据结构间的数据传递现在可以只通过一行代码就完成，这要感谢STL中的迭代适配器。本节会展示如何使用迭代适配器。

介绍 std::copy 与 std::front_inserter、std::front_insert_iterator 等等的使用。
### 使用迭代器实现算法
实现一个能够如下使用的类，计算斐波那契数列：
```
for (size_t i : fib_range(10))
    std::cout << i << " ";  // 1 1 2 3 5 8 13 21 34 55 
```
### 使用反向迭代适配器进行迭代
> 当我们想让我们自定义的容器类支持反向迭代，我们不用将所有细节一一实现；我们只需使用 std::make_reverse_iterator 工厂函数，将普通的迭代器包装成反向迭代器即可，背后的操作STL会帮我们完成。

### 使用哨兵终止迭代
### 使用检查过的迭代器自动化检查迭代器代码
> 在vector自增的过程中，会分配新的内存，删除旧的内存，但是迭代器却不知道这个改变。这就意味着，迭代器将会指向旧地址，并且我们不知道这样做会怎样。

使用工具帮忙检查错误
> GUN STL支持一种预处理宏_GLIBCXX_DEBUG，其会激活STL中对健壮性检查的代码。这会让程序变慢，不过更容易找到Bug。我们可以通过-D_GLIBCXX_DEBUG编译选项来启用这些代码，或者在代码的最开始加上这个宏。

> Microsoft Visual C++ 编译器可以通过/D_ITERATOR_DEBUG_LEVEL=1启用检查。

> LLVM/clang实现的STL也有调试标识，其目的是为了调试STL代码，而非用户的代码。对于用户的代码的调试，我们会使用不同的选项来调试，向clang编译器传入-fsanitize=address -fsanitize=undefined

### 构建zip迭代适配器
boost 的 zip_iterator、std::valarray、c++范围库

## 第四章 Lambda表达式