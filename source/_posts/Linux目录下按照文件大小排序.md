---
title: Linux目录下按照文件大小排序
date: 2022-05-18 17:35:19
tags:
  - ll
  - find
  - du
categories:
  - Linux
---

### Linux对目录下文件排序命令

实际工作中我们经常需要对目录下的文件进行由大到小的排序后进行清理，释放磁盘空间。
有常用的三个命令可以将文件进行大小排序，如下：

#### 1. ll

**1) 按照文件大小降序排列**

``` bash
ll -hS
```

![img](1.png)

**2) 按照文件大小升序排列**

``` bash
ll -hrS
```

![img](2.png)

#### 2. du

**1) 按照 B（字节）单位转换排序（升序排序）**

``` bash
du -b * | sort -n
```

-b的意思是字节，可以改为-k或者-m，分别为KB,MB的单位。

![img](3.png)

**2) 按照 B（字节）单位转换排序（降序排序）**

``` bash
du -b * | sort -rn
```

![img](4.png)

#### 3. find

**1) 按照文件大小进行降序排列**

``` bash
find ./ -type f -printf '%s %p\n' | sort -rn
```

![img](5.png)

**2) 按照文件大小进行升序排列**

``` bash
find ./ -type f -printf '%s %p\n' | sort -n
```

![img](6.png)

