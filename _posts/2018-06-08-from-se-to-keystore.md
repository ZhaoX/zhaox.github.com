---
layout: post
title: "从 Secure Element 到 Android KeyStore"
description: "From Secure Element to Android KeyStore"
category: Security 
tags: [Secure Element, NFC, HCE, TEE, Android, Fingerprint HAL, KeyStore]
---
{% include JB/setup %}

忽如一夜春风来，智能手机来到每个人的手上，我们用它支付、理财、娱乐、工作、记录生活、存储私密信息、乘坐公共交通、开启家门、控制汽车...。智能手机是如此的重要，不知天天把它拿在手上的你,是否关心过它是否足够安全。

本文从Secure Element（安全单元）说起，介绍手机设备上若干重要的安全角色和概念。为后续文章介绍如何基于手机安全地实现认证、支付、DRM等业务流程打下基础。


### SE（Secure Element）

按照 Global Platform的定义：安全单元提供私密信息的安全存储、重要程序的安全执行等功能。其内部组件包含有：CPU、RAM、ROM、加密引擎、传感器等，大致如下图所示：

![SecureElementInternal](http://zhaox.github.io/assets/images/SEInternal.PNG)

外在表现上SE是一块物理上独立的芯片卡。从外在表现上可以分为三种：

- UICC 通用集成电路卡，由电信运营商发布和使用。就是大家购买手机号时的手机SIM卡；
- Embedded SE 虽然也是独立的芯片，但普通用户看不到，由手机制造厂商在手机出厂前集成在手机内部；
- Micro SD 以SD存储卡的形式存在，通过插入SD卡槽集成到手机上。由独立的SE制造商制造和销售；

![SecureElementShape](http://zhaox.github.io/assets/images/SEShape.PNG)

SE物理上独立，采用安全协议与外部通讯。具有自己独立的执行环境和安全存储，软件和硬件上防篡改。软件通过签名等方式防篡改很多人都了解，说下硬件防篡改，简单说就是物理拆改SE，它会自毁。最简单的硬件防篡改的例子，大家可以参考大家给自己车安装车牌时所使用的单向螺丝和防盗帽。

SE固若金汤，但保存在其中的数据和程序需要有更新机制，这通过TSM（Trusted Service Manager）来实现，以保证安全。

![TrustedServiceManager](http://zhaox.github.io/assets/images/TrustedServiceManager.PNG)

SE不年轻了从19世纪70年代就开始发展，但它十分安全，是目前手机上最安全的技术措施。

### NFC（Near-field Communication）

近场通信是一种短距高频的无线电技术，在13.56MHz频率运行于20厘米距离内，由非接触式射频识别（RFID，公交卡、校园一卡通、门禁卡等都采用RFID技术实现）演变而来，由飞利浦、诺基亚和索尼于2004年共同研制开发。目前已成为ISO/IEC IS 18092国际标准、EMCA-340标准与ETSI TS 102 190标准。

NFC设备有三种工作模式：

- 卡模拟模式（Card emulation mode）：这个模式其实就是相当于一张采用RFID技术的IC卡。可以替代现在大量的IC卡（包括信用卡）场合商场刷卡、IPASS、门禁管制、车票、门票等等。
- 读卡器模式（Reader/Writer mode）：作为非接触读卡器使用，可用来实现给公交卡充值等功能。
- 点对点模式（P2P mode）：这个模式和红外线差不多，可用于数据交换，只是传输距离较短，传输创建速度较快，传输速度也快些，功耗低（蓝牙也类似）。将两个具备NFC功能的设备链接，能实现数据点对点传输，如下载音乐、交换图片或者同步设备地址薄。

三种工作模式中，卡模拟模式用途最为广泛，可将用平时使用的各种卡通过手机模拟实现，从此出门不再带卡。此种方式下，NFC芯片通过非接触读卡器的RF域来供电，即便是手机没电也可以工作。

NFC设备若要进行卡片模拟（Card Emulation）相关应用，则必须内置安全单元（Security Element, SE）以保存重要隐私数据。可以说NFC给SE插上了翅膀，在NFC广泛应用的今天，SE如此的重要，成为电信运营商（移动、联通、电信等）、手机厂商（华为、小米等）、操作系统厂商（谷歌、苹果等）的兵家必争之地。

![NFC](http://zhaox.github.io/assets/images/NFC.PNG)

### HCE（Host Card Emulation） 
### TEE（Trusted Execution Environment）
### Android Fingerprint HAL
### Android KeyStore 

### 总结
