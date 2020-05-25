---
layout: post
title: lldb 源码映射的技术实现
categories: lldb
description:lldb 源码映射的技术实现
keywords: lldb
---  

一、需求


由于我们的项目比较大，编译一次大概需要一个多小时。对于线上问题，需要先切换分支，然后编译，然后再进行调试，定位问题，时间比较长，因此，有没有一种方案，在线上出问题后，能够立马进行远吗级别 的调试，定位问题。


二、方案

针对以上需求，提出了一种方案


在每次发版时先编译一个debug 版本，当有线上问题发生时，直接下载对应的debug 版本，将debug 版本的.app 文件放到product 目录中，执行run without building，便可直接进行调试，定位问题 


三、问题

1、业务组同学端上的代码与线上发生问题的版本代码极大概率上是不一样的，这样导致业务组同学只能查看汇编代码而加大了调试难度


2、debug版本的.app 文件中是不包含符号信息的，

```
$ dwarfdump testRuntime.app/testRuntime 
testRuntime.app/testRuntime:	file format Mach-O 64-bit x86-64

.debug_info contents:

```
如何获取符号信息


四、解决问题


针对以上问题，有以下解决思路

1、添加符号信息

a 在调试时，将符号信息导入到lldb 中

```
(lldb) add-dsym /Users/haibing6/Downloads/Archive/testRuntime.dSYM
symbol file '/Users/haibing6/Downloads/Archive/testRuntime.dSYM/Contents/Resources/DWARF/testRuntime' has been added to '/Users/haibing6/Library/Developer/Xcode/DerivedData/testRuntime-bbzzorffuqwpioemkhzriysmacxo/Build/Products/Debug-iphonesimulator/testRuntime.app/testRuntime'
```
b 源码映射

```
(lldb) settings set target.source-map /Users/heyonly/Downloads/testRuntime /Users/haibing6/Downloads/testRuntime
```

2、获取dwarf 文件


两种方式


使用xcode 生成
![](/images/blog/lldb/01/2020-05-25-6.17.44.png)

或者

```
dsymutil /Users/haibing6/Downloads/Archive/testRuntime.app/testRuntime -o /Users/haibing6/Downloads/Archive/testRuntime.dSYM
```
附：关于dwarf 文件
```
$ dwarfdump /Users/haibing6/Downloads/Archive/testRuntime.dSYM/Contents/Resources/DWARF/testRuntime > ts.mm
```


可以看到

![](/images/blog/lldb/01/2020-05-25-2.41.12.png)



更多详细信息，请移步



四、实施
1、流程图

![](/images/blog/lldb/01/2020-05-25-2.11.58.png)


2、实施步骤


* Jenkins服务器打包时生成dwarf 文件，以tag值作为标识符保存
	
	
* 客户端执行提供的脚本（提供tag 值），下载对应的.app文件以及dwarf文件
	
	
* 启动lldb，执行 `command script import WBLoadSymbols.py`
	
	
	
五、收益


参考：

[1]\:[探索 DWARF 调试格式信息](https://www.ibm.com/developerworks/cn/aix/library/au-dwarf-debug-format/index.html)

[2]\:[DWARF Debugging Information Format](http://dwarfstd.org/doc/DWARF5.pdf)


