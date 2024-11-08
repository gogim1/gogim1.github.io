---
title: "策略引擎底层改造：如何解决性能和灵活性问题"
tags: [work]
---

<!--more-->
### 引言

作为集团内部写操作链路的关键一环，策略引擎必须兼顾高性能和灵活性，避免成为上下游的性能瓶颈，同时要满足安全对抗的各种策略编排需求。

本文主要分享安全内部自研的策略引擎底层改造过程，介绍在这个过程中遇到的坑和解决方案，同时还有一些思考和总结，希望对从事相关领域工作的同学能够有所启发或者帮助。

### 背景与挑战

#### 什么是策略引擎

在内容安全领域，策略引擎能够根据一系列复杂的决策规则，识别出业务场景中的内容是否违规，并给出该内容的处置建议。在后续的安全审核环节中，我们会参考策略引擎给出的建议，拦截违规内容，放通安全内容，减少风险的透出。

为了对抗瞬息万变的违规内容变种形式，安全策略方需要及时制定、调整各个场景的策略方案。以往采用硬编码将安全策略部署到服务中的方法，灵活性已经跟不上策略的频繁变动。业界中大多采用专门的 DSL（Domain-Specific Language） 语言来编写策略规则，避免服务的更新发布，从而降低策略迭代的风险，缩短策略部署上线的时间周期。

更进一步的，集团安全内部采用低代码开发的方案：安全策略方通过用户友好的界面来编排规则，而策略引擎根据配置自动生成 DSL 语言。这种方案不需要人工编写 DSL 语言，降低了策略引擎的使用难度。

<center>
  <p align="center">
    <img src="{{ site.url }}/assets/2024-08-02-strategy_engine/pic1.svg" width="60%" height="60%">
  </p>
  <p align="center">图 1 - 策略引擎执行流程图</p>
</center>

#### 遇到的问题

当下形势安全对抗愈演愈烈，一条内容是否违规，我们需要结合更多的信息，根据更复杂的决策规则才能准确判断。安全策略方对于策略引擎规则编排的精细化、规则之间完备的布尔逻辑关系需求日益增长。

然而目前策略引擎灵活性跟不上策略方的需要：由于一些历史原因，策略引擎生成的 DSL 代码和执行 DSL 的流程不能很好地支持算法类策略规则，缺乏算法类规则且关系的编排逻辑，也缺乏非算法类规则的命中结果返回。因此策略引擎改造的目标是要重新设计能够满足需求的 DSL 生成及执行方案。

在集团安全内部，策略引擎管理上百个业务场景的规则，对接着若干上下游服务。这决定了新的 DSL 生成方案要做好存量数据的兼容，新的 DSL 执行方案不能影响到上下游的调用逻辑。而 DSL 执行流程又反过来确定 DSL 的代码形式，环环相扣，无异于戴着镣铐跳舞。

除此之外，策略引擎作为写操作的核心链路，调用量日常峰值可达 1w+ QPS，性能至关重要。如果底层改造能够带来性能的提升，将带来很可观的收益。同样的，新方案引进的细微性能损耗，在海量 QPS 的放大下也会变得十分惊人。目前策略引擎的底层设计仍然有很大的优化空间，新的设计方案需要兼顾性能，满足集团降本增效的要求。

### 方案设计

#### 执行流程

策略引擎为每个业务场景维护一份 DSL Runtime 的对象池。在执行 DSL 时，策略引擎会互斥地从池中取出一个 Runtime 对象，向其注入调用参数，执行得到结果。

目前的实现中，当策略引擎收到调用请求，会抽取出请求中的算法结果，分别注入到 DSL Runtime 对象执行，最后根据每个 DSL 执行的结果综合得出处置建议。

由此可见，多个算法类的策略规则编排天然的难以实现，并且当调用请求中的算法结果越多，策略引擎从池中获取对象的锁争用会越严重，也更容易耗尽对象池。

> 这种实现方式是有历史原因的：我们需要知道调用请求中每个算法结果经过 DSL 执行后的处置建议。将每个参数单独执行后再综合计算结果的方案，既直观又容易实现。

针对上述问题，在新的 DSL 执行方案中，策略引擎收到调用请求，不再将算法结果分别注入到 DSL 执行执行 n 次；而是将算法结果全部注入到 DSL 执行，只用执行 1 次 。这种方案对于参数多的请求，性能提升非常明显:
<center>
  <p align="center">
    <img src="{{ site.url }}/assets/2024-08-02-strategy_engine/pic2.svg" width="60%" height="60%">
  </p>
  <p align="center">图 2 - 原 DSL 执行流程</p>
</center>

<center>
  <p align="center">
    <img src="{{ site.url }}/assets/2024-08-02-strategy_engine/pic3.svg"  width="60%" height="60%">
  </p>
  <p align="center">图 3 - 新 DSL 执行流程</p>
</center>

此外，我们还做了一些性能优化：由于 DSL 与宿主语言存在性能差异，我们将部分计算逻辑通过 FFI（Foreign Function Interface） 迁移到宿主语言执行，而 DSL 只保留必要的逻辑判断。表面上看，两种方案的时间复杂度是一样的，但是由于执行环境的差异，新方案的执行效率会更高。

#### DSL 设计

基于上一小节的执行流程方案，我们重新设计了 DSL 的代码形式。

```golang
// 旧方案
// 省略敏感内容...

// 新方案
// 省略敏感内容...
```

在新的 DSL 中，我们修改了 DSL 注入参数的取值方式，将原来的多次函数调用合并成一次 XXX 调用。这减少了 DSL 代码长度和 IR（Intermediate Representation） 节点数，有助于提高 DSL 执行效率以及降低 DSL Runtime 的内存开销。

<center>
  <p align="center">
    <img src="{{ site.url }}/assets/2024-08-02-strategy_engine/pic4.svg" width="100%" height="100%">
  </p>
  <p align="center">图 4 - 旧方案 IR 图</p>
</center>

<center>
  <p align="center">
    <img src="{{ site.url }}/assets/2024-08-02-strategy_engine/pic5.svg" width="60%" height="60%">
  </p>
  <p align="center">图 5 - 新方案 IR 图</p>
</center>

为了兼容历史需求，我们在 XXX 方法里还额外维护了注入参数的队列，并在生成 DSL 的必要位置中，插入队列处理的相关代码。 通过这种方式，我们可以得到调用请求中每个结果经过 DSL 执行后的处置建议。

> 这里补充个插曲。在开发过程中，我们意外发现 DSL 解释器居然不支持短路求值。感谢这个缺陷，前面的功能才得以实现🙏


#### 代码生成

为了兼容存量数据，策略引擎不直接从前端配置生成 DSL 语句，而是在代码生成环节引入了中间层。在生成 DSL 之前，策略引擎会把前端配置转化成叶节点是配置内容，非叶节点是布尔逻辑运算符的二叉树结构。在二叉树层面处理兼容问题后，再将二叉树转换为 DSL 语言。

由于新方案修改了 DSL 的代码形式和执行流程，导致 DSL 语句对算法参数的取值方式发生了改变。因此我们把算法相关的配置项合并成二叉树的一个特殊节点，在代码生成时做专门处理。

<center>
  <p align="center">[省略敏感内容]</p>
  <p align="center">图 6 - 前端配置示例</p>
</center>

<center>
  <p align="center">[省略敏感内容]</p>
  <p align="center">图 7 - 示例对应的中间层</p>
</center>

#### 依赖库优化

策略引擎基于开源 DSL 解释器，仅仅使用到解释器所提供的一小部份特性。经过一番性能分析排查之后，我们发现开源版本的解释器提供的部分特性比较消耗资源。

基于这种情况，我们 fork 开源代码自行维护，对开源 DSL 解释器的功能进行裁剪。在不影响正常运行的情况下，去除一些运行时默认注入的参数，只保留需要的特性。这种优化方案带来非常好的效果，节省了一半以上的内存开销。

### 具体实践

#### 平滑迁移

策略引擎掌握着数据的”生杀大权“，假如底层改造的过程中引入了故障，导致数据被误伤或漏放，这会带来非常大的风险。策略引擎给出了错误处置建议，在机器自动打击的业务场景中将会带来大规模的用户反馈。因此我们需要一些平滑迁移的方案，避免灰度、回滚时的数据不一致，确保新版本上线万无一失。

在 DSL 生成阶段，我们采用数据双写方式：新增策略会生成新旧方案的 DSL，并写入数据库。服务稳定运行后再逐步清洗，为存量策略批量生成新 DSL 代码；在 DSL 执行阶段，策略引擎以业务场景为维度，逐步灰度上线新方案的功能。只有当该业务场景存在新 DSL 且命中配置的灰度开关时，才会执行新逻辑。

在核心服务上线前，我们会运行若干个空跑任务。核心服务在运行时会复制一份流量给空跑任务，通过对比核心服务旧 DSL 执行结果和空跑任务新 DSL 执行结果，我们可以验证新服务的正确性。

<center>
  <p align="center">[省略敏感内容]</p>
  <p align="center">图 8 - 空跑任务执行结果</p>
</center>

最后单元测试、基准测试也是必不可少的。我们通过单元测试验证了核心功能的正确性，通过基准测试验证了新方案不会带来性能退化。

<center>
  <p align="center">
    <img src="{{ site.url }}/assets/2024-08-02-strategy_engine/pic9.png" width="60%" height="60%">
  </p>
  <p align="center">图 9 - 单元测试执行结果</p>
</center>

#### 性能分析

DSL 代码平均长度减少为原来的 <span style="color:red">84.2%</span>。DSL 执行的 CPU 占用率从晚高峰期的 <span style="color:red">20.4% 降低到 11.2%</span>，性能提升约 <span style="color:red"> 1 </span>倍。内存占用从峰值的 <span style="color:red"> 4.73G 降低到 1.68G</span>，平均减少约 <span style="color:red"> 2/3 </span>的内存占用。预计可减少一半的 pod 数量，每月可节省几千元的服务器费用（具体费用需要根据实际情况而定）。

下面的性能对比图中，折线在横坐标 11:00 左右会有突增的现象。这是因为策略引擎在每天 11 点会全量更新对象池，资源占用量突增在预期之内。

<center>
  <p align="center">
    <img src="{{ site.url }}/assets/2024-08-02-strategy_engine/pic10.svg" width="60%" height="60%">
  </p>
  <p align="center">图 10 - CPU 占用率对比</p>
</center>

<center>
  <p align="center">
    <img src="{{ site.url }}/assets/2024-08-02-strategy_engine/pic11.svg" width="60%" height="60%">
  </p>
  <p align="center">图 11 - 内存占用量对比</p>
</center>

接口平均时延降低，长尾延迟显著减少。

<center>
  <p align="center">
    <img src="{{ site.url }}/assets/2024-08-02-strategy_engine/pic12.svg" width="60%" height="60%">
  </p>
  <p align="center">图 12 - 接口时延对比</p>
</center>

### 后继展望

1. 优化策略引擎的分支预测。我们观察到安全链路写操作中的违规内容占比非常少，基于这个假设，我们可以探索诸如 PGO（Profile-Guided Optimizations） 反馈优化等技术，提高 CPU 的分支预测命中率，这在计算密集型的服务中能够带来不小的性能提升。
2. 探索 DSL 解释器的类型推导技术。目前的 DSL 属于动态语言，缺乏类型约束的能力。类型匹配错误往往需要到运行时 panic 才能被发现。 我们可以探索一些诸如 Unification 算法的类型推导技术，将类型错误及时暴露出来。
3. 重新设计并实现 DSL 解释器的 IR 及后端。目前的 DSL 代码是由配置生成，会产生很多冗余代码。此外解释器采用树遍历 IR 的运行模型，空间局部性较差，性能较低。我们可以重新设计 DSL 的 IR，基于新 IR 探索静态分析技术来优化生成的代码。参考堆虚拟机或寄存器虚拟机来实现解释器后端，以此提高 DSL 执行性能。