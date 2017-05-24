---
layout: wiki
title: Vim
categories: Vim
description: 个人最常用的 Vim 常用操作。
keywords: Vim
---
Vim 的常用操作         
块列选择：ctl+v     
块行选择：V 
gg 回到文件头   
G  定位到文件尾  
$ 定位到行尾  
0 定位到行头  
o 下一行编辑  
O 上一行编辑  
A 行尾符编辑  
a 光标下一个字符编辑  
i 光标上一个字符编辑  
u 回退  
yy 复制一行  
nyy 从当前行复制n行  
p 粘贴  
r 替换字符  

以上是插入模式下。  
下面介绍一些命令模式下的常用命令：  
按 ： 进入命令行模式  
set number 显示行号
s : 表示替换操作  
[option] : 表示操作类型  
如：g 表示全局替换;   
如将文本中test 替换为haha  
:%s/test/haha/g  
#删除所有空行
:g/^$/d  
#将That or this 换成 This or that
:%s/\(That\) or \(this\)/\u\2 or \l\1/

好了 ，就先介绍这些吧，做笔记用

