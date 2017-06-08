---
layout: post
title: "动态添加Redis密码认证"
description: ""
category: Database 
tags: [Redis, Security]
---
{% include JB/setup %}

如果redis已在线上业务使用中，但没有添加密码认证，那么如何在不影响业务服务的前提下给redis添加密码认证，就是一个需要仔细考虑的问题。

本文描述一种可行的方案，适用于客户端使用了jedis连接池，服务端使用了redis master-slave集群的情况。

### 1.定制jedis

对redis返回的错误的处理，做两处修改：

忽略 (error) ERR Client sent AUTH, but no password is set。使配置了密码的jedis可以在没有配置密码redis上使用；

发生(error) NOAUTH Authentication required时，将当前connection置为broken，从而将连接踢出连接池。这样动态给redis添加上密码时，jedis会自动重新创建可用连接。

我已经对jedis 2.8.x版本做好了以上修改。可以直接[下载](http://zhaox.github.io/assets/files/jedis-2.8.3.jar)使用 。如果使用了更高的版本jedis，可以参考我的[代码](https://github.com/ZhaoX/jedis)自行修改；如果使用了更低版本的，建议升级到2.8.x。

### 2.在项目代码中使用定制的jedis

修改maven配置。将原来的jedis依赖注释掉，添加对本地的定制jedis的依赖：

```
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.8.3</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/../libs/jedis-2.8.3.jar</systemPath> <!-- 此处的systemPath是jedis-2.8.3所在的相对路径 -->
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
    <version>2.4.2</version>
</dependency>
<!--
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.8.1</version>
</dependency>
-->
```

因为把定制jedis通过本地jar包的形式提供，maven不会自动加载jedis的依赖，所以需额外添加对commons-pool2的依赖。

### 3.如果使用了低版本的jedis

老版本jedis的returnBrokenResource和returnResource这两个方法在新版本jedis中已经废弃，如果升级jedis版本的话，需要替换为close方法。

替换前：

```
try {   
  // ...  
} catch (JedisException e) {
  // ...   
  pool.returnBrokenResource(jedis);   
}   
finally {   
  pool.returnResource(jedis);   
}
```

替换后：

```
try {   
  // ...  
} catch (JedisException e) {   
  // ...   
}   
finally {   
  jedis.close();
}
```

### 4.将使用定制jedis的项目代码上线
此时redis尚未添加密码，但定制jedis忽略了“ERR Client sent AUTH, but no password is set”，所以线上运行正常。

### 5.给redis server添加密码认证
动态添加密码会导致redis主从同步断开，为避免引起全量同步对业务造成较大影响。需要dba先调大redis master的client-output-buffer-limit和repl-backlog-size参数，再做配置密码操作。

给redis server添加密码的同时，观察业务代码的log，添加完密码后，log中会出现数次如下报错，随后恢复正常。报错次数是添加密码时，业务服务器的jedis连接池中与该redis server之间连接数量。

```
redis.clients.jedis.exceptions.JedisConnectionException: NOAUTH Authentication required.
```

如果使用了shardedJedis，请逐个分片进行操作，最小化对业务服务的影响。

### 6.更换jedis为官方版本
定制jedis就是为了动态添加密码认证。添加完毕后，换回官方jedis，方便今后升级。

```
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.8.1</version>
</dependency>
```
