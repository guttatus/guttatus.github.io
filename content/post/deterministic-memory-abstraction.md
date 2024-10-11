+++
author = "guttatus"
title = "Deterministic Memory Abstraction and Supporting Multicore System Architecture"
date = "2024-10-10"
description = "多核实时系统中的确定性内存抽象"
tags = [
    "Arch",
    "multicore",
    "real-time system",
]
categories = [
    "Reading Notes  ",
]
toc = true
autonumbering = true
+++
# Deterministic Memory Abstraction and Supporting Multicore System Architecture
## 简介
多核处理器的时间可预测性差一直是实时系统社区的一个长期挑战。因此，这篇论文提出了一种新的整体资源管理方法，该方法提出了一种新的内存抽象，称之为**确定性内存**。确定性内存的关键特征是，平台 —— 操作系统和硬件 —— 保证小且严格绑定的最坏情况内存访问时序。相比之下，将传统的内存抽象称为best-effort内存，其中只能实现高度悲观的最坏情况边界。论文介绍了确定性内存感知操作系统和架构设计，包括os级页面分配器、硬件级Cache和DRAM控制器设计。

## 与传统方法的区别
多核处理器在实时系统领域面临的关键问题是由于核间共享资源争用导致的核间任务干扰。面对这一问题，有两种常见的解决策略：
1. 在任务或核心之间划分共享资源以实现空间隔离
2. 在访问共享资源时应用可分析的仲裁方案（例如，时分多址）以实现时间隔离

传统方法大多是以Task或Core为粒度对共享资源进行划分。

论文提出了一种以页面为粒度的共享资源划分方法，可以在保证确定性的同时，提高共享资源利用率，减少性能损失。

## 具体做法
在Page Table Entry中增加DM bit用于指示内存类型。当DM bit为1时，内存类型为确定性内存；当DM bit为0时，内存类型为best-effort内存。

![Deterministic Memory Abstraction](/img/posts/deterministic/deterministic-memory-abstraction.png)

### Cache
基本方法是，只对确定性内存访问使用way分区，best-effect内存访问可以使用当前不持有确定性缓存行的所有缓存行。
![Deterministic memory-aware cache management](/img/posts/deterministic/cache.png)
当插入一个新的缓存行时，如果请求的内存访问是针对确定性内存的，那么从Core的way分区中选择victim Cacheline例如，上图中Core 0的way 0 和 way 1）。如果请求内存访问是best-effect内存，则从不保存确定性Cacheline的way中选择victim Cacheline。（例如，上图的set 0中，除way 2之外的所有Cacheline都是best-effect的Cacheline，在set 1中，只有way 4是的best-effect Cacheline。

算法首先尝试驱逐 best-effort Cacheline（如果存在这样的Cacheline）。如果不存在（即所有Cacheline都是确定性的），则选择其中一条确定性Cacheline作为受害者。如果请求best-effert的内存，它会驱逐一个best-effert高速缓存Cacheline，但不驱逐任何确定性的Cacheline。这样，虽然分区的确定性Cacheline与分区的已分配核心以外的任何访问完全隔离，但分区的任何未充分利用的Cacheline仍可被所有内核用作尽力而为的Cacheline。

为了以可预测的方式在任何给定的分区上只保留最少数量的确定性缓存行，缓存控制器提供了一种特殊的硬件机制，可以清除所有确定性Cacheline的DM位，有效地将它们转换为best-effert的Cacheline。内核的 OS 调度程序在每个上下文切换上都使用这种机制，以便当前任务可以驱逐先前任务的确定性Cachelines。当再次访问任务的确定性转为best-effert的Cache-line并且它们仍然存在于缓存中时，它们将被简单地重新标记为确定性，而无需从内存中重新加载。

### DRAM Controller
首先，操作系统主动控制为页面帧分配的DRAM bank。具体来说，操作系统为每个Core保留少量的bank，用作内核的确定性内存，而其余的bank用于所有内核的best-effect内存。当操作系统分配应用程序任务的内存页时，应在内核私有DRAM bank上分配任务的确定性内存页，以消除DRAM bank级内核间干扰，而best-effert的内存页则在共享的DRAM bank上分配。
![Deterministic memory-aware memory controller architecture and scheduling algorithm.](/img/posts/deterministic/dram.png)
其次，内存控制器（MC）实现了一种两级调度算法，该算法首先优先考虑确定性内存请求，而不是尽力而为内存的请求。对于确定性内存请求，使用循环调度策略，因为它提供了更强的时间可预测性；对于best-effert内存，使用先就绪先到先得 （FR-FCFS） 策略，因为它提供了高平均吞吐量。

但是需要注意，严格确定性内存请求的优先级可能会无限期地耗best-effect的内存请求。由于我们假设best-effect内存存在悲观的最坏情况边界，因此在存在best-effect内存请求的情况下，我们限制了确定性内存请求的最大连续处理次数，以实现确定性内存的严格限制的最坏情况时序，同时实现best-effect内存实现悲观但仍然有界的最坏情况时序。