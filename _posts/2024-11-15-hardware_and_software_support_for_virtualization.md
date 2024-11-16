---
title: "Hardware and Software Support for Virtualization"
tags: [virtualization, notes, wip]
---

<!--more-->

> 修订历史
> - 2024.11.15 创建笔记

## 定义
### 虚拟化
> **Virtualization** is the application of the **layering** principle through **enforced modularity**,
 whereby the exposed virtual resource is identical to the underlying physical resource being
 virtualized.

虚拟化增加中间层，通过模块化保证隔离。

以 RAID 为例，RAID 作为中间层，兼容块设备接口。对上层应用（文件系统）无感知。

因此广义的虚拟化并不局限虚拟机。

- 计算机体系结构中的虚拟化：被 MMU 管理的虚拟内存。因为 MMU 兼容字节寻址的方式，所以启用 MMU，指令在虚拟内存上工作，关闭则在物理内存上工作。

- 操作系统中的虚拟化：本身作为中间层，面向应用程序 expose 计算机资源（CPU、内存、IO）

- IO 子系统中的虚拟化：被虚拟化的是块寻址的扇区，RAID 控制器、存储阵列、SSD 做兼容，为操作系统服务

虚拟化由三种方式实现：
- 复用。虚拟化成多种资源。
    - 空间复用。物理资源被分割成多个虚拟实体。比如操作系统复用物理内存
    - 时间复用。同个物理资源在时间上被调度。比如操作系统调度进程对 CPU 复用、上下文切换对寄存器复用
- 聚合。把多个物理资源虚拟化成一个资源。比如 RAID 控制器将多个磁盘聚合成一个单一的卷
- 模拟（emulation） 软件通过间接层暴露出虚拟资源，即使物理资源不存在。比如内存和磁盘可以互相模拟：内存使用 DRAM 模拟磁盘，磁盘通过虚拟内存的分页来模拟内存

### 虚拟机
> A **virtual machine** is an abstraction of a complete compute environment through the com
bined virtualization of the processor, memory, and I/O components of a computer.

- 基于语言的虚拟机（language-based virtual machines）不在本书讨论范围
- 轻量级虚拟机（lightweight virtual machines）软硬件隔离，使得应用直接在处理器上运行。比如 docker
- 系统级虚拟机（system-level virtual machines）提供完整的、隔离的计算机系统环境。本书的重点。可分成两类
    - hypervisor。指令直接在 cpu 上执行，或者 trap-and-emulate
    - machine simulator。用户级应用程序实现，模拟执行

### Hypervisor
>  the **hypervisor** is a specialized piece of system software that manages and runs virtual machines.

hypervisor 在虚拟机之间对物理资源进行分配和调度，在计算机上应用分层原则，确保：
- 等效性。虚拟机与底层计算机等效
- 安全性。虚拟机之间、虚拟机与 hypervisor 彼此隔离
- 性能。模拟性能只能是轻微下降。这里与 machine simulator 区分开，machine simulator 模拟成本高，性能损耗大

### Type-1 and Type-2 Hypervisors
hypervisors 继续分类：
- type-1。裸金属上运行，如 VMware ESX Server、Xen、Microsoft Hyper-V
- type-2。在 host OS 上运行。如 VMware Workstation、VMware Fusion、KVM、Microsoft VirtualPC、Parallels、Oracle VirtualBox

### A SKETCH HYPERVISOR: MULTIPLEXING AND EMULATION
*没讲什么重要的*
### NAMES FOR MEMORY
为避免名词混淆，这里定义
- guest-physical memory。虚拟机暴露的内存抽象
- host-physical memory。底层资源

### APPROACHES TO VIRTUALIZATION AND PARAVIRTUALIZATION
早期的虚拟化方式：
- Full (software) virtualization。执行前翻译客户指令
- Hardware Virtualization (HVM)。直接执行
- Paravirtualization。看重简单性和整体效率，不完全兼容

### 优势
- 一台机器运行多个操作系统
- 满足每个应用一台服务器的要求
- 快速部署
- 安全。审查客户操作系统行为等
- 高可用
- 资源调度
- 云计算

## The Popek/Goldberg Theorem
验证是否可以使用复用 VMM 来虚拟化给定 ISA 的理论。防止体系结构的设计方案不支持虚拟化

### 模型
## 参考
- https://zhuanlan.zhihu.com/p/186286059
