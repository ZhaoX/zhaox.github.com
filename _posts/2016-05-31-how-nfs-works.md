---
layout: post
title: "NFS原理介绍"
description: ""
category: Storage 
tags: [Storage, NFS]
---
{% include JB/setup %}

NFS全称Network File System，即网络文件系统。NFS独立于操作系统，可以在不同的物理机器、不同的操作系统之间共享文件系统。用户访问NFS上的文件就像访问自己本地的文件系统一样，非常便捷。由Sun公司开发，于1984年向外公布。NFS应用广泛，除了是类Unix系统间共享文件的事实标准外，也是NAS系统必备的存储协议之一。

本文不会介绍NFS的配置管理和协议细节，而将着重NFS的基本工作原理，之后会介绍如何用Java实现和定制NFS。

![NFS](http://zhaox.github.io/assets/images/NFS.png) 

(未完待续) 最近挖坑很多，希望能都填上。
