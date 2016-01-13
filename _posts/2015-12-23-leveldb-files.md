---
layout: post
title: "一个LevelDB数据库在磁盘上由哪些文件组成？"
description: ""
category: LevelDB
tags: [LevelDB, Storage]
---
{% include JB/setup %}

如果你不知道LevelDB是什么，戳[这里](https://github.com/google/leveldb)。当然了，如果你还不知道LevelDB是什么的话，估计对本文也没什么兴趣。

一个LevelDB数据库在磁盘上表现为一个目录，本文介绍这个数据库目录由哪些类型的文件组成，这些文件的结构是什么样子的，又承担什么样的功能。

下图中，画出了一个LevelDB数据库目录的文件组成。

![Leveldb files](http://zhaox.github.io/assets/images/LeveldbFiles.png)

下面，我来逐个介绍每一种文件类型的格式和作用。

####LOCK

使用编辑器打开，会发现LOCK中什么内容都没有，是一个空文件。它的作用是保证一个LevelDB数据库，在同一时刻，只被一个进程打开着。

这是怎么实现的呢？在Open一个磁盘上的LevelDB数据库流程中，其中会有一个LockFile的步骤，如代码所示：

``` C++
// 代码来自db/db_impl.cc

Status s = env_->LockFile(LockFileName(dbname_), &db_lock_);

// 代码来自db/filename.cc

std::string LockFileName(const std::string& dbname) {
  return dbname + "/LOCK";
}
```

可以看到，这个LockFile的步骤，要锁住的，就是数据库目录下的LOCK这个空文件。那么具体的加锁是怎么实现的呢？我们来看一下代码：

``` C++
// 代码来自util/env_posix.cc

virtual Status LockFile(const std::string& fname, FileLock** lock) {
  *lock = NULL;
  Status result;
  int fd = open(fname.c_str(), O_RDWR | O_CREAT, 0644);
  if (fd < 0) {
    result = IOError(fname, errno);
  } else if (!locks_.Insert(fname)) {
    close(fd);
    result = Status::IOError("lock " + fname, "already held by process");
  } else if (LockOrUnlock(fd, true) == -1) {
    result = IOError("lock " + fname, errno);
    close(fd);
    locks_.Remove(fname);
  } else {
    PosixFileLock* my_lock = new PosixFileLock;
    my_lock->fd_ = fd;
    my_lock->name_ = fname;
    *lock = my_lock;
  }
  return result;
}

static int LockOrUnlock(int fd, bool lock) {
  errno = 0;
  struct flock f;
  memset(&f, 0, sizeof(f));
  f.l_type = (lock ? F_WRLCK : F_UNLCK);
  f.l_whence = SEEK_SET;
  f.l_start = 0;
  f.l_len = 0;        // Lock/unlock entire file
  return fcntl(fd, F_SETLK, &f);
}
```

可以看到实际的加锁是利用fcntl这个系统调用，在LOCK这个文件上，设置排它锁（F_WRLCK）。因为同一个文件上只能设置一个排它锁，这样一来，当有其它进程打开这个LevelDB数据库时，就会加锁失败。

####CURRENT

CURRENT是一个简单的文本文件，包含着最新的MANIFEST文件的名字。如果你现在还不明白为什么会有多个MANIFEST文件，没关系，看下去。CURRENT文件的内容可能是这样的：

```
MANIFEST-000019
```

####MANIFEST


