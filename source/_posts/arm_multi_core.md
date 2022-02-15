---
title: Arm 多核类型
date: 2022-02-14 15:20:44
tags: arm
---

## SMP
Symmetric Multi-Processing (SMP) 直译是同步多处理，即多个核心进行同步的管理调度，可以共享内存、外设等硬件。通常是多个完全相同的核心运行在一个操作系统中，并由任务调度器统一调度。

## AMP
Asymmetric Multi-processing (AMP) 直译是异步多处理，即多个核心分别承担工作，相互之间隔离内存、外设等硬件。出于安全、时效等设计考虑，多个核心可能独立运行不同的操作系统。

## HMP
Heterogeneous multi-processing (HMP) 是异质的多处理，即将多核以cluster分组，组内多核是微架构相同的核，组间只需保证指令集架构相同。通常HMP多个核心运行在统一的调度器中。

## CACHE
SMP最简单，多个完全相同的核组合在一起，共享L2 cache,可以保证cache一致性。   
AMP因为多个核心是独立运行,不需要关心cache一致性。     
HMP可以理解为多组SMP，需要保证组间cache一致性。   
在2013引入big.LITTLE技术中，采用cache coherent interconnect,在2017年引入第二代DynamIQ big.LITTLE技术中，增加了shared L3
![DSU-110 arch](/images/dynamIQ110.svg "DSU-110")