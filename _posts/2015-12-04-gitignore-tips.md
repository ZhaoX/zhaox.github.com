---
layout: post
title: "编写.gitignore文件的几个小技巧"
description: ""
category: Git 
tags: [Git]
---
{% include JB/setup %}

记录几个编写.gitignore文件的小技巧，可能你早就知道了，但我是google了一番才找到写法。

###忽略所有名称为bin的文件夹

```
bin/
```

###只忽略第一级目录中，名称为bin的文件夹

```
/bin/
```

###忽略所有后缀名为.o的文件

```
*.o
```

###忽略所有deps文件夹，除了第一级deps目录下的test文件夹

```
deps/
!/deps/test/
```

1. 加！表示白名单。
2. .gitignore文件中的内容，自上而下，覆盖生效。

