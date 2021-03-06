---
layout: post
title: "每个程序员都应该了解的内存知识（开篇）"
description: ""
category: "What Every Programmer Should Know About Memory"
tags: [Computer Architecture, Memory, CPU]
---
{% include JB/setup %}

注：本文多数内容来自对[What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)一文的理解和翻译。但这不是一篇严格的译文，只摘取我读了有心得的部分，并尽量加上自己的理解和自己收集的资料。

###开篇
早期计算机系统的架构要比现在的简单多了，CPU、内存、外存、网卡等几个重要部件一组合就成了鼎鼎大名的计算机。每个部件自身结构相对简单，各部件的工作能力也旗鼓相当，它们按照冯诺依曼体系结构协调工作在一起，十分协调融洽。说是早期，其实计算机技术的发展日新月异，也就是几十年前吧。

计算机系统的几大部件都凝聚了工程师的心血，并有很多优秀的工程师一直工作在优化这些部件上。很快，各个部件的工作能力就出现了差距，其中尤以内存和外存为甚，受限于价格的因素，内存外存的工作能力被CPU远远的落在了后面。这样一来，计算及系统就存在短板效应，CPU空有飞速的计算速度，却不能带动整个计算机系统的工作能力。

外存瓶颈的解决，多是通过软件技术来做缓存。

1. 操作系统实现了针对外存的缓存，保存经常读写的数据在内存中；
2. 存储系统也被内置到外存设备中，比如磁盘控制器中，这样即便没有操作系统缓存存在的情况下，也能提升整体性能。

遗憾的是，内存存取瓶颈的解决，要比外存瓶颈的解决难得多，并且几乎所有的解决方案，都需要改动硬件。这些为改善内存瓶颈而对硬件的改动，主要包含以下几种：

1. 内存的硬件结构设计（体现在速度和并发上）;
2. 内存控制器的设计；
3. CPU缓存；
4. 设备直接内存访问（DMA）。


###后续内容预告

- [每个程序员都应该了解的内存知识（RAM篇）]()介绍随机访问内存（RAM）的技术细节；
- [每个程序员都应该了解的内存知识（CPU Cache篇）]()介绍CPU缓存的相关技术；
- [每个程序员都应该了解的内存知识（Virtual Memory篇）]()介绍虚拟内存技术；
- [每个程序员都应该了解的内存知识（NUMA篇）]()介绍非统一内存访问架构；
- [每个程序员都应该了解的内存知识（编程篇）]()介绍如何利用内存知识来优化自己的程序；
- [每个程序员都应该了解的内存知识（工具篇）]()介绍几个可以用来辅助分析程序内存使用的工具；
