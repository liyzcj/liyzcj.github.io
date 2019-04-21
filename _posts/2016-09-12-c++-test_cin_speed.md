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

💡代码部分较长, 不需要看代码的同学可以直接看结果与结论.

## 实验代码

首先导入需要的头文件, 规定命名空间:

```c++
#include<iostream>
#include<fstream>
#include<random>
#include<iomanip>  // for io manipulator
#include <fcntl.h>  // O_RDONLY for read()
#include<unistd.h>  // linux read()
#include <sys/mman.h> // for mmap()
#include <cstdlib> // for pascal()
#include<sys/time.h> // for gettimeofday()
#include<map>
#include<vector>

using namespace std;
```

声明各种方法的函数:

```c++
//读取函数
void cin_read();
void scanf_read();
void cin_read_nosync();
void ifstream_read();
void cin_read_notie();
void cin_read_nosync_notie();
void fread_parse();
void read_parse();
void mmap_parse();
void pascal();
```

声明需要的功能函数:

```c++
// 功能函数
void generate();  // 生成数据
double evaluate(void (*func)());  // 评估函数
void parse(char *buf); // 解析函数, 用于解析读取后的字符串 用于 fread read mmap 方法.
ostream& operator<< (ostream& sm, map<string, double>& m); //重写<< 输出结果
```

定义需要的全局变量:

```c++
const int MAXN = 10000000; // data个数
const int ITER = 10; // 每次实验循环次数
const int MAXS = 120*1024*1024; //存储整个文件, 用于先读取再解析的方法例如fread read mmap

char buf[MAXS];
int data[MAXN];
int i; // 全局指针
```

定义一个 `map` 用来保存不同的方法函数, 方便调用:

```c++
typedef void (*method)();
map<string,method> methods = {
    {"cin", cin_read},
    {"scanf", scanf_read},
    {"cin~sync", cin_read_nosync},
    {"cin~tie", cin_read_notie},
    {"cin~tie~sync", cin_read_nosync_notie},
    {"ifstream", ifstream_read},
    {"fread", fread_parse},
    {"read", read_parse},
    {"mmap", mmap_parse},
    {"pascal", pascal}
};
```

接下来实现各种方法函数:

### cin

```c++
// cin 读取
void cin_read(){
  freopen("data.txt", "r", stdin);
  for (i=0; i<MAXN; i++){
    cin >> data[i];
  }
}
```

直接将文件重定向到标准输入 stdin , 再用 `cin` 从标准输入读取到数组 `data`中.

### scanf

与 `cin` 相同, 仅仅修改一行换成 `scanf`.

```c++
// scanf 读取
void scanf_read(){
  freopen("data.txt", "r", stdin);
  for (i=0; i<MAXN; i++){
    scanf("%d",&data[i]);
  }
}
```

### cin~sync

接下来是取消同步后的 `cin`:

```c++
// 取消 ios::sync_with_stdio()
void cin_read_nosync(){
  freopen("data.txt", "r", stdin);
  ios::sync_with_stdio(false);
  for (i=0; i<MAXN; i++){
    cin >> data[i];
  }
}
```

### cin~tie

取消绑定:

```c++
// 取消 cin.tie()
void cin_read_notie(){
  freopen("data.txt", "r", stdin);
  cin.tie(NULL);
  for (i=0; i<MAXN; i++){
    cin >> data[i];
  }
}
```

### cin~tie~sync

同时需要同步与绑定:

```c++
// 同时取消sync 与 tie
void cin_read_nosync_notie(){
  freopen("data.txt", "r", stdin);
  ios::sync_with_stdio(false);
  cin.tie(NULL);
  for (i=0; i<MAXN; i++){
    cin >> data[i];
  }
}
```

### ifstream

接下来是直接使用 `ifstream` 从文件读取, `ifstream` 会自动解析字符串并转换为整数型:

```c++
// ifstream 读取
void ifstream_read(){
  ifstream inf("data.txt");
  for (i=0; i<MAXN; i++){
    inf >> data[i];
  }
}
```

### fread 与 read

接下来是先将整个文件当作 RAW 字符串读进内存, 再手动对字符串进行解析, 将数据转换为整数存进数组:

```c++
//先fread读取字符串再parse
void fread_parse(){
  freopen("data.txt", "rb", stdin);
  int len = fread(buf, 1, MAXS, stdin);
  buf[len] = '\0';
  parse(buf);
}
//先read读取字符串再parse
void read_parse(){
  int fd = open("data.txt", O_RDONLY);
  int len = read(fd, buf, MAXS);
  buf[len] = '\0';
  parse(buf);
}
```

`read` 是比 `fread` 更底层的函数, `使用read` 也许比 `fread` 更加快.

### mmap

mmap 是linux 的函数, 直接将文件映射进内存:

```c++
// mmap 读取 
void mmap_parse(){
  int fd = open("data.txt",O_RDONLY);
  int len = lseek(fd,0,SEEK_END);
  char *mbuf = (char *) mmap(NULL,len,PROT_READ,MAP_PRIVATE,fd,0);
  // buf = mbuf
  parse(mbuf);
}
```

### pascal

在竞赛中, 例如 ACM 有很多人使用 `pascal` 语言. 并认为 `pascal` 的读取速度要比 `C / C++` 快, 所以也对 `pascal` 的读取进行测试.
首先用 `pascal` 实现读取文本并加载进数组.

```pascal
const
    MAXN = 10000000;
var
    numbers :array[0..MAXN] of longint;
    i :longint;
begin
    assign(input,'data.txt');
    reset(input);
    for i:=0 to MAXN do
        read(numbers[i]);
end.
```

然后将 `pascal` 代码编译后, 再在 `C++` 中运行 `linux` 系统命令进行测试:

```c++
//pascal
void pascal(){
  system("./load_data");
  i = MAXN;
}
```

### 其他函数

其他函数包括, 字符串的解析函数, 将字符串解析为整数存进数组.

```c++
//parse 函数
void parse(char *buf){
  i=0;
  data[i] = 0;
  for (char *p=buf; *p != '\0'; p++)
  {
    if (*p == '\n' && *(p+1) != '\0')
      data[++i] = 0;
    else
      data[i] = data[i] * 10 + *p - '0';
  }
  i++;
}
```

评估函数, 对各个读取方法的函数的评估, 并在评估过程中打印结果. 为保证公平, 对每种方法分别评估 ITER 次, 最后再将结果取平均并返回.

```c++
// 评估函数
double evaluate(void (*func)()){

  double time_use, totaltime = 0;
  struct timeval start;
  struct timeval end;
  /*
  struct  timeval{
  long  tv_sec;  // 秒
  long  tv_usec; // 微妙
  };
  */

  for (int it=0; it<ITER; it++)
  {
    gettimeofday(&start,NULL); // start time
    func();
    if (i == MAXN)
      cout << "Iter:" << setw(2) << setfill(' ') << it+1 << "\tState: success\t";
    else
    {
      cout << "Error";
      break;
    }
    gettimeofday(&end,NULL); // end time
    time_use = end.tv_sec - start.tv_sec + (end.tv_usec - start.tv_usec)/1000000.0;
    totaltime += time_use;
    cout << "Time: " << setprecision(5) << fixed << time_use << endl;
  }
  return totaltime/ITER;
}
```

下面是生成数组的函数, 与重写的输出函数.

```c++
// 生成随机数到 data.txt
void generate(){
  default_random_engine e;
  ofstream of("data.txt");
  for(int i=0; i<MAXN; ++i)
    of<<e()<<'\n';
}

// 重写 << 输出 result
ostream& operator<< (ostream& sm, map<string, double>& m) {
  sm << setw(45) << setfill('-') << "Result:" << endl;
  for (auto v : m){
    sm << "Method: " << setw(12) << left << setfill(' ') << v.first;
    sm << "Average Time: " << v.second << endl;
  }
  return sm;
}
```

### main 函数

最后实现 `main` 函数, 通过 vector 控制待评测的函数.

```c++
vector<string> evaluate_methods = {
  "cin",
  "scanf",
  "ifstream",
  "cin~sync",
  "cin~tie",
  "cin~tie~sync",
  "fread",
  "read",
  "mmap",
  "pascal"
};
```

```c++
// Main
int main(){

  {
  ifstream fin("data.txt");
  if(!fin)
    {
      cout << "Generating 'data.txt' !! " << endl;
      generate();
    }
  } // RAII

  map<string, double> res; // save results

  for (auto m : evaluate_methods)
  {
    cout << "Start Evaluate: " << setw(25) << setfill('-') << m << endl;
    res[m] = evaluate(methods[m]);
  }
  cout << res << endl;
  // evaluate(methods["test1"]);
  return 0;
}
```

## 实验结果

| 方法         | 平均时间(s) |
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
