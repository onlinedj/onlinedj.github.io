---
title: Arm std service 之 DRTM
date: 2023-05-22 15:20:44
tags: DRTM
category: ARM
---

## 前言

最近在重新梳理Arm TrustedFirmware代码，恰逢TF-A拉出v2.9-rc0版本，相较现在很多厂商在用的旧版v1.4(2017.7)，增加了很多功能特性，包括Arm架构特性RME、SVE、SME、TRBE、BRBE、MPAM、PAUTH、MTE、BTI、PAN、AMU、SPE等，以及SPMD/SPMC（SPCI）、DRTM、SDEI、RMMD、MeasureBoot、CCA COT等软件服务。本文将针对DRTM方案及代码实现（待更新）进行深入分析。
<!--more-->
[Change Log](https://trustedfirmware-a.readthedocs.io/en/latest/change-log.html)

## 重要概念

- TCG(trusted computing group) 可信计算组织
- TCB(trusted compute base) 可信计算基
- CRTM(Core Root of Trust for Measurement) BootROM作为可信链的开始，被称为CRTM
- SRTM(Static Root of Trust for Measurement) 静态可信根测量
- DRTM(Dynamic Root of Trust for Measurement) 动态可信根测量
- PCR(Platform Configuration Register）TPM上用于记录平台配置的寄存器（0-15）
- DL event(Dynamic Launch event)
- DCE(DRTM Configuration Environment) DRTM配置环境
- D-CRTM(Dynamic Code Root of Trust for Measurement) 动态代码可信根测量
- DLME(Dynamically Launched Measured Environment) 动态启动的测量的环境

## 背景

SRTM设计方案中，测量结果是在启动过程中被记录到TPM的PCR中，只有SOC重启才能重新记录。如果操作系统运行时进行远程认证，只能获得启动过程的测量结果，如果后续时间，某个阶段的软件被攻破，并不能被测量到。由于Arm平台在桌面、服务器领域的投入较晚，可以直接跳过SRTM，直接开发DRTM。Arm平台在10年前就开始引入TFA固件作为可信固件，实际上可信固件的环境已有，主要是按照DRTM规范实施。

![srtm_boot_flow](/images/srtm_boot_flow.png "srtm_boot_flow")

### 应用案例

windows7及以前的系统，由于没有和SOC vendor做协同，只是单方面增加了TPM（1.2），这类的系统会在开机过程（Bootrom-> BIOS -> UEFI -> bootloader）进行测量，并记录到TPM，系统启动后会通过远程认证的方式对TPM记录的数据进行认证。如果上述早期过程有漏洞，则TPM记录的数据可能在漏洞利用之前，TPM的记录变得不可信。

windows10/11开始推进TPM2.0，并且跟SOC vendor做协同，微软对windows10上的windows-defender-system-guard进行了重构，将原来的SRTM修改为DRTM方案。在不改变原有启动流程的情况下，增加了Secure Launch步骤，这就是本文介绍的DRTM dynamic lauch，即由UEFI触发Secure Launch, 启动一个受保护的固件进行测量，再将测量结果写入TPM

配合组件：secure boot/measured boot/tpm2.0/remote attestation/hypervisor/硬件特权模式/SMMU

支持硬件：intel/amd/qcom(armv8)

![windows_drtm](/images/windows_drtm.png "windows_drtm")

[windows10/11 Secure Launch 1](https://learn.microsoft.com/en-us/windows/security/threat-protection/windows-defender-system-guard/how-hardware-based-root-of-trust-helps-protect-windows)
[windows10/11 Secure Launch 2](https://www.microsoft.com/en-us/security/blog/2020/09/01/force-firmware-code-to-be-measured-and-attested-by-secure-launch-on-windows-10/)


## 概念说明

### DRTM、D-CRTM、DCE、DLME关系

DRTM是TCG组织最早在2013年提出的一种用于MeasureBoot的动态可信根方案，用于解决以前的静态可信根可能产生的攻击面风险，其原理是在不需要重启的条件下，通过动态测量的方式记录测量结果，如果某个阶段的软件被攻破，可以被动态测量到。
Arm在2022年提出了Armv8及后续架构的DRTM规范，主要介绍了DRTM相关的概念，以及在Armv8架构上的解决方案。

下图中DRTM implementation主要包含2个部分：D-CRTM（用于初始化TPM、验证和跳转到DCE）、DCE（DCE用于验证系统状态，测量系统的关键安全属性，测量和跳转到DLME）
protected payload被称为DLME， 后边章节会具体展开。

![drtm_boot_flow](/images/drtm_boot_flow.png "drtm_boot_flow")

DRTM的核心理念是将原来的可信基缩小，排除掉非可信和可扩展的组件，通过一个被保护的负载进行测量和启动。

### Device

外设之类的可以被SMMU保护起来的被称为Device

### Non-Host Platform

系统中不受SMMU保护，并且能不被限制的读写DCE/DLME的被称为non-host platform

[TCG DRTM Spec](https://trustedcomputinggroup.org/wp-content/uploads/TCG_D-RTM_Architecture_v1-0_Published_06172013.pdf)
[Arm DRTM Spec: DEN0113](https://developer.arm.com/documentation/den0113/latest)
[TFA DRTM PPT](https://www.trustedfirmware.org/docs/DRTM_Implementation.pdf)
[TFA DRTM POC](https://trustedfirmware-a.readthedocs.io/en/latest/design_documents/drtm_poc.html)

## 安全约束

可信测量包含了DLME和安全状态，会被写入TPM
DLME处于安全状态：
- 单线程
- 中断和异常关闭
- SMMU保护
- 构建TCB的相关数据
  * 可信的内存布局
  * event log
  * 可信电源管理table或其hash

## Arm平台方案

按照Arm平台应用场景来举例：
D-CRTM : TFA
DCE : TFA
DLME : NS EL2 hypervisor层的独立payload，具体看vendor
（DCE允许在normal world, 但是方案会变复杂）

![dynamic_launch_arm](/images/dynamic_launch_arm.png "dynamic_launch_arm")

### SMC命令功能特性
| Function              | Desc                                                                                                                            |
|-----------------------|---------------------------------------------------------------------------------------------------------------------------------|
| DRTM_VERSION          | Returns the version of the DRTM implementation. See section 3.2.                                                                |
| DRTM_FEATURES         | Allows a client to query the DRTM implementation to determine the supported DRTM capabilities of the platform. See section 3.3. |
| DRTM_DYNAMIC_LAUNCH   | Initiates a dynamic launch. See section 3.4.                                                                                    |
| DRTM_UNPROTECT_MEMORY | Removes the memory protection put in place by the dynamic launch. See section 3.5.                                              |
| DRTM_CLOSE_LOCALITY   | Closes a locality in the TPM. See section 3.6.                                                                                  |
| DRTM_GET_ERROR        | Returns an error code from the previous dynamic launch. See section 3.7.                                                        |
| DRTM_SET_ERROR        | Sets a DRTM error code indicating that a dynamic launch has failed. See section 3.8.                                            |
| DRTM_SET_TCB_HASH     | Used by firmware to record hashes of components of the TCB of the system prior to the dynamic launch. See section 3.9.          |
| DRTM_LOCK_TCB_HASHES  | Used by firmware to prevent additional hashes from being recorded. See section 3.10.                                            |

### SMC使用示例

![smc_example](/images/smc_example.png "smc_example")

### firmware-backend实现

TFA目前是按照firmware-backend实现的

![firmware_backend](/images/firmware_backend.png "firmware_backend")

### hardware-backend实现

![hardware_backend](/images/hardware_backend.png "hardware_backend")

### 与TCG标准的差异

- PCR使用方式不同，以及PCR关联的名词Authorities、Details、DLME.Authority没有使用
- 支持关闭locality, TCG未定义该概念
- TCG中定义的敏感资源的概念未被使用，Armv8使用安全异常等级保护该类资源
- TCG定义的数据结构，只使用了event log这一种
  * DLME data
  * DRTM_PUBLIC_KEY
  * DLME_DESCRIPTOR
  * DLME_ARGUMENTS
  * DLME_TRANSLATION_LIST
- 不支持TCG规范中的DLME_Exit接口

### 测量方法

#### firmware-based measurement

由D-CRTM/DCE来计算测量，然后通过 ==TPM2_PCR_Extend== 接口将HASH发送给TPM

#### TPM-based measurement

由D-CRTM/DCE通过 ==TPM2_PCR_Event== 接口发送要测量的数据给TPM，由TPM自行计算HASH和扩展
(由于TPM的性能较差，如果数据量较大，会很慢)

### Locality 1-4

- Locality 4: Usually associated with the CPU executing microcode. This is used by the D-CRTM to establish the Dynamic RTM.
- Locality 3: Auxiliary components. Use of this is optional and, if used, it is implementation dependent.
- Locality 2: Dynamically Launched OS (Dynamic OS) “runtime” environment.
- Locality 1: An environment for use by the Dynamic OS.
- Locality 0: The Static RTM, its chain of trust and its environment

|Type | Component|
|---------------|--------|
| Locality 4 | DCRTM |
| Locality 3 | DCE |
| Locality 2 | DLME |


### PCR 用途

![pcr_usage](/images/hardware_backend.png "hardware_backend")

### 内存保护

1. DLME和其他补充的images是放在Non-secure内存
2. DCE preamble launch的时候会带入要保护的内存范围，并配置好硬件保护
3. 保护类型：完全保护（通过SMMU阻止所有非安全访问）、区域保护（平台定义的支持部分区域保护的方式）

### Event log

1. 存放在DLME data中
2. 最小64KB
3. 遵循crypto agile event log结构体
4. 被写入TPM的测量动作都要放入event log
5. 每次dynamic launch event要写入event log

![event_type_part1](/images/event_type_part1.png "event_type_part1")
![event_type_part2](/images/event_type_part2.png "event_type_part2")
![event_type_part3](/images/event_type_part3.png "event_type_part3")

## 安全思考

### firmware-based

0. DRTM的目的是在Non-secure state实例化一个更小的可信计算基(DLME)
1. DRTM的安全目标是保证DLME和DLME执行环境的完整性
2. 可信的ACPI tables、可信的mem map
3. 有DMA能力的设备，必须有SMMU, 且配置好DMA保护策略
4. 关闭boot core以外的所有core
5. dynamic launch期间关闭所有Non-secure中断、异常、SDEI events

### hardware-based

0. 数字签名验证DCE，并记录到TPM
1. secure firmware重新加载，会被测量并记录到TPM
2. DRTM配置好DMA保护策略
3. DRTM要么mask安全中断，要么abort DRTM流程
4. D-CRTM检测debug口

## 代码剖析

前面介绍了arm spec上的一些标准信息，接下来可以针对TFA上的实现做一些分析。