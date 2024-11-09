---
title: "An Overview of Query Optimization in Relational Systems"
tags: [database, paper, wip]
---

<!--more-->

> https://dl.acm.org/doi/pdf/10.1145/275487.275492

### 介绍
sql 数据库查询过程有两个关键组件：查询优化器、查询执行引擎。查询执行引擎由一组物理算子构成，物理算子以一个或多个数据流为输入，输出一个数据流。常见的物理算子有 external sort、sequential scan、index scan、nested-loop join 和 sort-merge join。sql 执行的过程可以抽象为一颗物理算子树，也可以称为物理计划。其结构由查询执行引擎确定。查询优化器负责为查询执行引擎提供输入，它接受一颗 sql 语法树，生成高效的执行计划。优化器可以分解成三部分：

* statistics：维护统计信息，用于代价评估
* cost model：基于统计信息，对具体的执行计划评估代价
* plan enumeration：对可能的执行计划进行搜索，对其代价进行评估，选择最优执行计划


### System-R 


### search space 