---
title: Arm std service 之 SDEI
date: 2023-05-18 15:20:44
tags: SDEI
category: ARM
---

## 前言
最近在重新梳理Arm TrustedFirmware代码，恰逢TF-A拉出v2.9-rc0版本，相较现在很多厂商在用的旧版v1.4(2017.7)，增加了很多功能特性，包括Arm架构特性RME、SVE、SME、TRBE、BRBE、MPAM、PAUTH、MTE、BTI、PAN、AMU、SPE等，以及SPMD/SPMC（SPCI）、DRTM、SDEI、RMMD、MeasureBoot、CCA COT等软件服务。本文将针对SDEI标准及代码实现（待更新）进行深入分析。
<!--more-->
[Change Log](https://trustedfirmware-a.readthedocs.io/en/latest/change-log.html)

## SDEI
Software Delegated Exception interface (SDEI) 是Arm定义的软件代理异常处理接口，用于非安全世界注册handler到TFA来接收底层系统事件通知。

[Arm Specification: DEN0054](https://developer.arm.com/documentation/den0054/c/?lang=en)
[TFA SDEI section](https://trustedfirmware-a.readthedocs.io/en/latest/components/sdei.html)

## 场景及用途

### 场景

- 系统错误处理
- 软件看门狗计时器
- 内和调试
- 采样分析

### 用途

1. 订阅系统事件
2. 屏蔽系统事件
3. 系统事件核间迁移
4. 事件处理过程添加或者移除核
5. 转换现有中断为SDEI事件
6. 产生软件事件

## 软件模型与定义

### 模型

提升关键中断的优先级的方式有很多种，SDEI利用将系统事件陷入更高异常等级的方式来实现提升优先级，因为高异常等级永远会抢占低异常等级的执行

| Exception Level | Application         |
|-----------------|---------------------|
| EL0             | Linux App           |
| EL1             | Linux OS            |
| EL2             | Hypervisor Firmware |
| EL3             | Trusted Firmware    |


### 定义

- Client
  低异常等级的被成为Client
- Dispatcher
  高异常等级的被成为Dispatcher

* Event
  -  source        来源通常是硬件异常或者中断，当然也可以是软件中断（SGI）
  -  type          Private(PPI)/Shared(SPI)
  -  definition    platform event(静态定义)/bound event（client动态注册）

### SDEI接口和异常等级

#### 层级关系及分类

- Physical SDEI  TFA作为dispatcher，nonsecure作为client
- Virtual SDEI   hypervisor作为dispatcher，guest os作为client

> 注意：secure侧作为client是implementation defined，此处不多介绍

![sdei_instances](/images/sdei_instances.png "sdei_instances")


#### 关系对应

下表总结了当前存在的SDEI实例关系

![sdei_el_relation](/images/sdei_el_relation.png "sdei_el_relation")

## SDEI系统概览

### 提升事件优先级

- CPU PSTSATE中断mask用于防止中断路由到当前EL
- GIC 中断路由控制寄存器用于将中断通知到更高EL

### Event优先级分类

- Normal priority    动态注册的只能是normal类，比如watchdog timer事件
- Critical priority  只有platform event可以是critical类，抢占所有normal priority event，但是同级不能抢占，比如错误处理事件

#### 抢占策略

![sdei_preempt](/images/sdei_preempt.png "sdei_preempt")

#### 嵌套层级

由于只有2级，同级不能抢占，所以最多嵌套2层

### Event 编号

![sdei_event_num](/images/sdei_event_num.png "sdei_event_num")

## SDEI 接口

| 接口                             | 功能                        |
|--------------------------------|---------------------------|
| SDEI_VERSION                   | 返回版本号                     |
| SDEI_EVENT_REGISTER            | 注册事件                      |
| SDEI_EVENT_ENABLE              | 使能事件                      |
| SDEI_EVENT_DISABLE             | 去使能事件                     |
| SDEI_EVENT_CONTEXT             | 返回事件触发时client的X0-X17寄存器值  |
| SDEI_EVENT_COMPLETE            | 完成事件                      |
| SDEI_EVENT_COMPLETE_AND_RESUME | 完成事件，恢复client运行           |
| SDEI_EVENT_UNREGISTER          | 取消注册事件                    |
| SDEI_EVENT_STATUS              | 事件状态（注册？使能？运行？）           |
| SDEI_EVENT_GET_INFO            | 事件信息（类型、可触发、优先级、路由模式、AFF） |
| SDEI_EVENT_ROUTING_SET         | 设置路由信息                    |
| SDEI_PE_MASK                   | 屏蔽PE接收事件                  |
| SDEI_PE_UNMASK                 | 取消屏蔽PE接收事件                |
| SDEI_INTERRUPT_BIND            | 中断绑定事件                    |
| SDEI_INTERRUPT_RELEASE         | 中断绑定释放                    |
| SDEI_EVENT_SIGNAL              | 触发事件到PE                   |
| SDEI_FEATURES                  | SDEI支持的特性查询               |
| SDEI_PRIVATE_RESET             | 重置当前调用PE的所有私有SDEI事件       |
| SDEI_SHARED_RESET              | 重置当前调用PE的所有共享SDEI事件       |

## 开发者概述

### 事件状态机

> 私有事件是PE私有状态
> 共享事件是全局状态，多PE共享，同一时刻只能有1个实例处理该共享事件
> bond event必须先bind，再执行下述状态机流程
> bond event必须取消全部注册才能release

![sdei_event_state](/images/sdei_event_state.png "sdei_event_state")

### 事件注册和处理

#### Physical SDEI event

中断触发的SDEI事件的绑定、注册与处理
![sdei_bind_reg_handle_full](/images/sdei_bind_reg_handle_full.png "sdei_bind_reg_handle_full")

显式调用的SDEI事件的注册、处理流程
![sdei_explicit_event_handle](/images/sdei_explicit_event_handle.png "sdei_explicit_event_handle")

#### Virtual SDEI 事件

带有hypervisor的事件注册
![sdei_event_reg_with_hypervisor](/images/sdei_event_reg_with_hypervisor.png "sdei_event_reg_with_hypervisor")

带有hypervisor的事件处理
![sdei_event_handle_with_hypervisor](/images/sdei_event_handle_with_hypervisor.png "sdei_event_handle_with_hypervisor")

### 电源管理与事件处理

#### CPU_ON

![sdei_cpu_on](/images/sdei_cpu_on.png "sdei_cpu_on")

#### CPU_OFF

![sdei_cpu_off](/images/sdei_cpu_off.png "sdei_cpu_off")

#### CPU_SUSPEND

![sdei_cpu_sus](/images/sdei_cpu_sus.png "sdei_cpu_sus")

### 代码剖析

前面介绍了arm spec上的一些标准信息，接下来可以针对TFA上的实现做一些分析。