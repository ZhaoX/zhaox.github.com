---
layout: post
title: "B站账号服务浅析"
description: "Bilibili Passport"
category: Passport
tags: [Passport]
---
{% include JB/setup %}

### 技术选型

##### 编程语言
Go

##### 数据存储
HBase

##### 监控
trace



### 数据模型

### 功能描述


### 用户密码加密

加随机盐之后md5

``` go
func getSaltPwd(pwd string, salt string) string {
	return md5Hex(fmt.Sprintf("%s>>BiLiSaLt<<%s", pwd, salt))
}

func md5Hex(s string) string {
	sum := md5.Sum([]byte(s))
	return hex.EncodeToString(sum[:])
}
```
