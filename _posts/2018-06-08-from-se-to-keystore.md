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

按照Global Platform的定义：安全单元提供私密信息的安全存储、重要程序的安全执行等功能。其内部组件包含有：CPU、RAM、ROM、加密引擎、传感器等，大致如下图所示：

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

因为不涉及硬件制造，在SE的竞争过程中，操作系统厂商相对弱势，确切的说是谷歌弱势，因为苹果既是操作系统厂商，也是手机厂商。

早期Goole Pay是基于SE实现的，但由于在SE生态环境中弱势的竞争地位，导致Google Pay适配的机型少，难以发展。从Android 4.4开始，谷歌独辟蹊径在Android系统中提供了HCE服务，用来绕过SE直接控制NFC Controller。大概的模式如下图所示：

![HCE](http://zhaox.github.io/assets/images/HCE.PNG)

HCE不在依赖设备上的SE模块，只要有NFC芯片就可以实现支付等功能，但其实是无奈之举。方便是方便了，有两个主要缺点：一个是安全性有所降低，虽然可以使用白盒密码、服务端token校验等一系列手段来提升安全性，但相比SE，安全性降低到依赖Android OS，只要OS被攻破，HCE就无法保证安全；一个是费电，NFC Controller + SE的方案，可以在手机无电的情况下，使用NFC读卡器的电磁信号供电。而HCE则必须在手机供电，OS正常工作甚至还要联网的情况下才能使用。

相对的，因为对设备有这强的控制力，苹果的Apple Pay是基于SE实现的，更安全一些。

### TEE（Trusted Execution Environment）

SE千般好，除了慢。硬件隔离，独立的计算和存储资源，意味着SE的计算性能差、跟主机的数据传输速度也慢，这限制了SE的应用场景。与此同时，移动互联网发展迅速，迫切需要一个更好的安全生态。因此TEE应运而生。

TEE是一个硬件安全执行环境，通常跟平时使用的Rich OS（Android等）共用同一个主处理器（CPU），提供了代码和数据的安全防护、外置设备的安全访问等功能。TEE具有自己的TEE OS，可以安装和卸载执行其中的安全应用TA（TEE Application）。跟SE相比，是一个相对不那么安全，但运行速度更快、功能更丰富的安全环境。为所有支持TEE的手机，提供了操作系统之外的安全方案。

SE、TEE以及REE的对比：

对比项 | SE | TEE | REE
----- | ----- | ----- | ----
安全级别 | 最高（硬件防篡改） | 高（硬件安全方案） | 普通
性能 | 差 | 高 | 高
是否在主处理器执行 | 否 | 是（极个别情况有独立处理器） | 是
安全的外设访问 | 不支持 | 支持 | 不支持
提供硬件证明 | 一定程度上提供  | 提供 | 不提供
软件生态 | 较差  | 较好 | 极好

TEE的内部API和外部API都由Global Platform定义和发布。TEE得到了业界广泛的支持，比如ARM在2006年就发布了ARM处理器下的TEE方案TrustZone，AMD、Intel、华为海思等，也有自己的TEE方案。

![TEE](http://zhaox.github.io/assets/images/TEE.PNG)

TEE广泛应用在支付、身份认证、内容保护等领域。举例来讲，视频厂商往往需要DRM（Digital rights management）系统来保护版权内容能够顺利得在用户设备上播放，而不被泄露。TEE天然适合用来完成这种需求，其安全存储的能力可以用来保存解密版权内容所需密钥，这样，TEE Application访问可信的服务端获取已加密的版权视频后，使用安全密钥解密，然后利用安全访问外置设备的能力，锁住显卡和声卡，将解密后的视频送往显卡和声卡播放。整个过程中，不管是加密密钥还是视频内容都没有离开过TEE，保护了版权视频的安全。尤其值得一提的，因其锁定外置设备的能力，想通过录屏来窃取内容，也是不可能的。

### Android Fingerprint

Android设备的指纹识别，依赖TEE来实现用户指纹认证，要求指纹采集、注册和识别都必须在TEE内部进行，已保证安全。

![AndroidFingerprint](http://zhaox.github.io/assets/images/AndroidFingerprint.png)

### Android KeyStore 

Android从4.0开始引入了KeyStore，开发者可以使用KeyStore API生成密钥、使用密钥签名、使用密钥加解密、获取密钥的属性信息，但无法将密钥本身从KeyStore中取出。因为密钥不进入应用进程，这大大提高了密钥的安全性。随着Android版本更迭，KeyStore的实现不断进化得更加安全，在有些设备上，不仅密钥不进入应用进程，甚至不进入Android OS只存储在TEE或SE中，接下来我们大概列举下KeyStore的进化。

Android 版本 | 新增的KeyStore能力
----- | -----
4.0 | 创世版本，密钥使用用户的passcde加密后存储，支持RSA、ECDSA
4.1 | 增加了使用TEE的基础设施，在可能的情况下密钥会被存储到TEE中
6.0 | 增加支持AES、HMAC；增加了密钥绑定用户认证的能力，即可以指定某些密钥，在每一次使用时，必须由用户进行认证（指纹、passcode等）
7.0 | 强制要求预装7.0系统的设备必须拥有TEE并且支持基于TEE的KeyStore
8.0 | 增加了设备证明（Key Attestation）能力，开发者可通过验证Key Attestation的证书链，来确认密钥的确保存在了TEE中
9.0 | 增加了StrongBox API，如果密钥被放到了StrongBox，那么密钥一定被放到了安全硬件。Google并没有说明安全硬件一定是SE，而是提了一些安全硬件的要求。

### 总结

能被业界接受的，就是好方案。
