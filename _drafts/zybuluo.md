---
title:  作业部落Cpp
date: 2021-05-28
---


## 静态变量初始化
静态变量存储在虚拟地址空间的数据段和bss段，C语言中其在代码执行之前初始化，属于编译期初始化。而C++中由于引入对象，对象生成必须调用构造函数，因此C++规定全局或局部静态对象当且仅当对象首次用到时进行构造 

《C++设计新思维》说，使用编译期常量加以初始化的 static 变量`static int i = 100;`是在程序装载时初始化的。

----------

> https://www.cnblogs.com/wanyuanchun/p/4041080.html
在类内部初始化静态成员`class A { static int i = 1; };`必须为**字面值常量**类型的 **constexpr** (常量表达式) 

*不符合条件的只能类外初始化？*
```
static vector<double> vec(vecSize);  // error，不是字面值
static double rate = 6.5;   // error，不是常量
constexpr static const double rate = 6.5  // ok
```

## C++语言的15个晦涩特性
> https://www.cnblogs.com/lanxuezaipiao/p/3501078.html
http://madebyevan.com/obscure-cpp-features/

----------

ptr[3]其实只是*(ptr + 3)的缩写，与用*(3 + ptr)是等价的，因此反过来3[ptr]也合法。

----------

```
// 1)通过int(x)实例化的类型int变量，int y = 3;等价于int(y) = 3;
// 2)一个函数声明，返回一个int值并有一个参数，参数是一个名为x的int型变量
int bar(int(x));
```
C++标准要求的是第二种解释，即使第一种解释看起来更直观。程序员可以通过包围括号中变量的初始值来消除歧义`int bar((int(x)));`

**TODO 待测试**

----------

> 标记符and, and_eq, bitand, bitor, compl, not, not_eq, or, or_eq, xor, xor_eq, <%, %>, <: 和 :>都可以用来代替我们常用的&&, &=, &, |, ~, !, !=, ||, |=, ^, ^=, {, }, [ 和 ]。在键盘上缺乏必要的符号时你可以使用这些运算标记符来代替。

----------
C++11允许成员函数在对象的值类型上进行重载，依据 this 对象是左值还是右值选择重载：
```
struct Foo {
  void foo() & { std::cout << "lvalue" << std::endl; }
  void foo() && { std::cout << "rvalue" << std::endl; }
};
 
int main() {
  Foo foo;
  foo.foo(); // Prints "lvalue"
  Foo().foo(); // Prints "rvalue"
}
```

----------

指向成员的指针
```
struct Test {
  int num;
  void func() {}
};

int Test::*ptr_num = &Test::num;
void (Test::*ptr_func)() = &Test::func;

Test t;
Test *pt = new Test;
(t.*ptr_func)();
(pt->*ptr_func)();
t.*ptr_num = 1;
pt->*ptr_num = 2;
```
成员函数指针与普通函数指针是不同的，在成员函数指针和普通函数指针之间casting是无效的。
thiscall 
**TODO**

----------

模板参数可以是函数
```
template <int (*F)(int)>
int memoize(int x) {
  return F(x);
}
```

