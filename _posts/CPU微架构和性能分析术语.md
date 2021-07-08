---
layout: post
title: CPU性能分析中的术语和指标
date: 2021-06-25
author: Youngh
categories: 计算机系统
tags: 多发射 动态调度 保留站 寄存器重命名 精确异常
---

简单介绍一下 Linux 性能分析工具 `perf` 中所使用的的基本术语和指标。

## 4.1 Retired vs. Executed Instruction

现代处理器通常执行比程序流所需更多的指令，这主要是因为其中一些是推测性执行的。CPU 处理的指令可以 excuted 但不一定 retired。因此，我们通常可以预测 excuted 指令的数量高于 retired 指令的数量。

`perf` 可以收集 retired 指令的数量：

```apl
$ perf stat -e instructions ./hello
  4,564,863		instructions
```

## 4.2 CPU利用率

CPU 在某个时间段内的忙碌时间所占比例。

如果 CPU 利用率低，通常意味着应用程序性能不佳。但是高 CPU 使用率也未必总是好的。这表明系统正在做一些工作，但并没有确切说明它在做什么。在多线程上下文中，线程也可以在等待资源处理时自旋。

`perf` 可以计算 CPU 利用率：

```apl
$ perf stat -e task-clock ./hello
  1.867875      task-clock (msec)	# 0.878 CPUs utilized
```

## 4.3 CPI & IPC

两个很重要的指标，可用于评估硬件和软件效率，二者互为倒数。

`perf` 可以计算 IPC：

```apl
$ perf stat -e cycles,instructions ./hello
  4,460,351      cycles                                                      
  4,531,686      instructions		# 1.02  insn per cycle
```

## 4.4 UOPs（micro-ops）

具有 x86 架构的微处理器将复杂的 CISC-like 指令转换为简单的 RISC-like 微指令（缩写为 µops 或 uops）。 这样做的主要优点是可以乱序执行 µops。

```apl
1| 	ADD		EAX, EBX
2| 	ADD		EAX, [MEM1]
3| 	ADD 	[MEM1], EAX
```

指令之间的关系及其拆分为微指令的方式 CPU 而异。

除了将复杂的 CISC-like 指令拆分成 RISC-like 微指令外，微指令还可以融合。现代 Intel CPU 有两种类型的融合：

* 微融合。 微指令来自相同的机器指令。微融合只能应用于两种类型的组合：内存写操作 和 读-修改操作。

  ```apl
  # Read the memory location [ESI] and add it to EAX
  # Two uops are fused into one at the decoding step.
  add eax, [esi]
  ```

* 宏融合。微指令来自不同的机器指令。在某些情况下，译码器可以将算术或逻辑指令与后续条件跳转指令融合为单个计算和分支微指令。

  ```apl
  # Two uops from DEC and JNZ instructions are fused into one
  .loop:
  	dec rdi
  	jnz .loop
  ```

融合操作节省了从 decode 到 retire 的流水线所有阶段的带宽。

融合以后会共用 ROB 中的一个条目，这样 ROB 的容量就增加了。虽然共用同一个 ROB 条目，但还是需要由两个不同的执行单元来完成两个操作。融合条目会被分派到两个不同的执行端口，但最终还是作为一个单元 retire。

`perf` 可以计算 issued, executed, 和 retired 微指令的数量：

```apl
$ perf stat -e uops_issued.any,uops_executed.thread,uops_retired.all ./hello
  5,210,147      uops_issued.any
  5,360,703      uops_retired.all
```

## 4.5 流水线槽

处理一个微指令所需的硬件资源。

下图展示了一个 CPU 的执行流水线，它每个周期可以处理 4 个 uops。在图中的六个连续周期中，只有一半的可用插槽被利用。

<img src="E:\File\TyporaImg\image-20210708143339728.png" alt="image-20210708143339728" style="zoom:40%;" />

## 4.6 内核周期和参考周期

大多数现代 CPU，包括 Intel 和 AMD CPU，都没有固定的运行频率，而是实现了动态频率缩放。在英特尔的 CPU 中，这项技术称为 Turbo Boost，在 AMD 的处理器中称为 Turbo Core。它允许 CPU 动态地增加和降低其频率：**减小频率降低了功耗但牺牲了性能，增加频率提高了性能但牺牲了功耗**。

**内核周期** 是以 CPU 内核运行的实际时钟频率计算出的时钟周期，而 **参考周期** 直接统计外部时钟周期。

## 4.7 缓存失效

根据自顶向下的微架构分析，指令缓存未命中会导致前端停顿，因此被归为前端问题，而数据缓存未命中会导致后端停顿，因此被归为后端问题。

缓存的命中与失效此时都可以通过 `perf`工具统计出来。

## 4.8 分支预测错误

分支错误预测时，CPU 需要撤销它最近所做的所有推测性工作，这通常涉及 10 到 20 个时钟周期的惩罚。

`perf` 可以查看分支预测错误次数。

