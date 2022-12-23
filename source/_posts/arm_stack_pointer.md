---
title: Arm 栈指针漫谈
date: 2022-02-14 15:20:44
tags: arm
---

## 前言
本文主要介绍了栈相关的技术点， 通过这些点的填充，让大家对栈有一个全面的认识。

## Arm通用寄存器说明

arm32 包含16个通用寄存器（r0-r15）和 2个特殊寄存器CPSR和SPSR。
r0-r12用于通用功能

r13 通常用来做栈指针(stack pointer), 简称SP

r14 通常用来做链接寄存器(link register)，用于保存子流程的返回地址，简称LR

r15 通常用来做程序计数器（program counter）, 用于保存当前要执行的程序地址，简称PC

*下表中带后缀的为banked register，也就是在对应的模式下有独立的寄存器*
![aarch32 registers](/images/aarch32_registers.png)

根据AAPCS32规范的定义, 对通用寄存器的使用做了进一步限定
其中，

r0-r3 作为输入参数

r4-r11 作为变量寄存器使用

根据平台定义，r9 还可以作为static base/ thread register

r11 可以用作栈帧(frame pointer)使用，简称FP

r12 是用作intra process scratch register，简称IP
![aapcs32 registers](/images/aapcs32_regs.png)

关于这些通用寄存器的别名，可以参考下图

![predeclared register names](/images/predeclared_regs.png)

把两者结合起来看，总结如下图

![predeclared register match](/images/apcs_predeclare_match.png)

## 栈和栈指针
对于执行一段程序而言，我们需要以下几个部分：
text/data/bss/heap/stack

![program layout](/images/program_memory_layout.jpg)

其中栈是一段连续的内存空间，栈的使用从高地址往低地址，先进后出，栈指针存储了当前栈顶的位置

#### 对于SP的约束条件
- stack_limit/stack_top < SP < stack_base
- SP必须4字节对齐

## 栈帧和栈帧指针
栈帧可以理解成函数所有局部变量压栈组合形成的一个数据结构，通常会把LR+FP放到栈顶

栈帧指针存储了父函数的栈顶位置，通过栈帧指针，形成了一个栈帧链表, 利用栈帧指针可以快速定位到函数的栈顶

![stack frame](/images/stack_frame.png)

## 异常处理
在各类异常、中断的处理中，由于切换了上下文，我们在执行异常处理代码之前，必须保存通用寄存器到栈中，否则可能无法恢复之前的上下文。

可以看到SP在各个模式下都有一个banked寄存器，这样就方便我们在不同上下文使用不同的栈内存。
比如收到一个irq中断，cpu切换到irq mode， sp默认使用的是sp_irq

## 引用链接

[aarch32_registers] https://developer.arm.com/documentation/dui0801/k/Overview-of-AArch32-state/Registers-in-AArch32-state?lang=en

[AAPCS32] https://github.com/ARM-software/abi-aa/releases/download/2022Q3/aapcs32.pdf

[predeclared register names] https://developer.arm.com/documentation/dui0068/b/Assembler-Reference/Predefined-register-and-coprocessor-names/Predeclared-register-names

[apcs register match] https://developer.arm.com/documentation/dui0041/c/ARM-Procedure-Call-Standard/APCS-definition/APCS-register-names-and-roles