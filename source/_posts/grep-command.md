---
title: grep命令
date: 2019-08-05 20:09:44
tags: 
- grep命令
categories: 
- linux命令
- grep命令
---

## 基本用法

### grep命令

```bash
-c 统计包含匹配字符串的行数。
-i 忽略字符大小写的差别。
-v 反转查找。
-o 只输出文件中匹配到的部分。
-E 将范本样式为延伸的普通表示法来使用，意味着使用能使用扩展正则表达式。
-r 递归查找。
```

[GNU Grep用法](https://www.gnu.org/savannah-checkouts/gnu/grep/manual/grep.html)



## 示例

```bash
grep "test[53]" jfedu.txt 		以字符test开头，接5或者3的行

```



