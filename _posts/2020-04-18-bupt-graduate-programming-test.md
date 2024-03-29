---
layout: article
title: '计算机研究生机试真题'
tags: code
---

下面是当时的机试题：


## 题目一、二进制数

### 题目描述：

大家都知道，数据在计算机里中存储是以二进制的形式存储的。有一天，小明学了 C 语言之后，他想知道一个类型为 unsigned int 类型的数字，存储在计算机中的二进制串是什么样子的。你能帮帮小明吗？并且，小明不想要二进制串中前面的没有意义的 0 串，即要去掉前导 0。

### 输入：

第一行，一个数字 T（T <= 1000），表示下面要求的数字的个数。接下来有 T 行，每行有一个数字 n（0 <= n<= 10^8），表示要求的二进制串。

### 输出：

输出共 T 行。每行输出求得的二进制串。

### 样例输入：

~~~text
5
23
535
2624
56275
989835
~~~

### 样例输出：

~~~text
10111
1000010111
101001000000
1101101111010011
11110001101010001011
~~~



## 题目二、矩阵幂

### 题目描述：

给定一个 n*n 的矩阵，求该矩阵的 k 次幂，即 P^k。

### 输入：

输入包含多组测试数据。

数据的第一行为一个整数 T（0 < T <= 10），表示要求矩阵的个数。接下来有 T 组测试数据，每组数据格式如下：

第一行：两个整数 n（2 <= n <= 10）、k（1 <= k <= 5），两个数字之间用一个空格隔开，含义如上所示。

接下来有 n 行，每行 n 个正整数，其中，第 i 行第 j 个整数表示矩阵中第 i 行第 j 列的矩阵元素 Pij 且（0 <= Pij <= 10）。另外，数据保证最后结果不会超过 10^8。


### 输出：

对于每组测试数据，输出其结果。格式为：

n 行 n 列个整数，每行数之间用空格隔开，注意，每行最后一个数后面不应该有多余的空格。

### 样例输入：

~~~text
3
2 2
9 8
9 3
3 3
4 8 4
9 3 0
3 5 7
5 2
4 0 3 0 1
0 0 5 8 5
8 9 8 5 3
9 6 1 7 8
7 2 5 7 3
~~~

### 样例输出：

~~~text
153 96
108 81
1216 1248 708
1089 927 504
1161 1151 739
47 29 41 22 16
147 103 73 116 94
162 108 153 168 126
163 67 112 158 122
152 93 93 111 97
~~~


## 题目三、二叉排序树

### 题目描述：

二叉排序树，也称为二叉查找树。可以是一颗空树，也可以是一颗具有如下特性的非空二叉树：

1. 若左子树非空，则左子树上所有节点关键字值均不大于根节点的关键字值；
2. 若右子树非空，则右子树上所有节点关键字值均不小于根节点的关键字值；
3. 左、右子树本身也是一颗二叉排序树。

现在给你 N 个关键字值各不相同的节点，要求你按顺序插入一个初始为空树的二叉排序树中，每次插入后成功后，求相应的父亲节点的关键字值，如果没有父亲节点，则输出 -1。

### 输入：

输入包含多组测试数据，每组测试数据两行。

第一行，一个数字 N（N <= 100），表示待插入的节点数。

第二行，N 个互不相同的正整数，表示要顺序插入节点的关键字值，这些值不超过 10^8。

### 输出：

输出共 N 行，每次插入节点后，该节点对应的父亲节点的关键字值。

### 样例输入：

~~~text
5
2 5 1 3 4
~~~

#### 样例输出：

~~~text
-1
2
2
5
3
~~~

## 题目四、IP 数据包解析

### 题目描述：

我们都学习过计算机网络，知道网络层 IP 协议数据包的头部格式如下：

![IP 报文结构]({{site.img_url}}/2012-ip-header.png){:.center}

其中 IHL 表示 IP 头的长度，单位是 4 字节；总长表示整个数据包的长度，单位是 1 字节。传输层的 TCP 协议数据段的头部格式如下：

![TCP 报文结构]({{site.img_url}}/2012-tcp-header.png){:.center}

头部长度单位为 4 字节。

你的任务是，简要分析输入数据中的若干个 TCP 数据段的头部。 详细要求请见输入输出部分的说明。

### 输入：

第一行为一个整数 T，代表测试数据的组数。

以下有 T 行，每行都是一个 TCP 数据包的头部分，字节用 16 进制表示，以空格隔开。数据保证字节之间仅有一个空格，且行首行尾没有多余的空白字符。保证输入数据都是合法的。

### 输出：

对于每个 TCP 数据包，输出如下信息：

* Case #x，x 是当前测试数据的序号，从 1 开始。
* Total length = L bytes，L 是整个 IP 数据包的长度，单位是 1 字节。
* Source = xxx.xxx.xxx.xxx，用点分十进制输出源 IP 地址。输入数据中不存在 IPV6 数据分组。
* Destination = xxx.xxx.xxx.xxx，用点分十进制输出源 IP 地址。输入数据中不存在 IPV6 数据分组。
* Source Port = sp，sp 是源端口号。
* Destination Port = dp，dp 是目标端口号。
* 对于每个 TCP 数据包，最后输出一个多余的空白行。

具体格式参见样例。

请注意，输出的信息中，所有的空格、大小写、点符号、换行均要与样例格式保持一致，并且不要在任何数字前输出多余的前导 0，也不要输出任何不必要的空白字符。

### 样例输入：

~~~text
2
45 00 00 34 7a 67 40 00 40 06 63 5a 0a cd 0a f4 7d 38 ca 09 cd f6 00 50 b4 d7 ae 1c 9b cf f2 40 80 10 ff 3d fd d0 00 00 01 01 08 0a 32 53 7d fb 5e 49 4e c8
45 00 00 c6 56 5a 40 00 34 06 e0 45 cb d0 2e 01 0a cd 0a f4 00 50 ce 61 e1 e9 b9 ee 47 c7 37 34 80 18 00 b5 81 8f 00 00 01 01 08 0a 88 24 fa c6 32 63 cd 8d
~~~

### 样例输出：

~~~text
Case #1
Total length = 52 bytes
Source = 10.205.10.244
Destination = 125.56.202.9
Source Port = 52726
Destination Port = 80

Case #2
Total length = 198 bytes
Source = 203.208.46.1
Destination = 10.205.10.244
Source Port = 80
Destination Port = 52833
~~~

如果想测试以上代码的话，可以到[九度 OJ](http://ac.jobdu.com/problemset.php?search=2012%E5%B9%B4%E5%8C%97%E4%BA%AC%E9%82%AE%E7%94%B5%E5%A4%A7%E5%AD%A6%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A0%94%E7%A9%B6%E7%94%9F%E6%9C%BA%E8%AF%95%E7%9C%9F%E9%A2%98) 上提交你的代码。