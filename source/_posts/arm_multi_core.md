---
title: Arm 多核类型
date: 2022-02-14 15:20:44
tags: arm
---

## SMP
Symmetric Multi-Processing (SMP) 直译是同步多处理，即多个核心进行同步的管理调度，可以共享内存、外设等硬件。通常多个核心运行在一个操作系统中，并由任务调度器统一调度。

## AMP
Asymmetric Multi-processing (AMP) 直译是异步多处理，即多个核心分别承担工作，相互之间隔离内存、外设等硬件。出于安全、时效等设计考虑，多个核心可能独立运行不同的操作系统。

## HMP
Heterogeneous multi-processing (HMP) 是异构的多处理，即将多核以cluster分组，组内多核是微架构相同的核，组间保证指令集架构相同。