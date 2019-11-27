---
title: linux基本命令
date: 2019-08-05 18:23:29
tags: 
- linux基本命令
categories: 
- linux命令
- linux基本命令
---

## sort: 排序

`sort` 命令的用法很简单。最基本的用法有两种：

```bash
cat data.txt | sort
sort data.txt
```

选项：

```bash
-r  倒序
-f	忽略字母大小写
```

`-k` 指定用于排序的栏目范围，`-t` 指定栏目的分隔符。

以下实例，用`:`分割行，排序值从第3栏开始：

```bash
$ sort -k 3 -t : sort.txt
aaa:30:1.6
bbb:10:2.5
ccc:50:3.3
ddd:20:4.2
eee:60:5.1
eee:40:5.4
```

## uniq: 重复

`uniq` 的作用可以描述为：针对相邻的重复行，进行相应的动作。

这句话中，有两个地方需要注意。首先，针对的是**相邻的重复行**。因此，`uniq` 对于不相邻的重复行是不起作用的。其次，进行**相应的**动作。这意味着，`uniq` 可以做的事情很多，不止一样。

不带任何参数的时候 `uniq` 的动作是：对相邻的重复行进行去除。例如：

```
cat <filename> | sort | uniq
```

我们已经见过了 `sort` 的作用，那么上面命令的作用就很显然了：将 `<filename>` 按照 ASCII 升序排序；然后去除重复出现的行；最后将这个没有重复行的内容输出到标准输出。

给 `uniq` 加上参数，就能解锁更多姿势。

```bash
cat <filename> | sort | uniq -d     # 只显示重复的行，每行只显示一次
cat <filename> | sort | uniq -D     # 只显示重复的行
cat <filename> | sort | uniq -i     # 忽略大小写
cat <filename> | sort | uniq -u     # 只显示只出现一次的行
cat <filename> | sort | uniq -c     # 统计每行重复的次数
```



## xargs: 将标准输入转为命令行参数

Unix有些命令可以接受"标准输入"（stdin）作为参数。

> ```bash
> $ cat /etc/passwd | grep root
> ```

上面的代码使用了管道命令（`|`）。管道命令的作用，是将左侧命令（`cat /etc/passwd`）的标准输出转换为标准输入，提供给右侧命令（`grep root`）作为参数。

但是，大多数命令都不接受标准输入作为参数，只能直接在命令行输入参数，这导致无法用管道命令传递参数。举例来说，`echo`命令就不接受管道传参。

> ```bash
> $ echo "hello world" | echo
> ```

上面的代码不会有输出。因为管道右侧的`echo`不接受管道传来的标准输入作为参数。

`xargs`命令的作用，是将标准输入转为命令行参数。

> ```bash
> $ echo "hello world" | xargs echo
> hello world
> ```

上面的代码将管道左侧的标准输入，转为命令行参数`hello world`，传给第二个`echo`命令。

### 示例

1. 查找所有后辍为txt的文件列表，然后进行grep: 

```bash
$ find . -name "*.txt" | xargs grep "abc"
```



2. 由于`xargs`默认将空格作为分隔符，所以不太适合处理文件名，因为文件名可能包含空格。

`find`命令有一个特别的参数`-print0`，指定输出的文件列表以`null`分隔。然后，`xargs`命令的`-0`参数表示用`null`当作分隔符。

```bash
find . -type f -name "*.log" -print0 | xargs -0 rm -f
```

统计一个源代码目录中所有 php 文件的行数：

```bash
find . -type f -name "*.php" -print0 | xargs -0 wc -l
```

查找所有的 jpg 文件，并且压缩它们：

```bash
find . -type f -name "*.jpg" -print | xargs tar -czvf images.tar.gz
```



## 打包压缩: `tar` 命令

### 常用打包压缩格式

```bash
.zip .gz .bz2 
.tar
.tar.gz .tar.bz2
```

### `.tar.gz` 格式

> 其实，`.tar.gz` 格式是先将文件或目录打包文 `.tar` 格式，再压缩为 `.gz` 格式

#### 压缩

```
tar -zcvf 压缩包名.tar.gz 源文件
```

- 选项

> `-z` : 压缩为 .tar.gz 格式
>
> `-c`：建立一个压缩档案的参数指令(create 的意思)；

#### 解压缩

```
tar -zxvf 压缩包名.tar.gz
```

- 选项

> `-x` : 解压缩
> `-t` : 查看压缩保内文件，但是不解压缩

- 实例

```bash
[vagrant/tmp/tmp] ]$tar -zcvf /tmp/etc.tar.gz /etc  <==打包后，以 gzip 压缩
[vagrant/tmp/tmp] ]$tar -ztvf etc.tar.gz   <==查阅上述/tmp/etc.tar.gz 档案内有哪些档案
[vagrant/tmp/tmp] ]$tar -zxvf /tmp/etc.tar.gz  <==将/tmp/etc.tar.gz 档案解压缩在vagrant/tmp/tmp下
```



## stat: 读取文件(夹)状态

stat命令可以显示文件的修改时间，大小，权限模式，磁盘占用等情况，相比ls命令，能显示更多的文件状态信息，也有更高的自由度: 

```bash
stat install.log
```

实例如下: 

```bash
[root@localhost ~]# stat install.log
  File: "install.log"
  Size: 7730            Blocks: 16         IO Block: 4096   普通文件
Device: fd00h/64768d    Inode: 1048578     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2015-10-20 10:21:05.473712000 +0800
Modify: 2015-10-20 10:23:50.061712001 +0800
Change: 2015-10-20 10:23:56.028712002 +0800
```

