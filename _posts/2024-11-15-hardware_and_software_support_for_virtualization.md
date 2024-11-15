---
title: "Hardware and Software Support for Virtualization"
tags: [virtualization, note, wip]
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
    - 时间复用。同个物理资源在时间上被调度。比如操作系统调度进程

- A virtualmachine is an abstraction of a complete compute environment through the com
bined virtualization of the processor, memory, and I/O components of a computer.
- The hypervisor is a specialized piece of system software that manages and runs virtual ma
chines.
- The virtual machine monitor (VMM) refers to the portion of the hypervisor that focuses
 on the CPU and memory virtualization.