---
title: 凯撒密码（Caesar cipher）
date: 2021-11-25T02:47:50+00:00
categories:
  - CTF
  - 编程
tags:
  - Crypto
---

## 一、Introduction

今天在做[CTF题目](https://buuoj.cn/challenges#看我回旋踢)的时候，看到给出内容有点一脸懵，题目如下，要求自然是解出flag了：

```
synt{5pq1004q-86n5-46q8-o720-oro5on0417r1}
```

该文本的格式和flag{xxxx}答案格式非常相像，于是做了个小实验，用flag的ASCII码减去 synt 的ASCII码， python代码如下：

```python
raw="sync"
result="flag"
for i in range(0, 3):
        print(ord(raw[i]) - ord(result[i]))
```

结果输入全都是13，证明了二者的字符是有固定的间隔的，答案也就呼之欲出了。

然后以13作为间隔，将题目内容给出的字符和13做差，得出给出后的ASCII码，这时候忽然发现还是有问题，后来google了以下，才知道这道题考察的是**凯撒密码**的知识。

凯撒密码的定义如下：

> **凯撒密码**（英语：Caesar cipher），或称**凯撒加密**、**凯撒变换**、**变换加密**，是一种最简单且最广为人知的加密技术。凯撒密码是一种[替换加密](https://zh.wikipedia.org/wiki/%E6%9B%BF%E6%8D%A2%E5%BC%8F%E5%AF%86%E7%A0%81)技术，**[明文](https://zh.wikipedia.org/wiki/%E6%98%8E%E6%96%87)中的所有字母都在[字母表](https://zh.wikipedia.org/wiki/%E5%AD%97%E6%AF%8D%E8%A1%A8)上向后（或向前）**按照一个固定数目进行偏移后被替换成[密文](https://zh.wikipedia.org/wiki/%E5%AF%86%E6%96%87)。

![凯撒密码示意图](https://wangaiguo.com/wp-content/uploads/2021/11/image-6.png)

这时候答案就明了了，仅对字母表内容做差，python代码内容如下：

```python
import re

pattern = re.compile(r'[a-z]', re.I)
raw="synt{5pq1004q-86n5-46q8-o720-oro5on0417r1}"
result=""

for i in raw:
        if pattern.match(i):
                result += chr(ord(i) - 13)
                continue
        result += i
print(result)

//flag{5cd1004d-86a5-46d8-b720-beb5ba0417e1}
```
