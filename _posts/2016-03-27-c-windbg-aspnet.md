---
layout: post
title: WinDbg排查.Net问题系列前言
date: 2016-03-27
author: luchigster
comments: false
categories: [Blog]
tags: [AspNet,WinDbg]
---

### WinDbg排查.Net问题系列
一直想写些关于`WinDbg`、`AspNet`的文章，主要是的CPU、内存等问题的解决方法。

度娘里面能够找到的文章不多，而且都比较浅显，对于熟悉`WinDbg`的同学来说，还能够理解那怕出现一些问题也能自行解决，对于新手则望而却步。

记得刚开始时，为了分析`w3wp`的`dump`而安装了`WinDbg`，但是一直没有加载`sos`成功，还好找到有个台湾的同学写到需要拿到服务器的`sos`到本地才能加载成功。

初步定有以下系列文章，以后工作上有实际经验则贴出典型的实例：

1. `WinDbg`
2. `SOS`、`SOSEX`和`Psscor`
3. 内存问题排查
4. CPU问题排查
5. 总结
6. 实例...