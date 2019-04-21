---
layout: post
title: "c, c++ 读取速度大比拼: sync_with_stdio(false) 的影响到底有多大?"
date: 2015-09-12
categories: C++
tags: [C, C++, iostream, speed]
---

<p align="center">
<img src="https://s2.ax1x.com/2019/04/21/EFseE9.jpg" style="width:650px;" />
</p>

## 摘要

C++ iostream 类一直被很多程序员诟病. 一是因为流式的输入输出处理使得代码很繁琐, 没有 `fmt` 形式来的简洁; 二是因为 `cin` 与 `cout` 的速度一直遭到质疑, 被认为比不过 C 语言 `scanf/printf` 输入输出的速度. 本次实验主要探讨的主要是后者: iostream 的速度. 主要是通过 `cin` 从文件读取的速度方面与多种方法进行对比.

<!-- more -->

很多博客已经指出 `cin` 与 `cout` 的速度主要是受到了两种限制, 一是为了在读取文件时与 `scanf/printf` 同步, 由 `ios::sync_with_stdio()` 控制. 二是由于 `cin` 与 `cout` 的 stream buffer 的绑定.

- `ios::sync_with_stdio(false)` : 打开时, `cin/cout` 在读取文件时可以与 `scanf/printf` 混用而不会造成指针混乱. 关闭时则不能混用.
- `cin.tie(NULL)`: 默认情况下, `cin` 与 `cout` 是绑定的. 意思是当其中任意一个进行输入输出操作时, 另一个 stream buffer 也会清空.

本次实验主要探讨的是以上两种机制对 C++ 的读写速度影响到底有多大以及 C++ 与 C 语言在读写速度上的差异. 并试图寻找较快的读取方式.

## 实验方法

首先生成 1000,0000 个随机整数到一个文件中, 再分别用不同方法从文件中读取数据到一个数组. 统计每次读取的时间, 对每种方法统计十次最后取平均. 主要测试了十种不同的读取方法:

| 代号         | 描述                                       |
|--------------|--------------------------------------------|
| cin          | freopen()到stdin, 再用cin读取读取;         |
| scanf        | freopen()到stdin, 再用scanf读取;           |
| fread        | 使用fread() 读取整个字符串再处理           |
| read         | 使用read() 读取整个字符串再处理            |
| mmap         | 使用mmap()映射进内存再处理                 |
| pascal       | 使用pascal读取                             |
| ifstream     | 直接使用ifstream读取文件;                  |
| cin~sync     | 使用cin,并关闭 sync_with_stdio(); 关闭同步 |
| cin~tie      | 使用cin, 并关闭 cin.tie(); 解除绑定        |
| cin~tie~sync | 等价于 cin~sync + cin~tie                  |

💡实验代码可以在文章的最后找到.


## 实验结果

| 方法         | ⏳平均时间(s) |
|--------------|-------------|
| cin          | 3.93276     |
| scanf        | 1.57656     |
| fread        | 0.45389     |
| read         | 0.42889     |
| mmap         | 0.41451     |
| pascal       | 1.41711     |
| ifstream     | 0.99613     |
| cin~sync     | 1.06971     |
| cin~tie      | 3.87770     |
| cin~tie~sync | 1.02246     |

## 结论

1. 由结果来看 sync_with_stdio() 是阻碍 cin 速度的关键.
2. cin.tie() 可以稍微再让 cin 的速度快一点点
3. sync_with_stdio() 关闭后, cin 比 scanf 还要快
4. pascal 和 scanf 相当
5. ifsteram 的速度比 cin 从 stdin 要快
6. 先读进内存再处理的有 fread, read, mmap. 其中 read 和 mmap 相当, fread 稍慢一点点.
7. 先整个读进内存再处理比从文件读取快很多, 大概只要0.5s, 但是这是牺牲了内存空间换来的.

**在关闭了同步与绑定以后(主要是同步), `cin` 的读取速度只要大约 1s , 仅仅是 `scanf` 1.5s 的三分之二. 而 `ifstream` 表现更是不俗 由此可见, `cin` 的读取速度并不输于 `scanf`.**

**要想最快的读取速度, 应该先将文件整个读进内存, 再解析内存中的字符串, 但是这样牺牲了内存空间, 并且有的文件巨大, 无法放入内存.**

⚠️ 本次实验环境为 Linux , 处理器 Intel Xeon E5, 不同的环境可能会得到不同的结果!

## 源代码

全部代码我已经上传到 github , 欢迎大家下载进行自己测试.  [源码](https://github.com/liyzcj/IOspeedtest)

💡 注意: 不可在一次评测中评测 `cin~sync` `cin~tie` `cin~tie~sync` `cin`. 因为 `tie(NULL);` `sync_with_stdio(false);` 执行后会一直有效

使用方法:

  1. 运行环境: Linux

  2. 第一次运行会生成 `data.txt`.

  3. 直接修改程序的 vector 选择要评测的方法. 再编译运行即可.

  4. 程序会对每种方法调用十次, 最后输出平均时间

### 文件结构

- Makefile : make编译cpp文件
- data.txt : 每行一个整数, 一共 1000,0000个 (第一次生成)
- load_data : pascal 编译后的执行程序
- load_data.pas : pascal 源码
- seventh.cpp : 实验c++源码
- seventh.out : 编译后的执行程序
