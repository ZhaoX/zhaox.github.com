---
layout: post
title: "这样写代码没有Bug​"
description: "How to write right programs"
category: Computer Science
tags: [Computer Science, BUG, Testing]
---
{% include JB/setup %}

最近看Fred大神在ElixrDaze 2018的演讲视频「The Hitchhiker's Guide to the Unexpected」，深有感触，想起以前上学时老师讲的八卦。

老师讲，全世界的程序员都想写出没有Bug的程序，欧洲人比较理想总想从数学上证明自己的程序是正确的；美国人偏务实倾向于用软件工程和测试保证程序可用。

最后美国人赢了。老师遗憾地说。

视频有点长，又是英文的。我结合自己的见解再加工一番，以飨读者。

### Bug从哪来

![DefineBug](http://zhaox.github.io/assets/images/DefineBug.png)



### 分类消灭Bug

![SolveBug](http://zhaox.github.io/assets/images/SolveBug.png)

#### 从开发工程师的角度

努力学习、少猜测勤动手多验证、做的越少错的越少、静态代码检查、复盘、

#### 从团队管理者的角度

Code Review、建立团队通识

#### 从测试的角度

#### 从系统设计的角度

奥卡姆剃刀、增强系统的可观测性（VM、操作系统）、fail fast、故障隔离、make it irrelevant

### 总结

结尾了再看老师的八卦，其实说欧洲人美国人有点偏见了，应该是数学家和工程师在面对问题时的两种思维方式。

相信数学家的好方法，最终会被优秀的工程师用起来。
​