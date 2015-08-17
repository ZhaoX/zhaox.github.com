---
layout: post
title: "读Avoiding the Disk Bottleneck in the Data Domain Deduplication File System"
description: ""
category: Storage 
tags: [Storage]
---
{% include JB/setup %}

最近在思考和实践怎样应用重复数据删除技术到云存储服务中。找了些论文来读，其中《Avoiding the Disk Bottleneck in the Data Domain Deduplication File System》是鼎鼎大名的李凯教授出品，读来收益匪浅。

###论文主要内容

Data Domain的去重存储系统是商业上大获成功的产品，从产品的角度来讲非常完善，其架构图如下：
![系统架构图][1]
去重存储系统在数据存储和重建的过程中，都需要频繁地访问数据块的索引，即图中的Segment Index。为了降低成本，一般系统都无法接受将index数据全部存储到内存中，而是基于硬盘来实现这个index的存储。这样一来，磁盘相对较低的性能就成了整个系统的性能瓶颈。本文介绍了Data Domain的去重存储系统中，针对这个问题所采用的三种优化技术。

 1. 总结向量。总结向量可以认为是Segment index在内存中的总结，本文使用Bloom filter来实现。当需要查询一个index值是否存在的时候，总结向量可以提供如下功能，如果总结向量指出该index不存在，那么该index就不存在，无需进一步查找；如果总结向量指出该index存在，那么该index有很大可能存在，但并不保证一定存在，需要进一步查找确认。总结向量要比原始的segment index小，从而放入内存，提升效率。当系统正常关闭时，将总结向量写入磁盘，并在启动时从磁盘读入内存。为了应对断电等非正常关闭，系统阶段性的创建checkpoint，将总结向量写入磁盘。在需要恢复的时候，只需读取最近的checkpoint，然后将checkpoint之后产生的新数据添加到总结向量中即可。
 2. 流感知的块排列技术。流感知的块排列技术 (stream-informed segment layout ，SISL)是基于这样一种假设，当一个文件的部分数据块存入系统后，再次使用时，这些数据块有很大的可能还会以想同的顺序出现。比如需要恢复这个文件时候，比如下次备份这个文件的新版本的时候。这种排列方式带来许多好处：a.当数据重建时，可以大幅减少磁盘IO；b.当备份相似数据流（数据新版本时），segment index的cache的局部性更高，更有效。c.在同一个container中，元数据段和数据段分开存储，使得可以快速读取一个container中涉及到的所有segment的index构建cache和Bloom filter。
 3. 局部性保持缓存。使用局部性保持缓存技术来加速重复segment的确认过程。由于segment使用内容的sha1来唯一标识，因此很难基于正在使用的segment来预测将要使用的segment是哪一个，从而预加载，提高缓存命中率。幸而应用了流感知的块排列技术（SISL），这使得实现局部性保持缓存有了可能性。

###我的思考和疑问

 1. 要实现Bloom Filter的check point，需要container是全局有序的，给定某一个container的id，可以从这个container开始一直遍历到最近产生的数据；
 2. 流感知的块排列技术中，如果一个流的数据很少是怎么处理的？还有，当有多个流在备份数据的时候，如果两个流要写入同一个系统中不存在的segment时，怎么办，是不是就重复写入了。
 3. 传统的数据去重存储系统是面向备份应用的，关注的是吞吐量而不是响应时间。如果要用来做online的实时系统，会有新的问题要解决。

 
  [1]: http://7lryjt.com1.z0.glb.clouddn.com/data%20domain.png
