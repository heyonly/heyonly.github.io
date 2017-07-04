---
layout: post
title: iOS 安全攻防之fishhook解析
categories: blog
description: fishhook
keywords: iOS安全攻防,fishhook
---


在iOS开发中，我们可以利用objective-c 的runtime 来实现函数的动态替换，这对修改系统函数行为来解决bug具有很重要的意义，特别是对于大型工程来说，修改bug更显得重要。然而在iOS开发中，我们还会用到大量的C  函数，在大型工程中，大量的C函数的使用，牵一发而动全身。因此我们也希望也能像OC 一样，能够动态修改C函数行为。此时，fishhook 就是Facebook 专门为此而开发的。fishhook 是安全的，没有用到私有API，在Yahoo的多个项目中都有用到。使用也非常简单。



通过\:[fishhook的官方文档](https://github.com/facebook/fishhook) 可以知道，fishhook很简单：

