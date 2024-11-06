---
title: "C++设计新思维"
tags: [cpp, wip, notes]
---

<!--more-->

> 修订历史
> - 2019.12.29 创建笔记
> - 2024.11.06 移出私密

## 技术
* 局部类，可以在函数中定义class

## Singleton实作技术
* 与一般单纯的全局变量相比，Singleton的特性
* 用以支持Singletons的C++基本手法
* 如何更好地厉行“Singleton的唯一性”
* 后的访问动作
* 实现Singleton对象的生命期高阶管理方案
* 多线程相关问题

### 静态数据 + 静态函数 != Singleton
使用静态成员函数 + 静态成员变量的缺点：**静态函数不能成为虚函数、初始化和析构工作变得困难**

### 用以支持 Singletons 的一些 C++ 基本手法
Singleton 类中使用的静态成员是 Singleton* pInstance_，而不是 Singleton Instance_。**pInstance_是在程序被装载时完成静态初始化 ，Instance_ 是动态初始化，由执行期的构造函数构造。但 C++ 未定义不同编译单元的初始化顺序，可能返回一个尚未构造的对象。**

### 实施“Singleton”的唯一性
* 默认构造函数、复制构造函数、赋值操作符声明为 private
* `static Singleton& Instance();` 让 Instance() 传回引用而非指针，防止使用者 delete
* 析构函数声明为 private，防止使用者意外删除

### 摧毁Singleton
* 内存泄漏、资源泄漏
* atexit()函数
* `Singleton& Singleton::Instance() { static Singleton obj; return obj; }`
> 《Effective C++》条款26