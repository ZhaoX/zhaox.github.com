---
layout: post
title: "在Mysql中Using filesort代表什么意思？"
description: ""
category: Database
tags: [mysql]
---
{% include JB/setup %}

在Mysql中使用explain来查看sql执行信息时，经常会看到Using filesort。那么Using filesort在MySQL中代表什么意思呢？

有人会说是外部排序，其实是不对或者不准确的。事实上Using filesort是一个非常差的命名。真实的情况是，如果一个排序操作不能通过索引来完成，那这次排序操作就叫做filesort，这跟file没有任何关系。filesort应该叫做sort，而它的实现，就是大家熟悉的快排。

参考：
https://www.percona.com/blog/2009/03/05/what-does-using-filesort-mean-in-mysql/
http://s.petrunia.net/blog/?p=24

