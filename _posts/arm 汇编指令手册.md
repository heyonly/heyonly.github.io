---
layout: post
title: arm 汇编指令手册
categories: blog
description: arm 汇编
keywords: arm 汇编
--- 

<h4>1、A64常用寄存器</h4>

x0-x30 64bit 通用寄存器，有只用低 32bit的w0-w30

FP(x29) 64bit 栈底指针

LR(x30) 64bit 通常称x30为程序链接寄存器，保存跳转返回信息地址

XZR 64bit Zero寄存器，写入此寄存器的数据被忽略，读出的数据全为0

WZR 32bit Zero寄存器的32bit形式

ELR_ELx 64bit 异常链接寄存器，保存异常进入ELx的异常地址(x={1,2,3})

SP_ELx 64bit 栈指针, 保存进入ELx的栈地址(x={0,1,2,3})

PC 64bit 当前程序运行的指针

SPSR_ELx 32bit 保存的处理状态寄存器(x={1,2,3})，用于存放程序运行中一些状态标识。

状态寄存器又分为 The Current Program Status Register (CPSR) 和

The Saved Program Status Registers (SPSRs)。 一般都是使用CPSR，

当发生异常时， CPSR会存入SPSR。当异常恢复，再拷贝回CPSR。

这里的ELx (x={0,1,2,3}) 是异常等级(exception level)，每个程序能跑到不同级别，最低的是L0最高的是L3，这里不具体展开说明。

x0-x7: 用于子程序调用时的参数传递，x0还用于返回值传递。

读取寄存器的值：
有两种方式:


```
(lldb) x -f A $sp
0x16d561ae0: 0x0000000000000001
0x16d561ae8: 0x000000010380e708
0x16d561af0: 0x0000000104109b80
0x16d561af8: 0x00000001f4fa0cde 
0x16d561b00: 0x000000016d561b70
0x16d561b08: 0x00000001028a060c Swift-ObjectiveC`-[ViewController viewDidLoad] + 84 at ViewController.m:159:5
0x16d561b10: 0x0000000000000000
0x16d561b18: 0x0000000000000000

(lldb) memory read -f s $sp
0x16d561ae0: "\x01"


(lldb) memory read -format x -size 4 0x16d561ae0
0x16d561ae0: 0x00000001 0x00000000 0x0380e708 0x00000001
0x16d561af0: 0x04109b80 0x00000001 0xf4fa0cde 0x00000001


dump 内存：
(lldb) memory read -outfile /Users/haibing6/Desktop/a.bin -binary 0x1028a0518 0x1028a0574
92 bytes written to '/Users/haibing6/Desktop/a.bin'


```

<h4>adr</h4>
作用：小范围的地址读取指令。adr 指令将基于PC 相对偏移的地址值读取到寄存器中


原理：将有符号的21位的偏移，加上PC，结果写入到通用寄存器，可用来计算 1MB 范围内的任意字节的有效地址


<h4>adrp</h4>
作用：以页为单位的大范围的地址读取指令，这里的p 就是page 的意思

原理： 符号扩展一个21位 的offset ，向左移动12位，PC 的值的低12位清零，然后将这两者相加，结果写入到Xd 寄存器中，用来得到一块含有4KB 对齐内存区域的base 地址

eg：


```
0x104d8055c <+68>: str    x10, [x11]
0x104d80560 <+72>: adrp   x0, 2
0x104d80564 <+76>: add    x0, x0, #0x3d1  

(lldb) register read $x0
      x0 = 0x0000000121e08f90
(lldb) si
(lldb) register read $x0
      x0 = 0x0000000104d82000  Swift-ObjectiveC`(anonymous namespace)::addImageCallback(mach_header const*) + 380
(lldb) register read $pc
      pc = 0x0000000104d80564  Swift-ObjectiveC`asm_mem_to_mem + 76 at ViewController.m:83:5
(lldb) 

符号扩展一个21位的offset： 0x000002000
左移12位： 0x002

0x0000000104d80564 低12位清零：0x0000000104d80000
0x0000000104d80000+0x000002000 = 0x0000000104d82000


```


<h4>.p2align</h4>
作用：

指明内存对齐所需的2次幂字节对齐

eg:

```
.p2align 2
内存对齐在4字节

.p2align 3
8 字节对齐

```





[arm64 架构之入栈/出栈操作](https://juejin.cn/post/6844903816362459144)