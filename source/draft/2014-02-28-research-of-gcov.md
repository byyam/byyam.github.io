---
layout: post
title:  "Research of gcov"
date:   2014-02-28 00:00:00
categories: gcov
---

## 编译安装合适的gcc版本

下载gcc包
`gcc-version-xx.tar.bz2`

解压
`tar -jxvf gcc-version-xx.tar.bz2`

进入目录
`cd gcc-version-xx`

执行configure试一下看缺少了什么依赖
`./configure`

(如本次缺少两个库，`mpfr` 和 `gmp`，可以用软件管理器安装，也可以自己添加库。

下载这两个包，解压放在gcc-version-xx目录，并且把目录的版本号删掉，名字改为： mpfr 和 gmp )
再试一下是否生成了Makefile文件

`../gcc-version-xx/configure --prefix=yourdir/gcc-version-xx/`

## 编译包含flush gcov函数的库：静态库与共享库的区别

> 动态库（-fprofile-arcs -ftest-coverage可能是第二步不必要的选项）


```
gcc -c -fPIC libats.c -fprofile-arcs -ftest-coverage

gcc -fprofile-arcs -ftest-coverage -shared -o libats.so libats.o

gcc -fprofile-arcs -ftest-coverage test.c -g -L. -lats -Iats -o test
```

运行期间刷出的gcda只是libats的


> 静态库


```
gcc -c libats.c -fprofile-arcs -ftest-coverage

ar -cr libats.a libats.o 

gcc -fprofile-arcs -ftest-coverage test.c -g -L. -lats -Iats -o test
```

运行期间刷出的gcda是libats和test全部的


this program is not .so
flush gcov的范围是this program

readelf -l objectfile
动态链接库是有自己的program header
被编译进来或者静态库没有program header

> 结论

包含libgcov.a并在运行期间调用flush_gcov函数，只能在静态库中操作，因为动态库有自己的program header。

除此之外，一般的函数静态或动态编译并没本质区别。只是加载时期不同而已。


## 编译处理过程

> 预处理 -E

程序员代码->编译器识别的代码
```
[root@Test gcov]# gcc -E test_merge_gcda.c -o test_merge_gcda.i
```

 
> 编译 -S

代码->汇编代码
```
[root@Test gcov]# gcc -S test_merge_gcda.i 
```

 > 合并以上两步

```
gcc -S -ftest-coverage -fprofile-arcs test.i
```

> 汇编 -c

代码->机器码

```
[root@Test gcov]# gcc -c test_merge_gcda.s -o test_merge_gcda.o
```

> 链接

机器码->可执行文件
```
[root@Test gcov]# gcc -o test_merge_gcda test_merge_gcda.o
```

> 合并以上两步

```
gcc -o test_merge_gcda test_merge_gcda.o -lgov
```


```
[root@Test gcov]# ls
test_merge_gcda  test_merge_gcda.c  test_merge_gcda.i  test_merge_gcda.o  test_merge_gcda.s
```

## gcc coverage理解

#### 生成gcno的原理？
在汇编代码插入gcov的一些计数器，链接生成可执行文件时，生成gcno文件。

#### 在哪插入？
每个BB后面会插入一个，具体说应该是连通各个的地方。(但是逻辑图和实际图不一致，因此stub并不等于arc)

#### 何为BB？
basic block， 基本块。
每个基本块只有一个入口，一个出口，因此是原子的。
同一行代码可以同时属于多个BB。

#### 生成gcda的原理？
执行时，被插入的代码会记录每个arc被运行的次数（counter），程序执行结束，最后一个被插入代码会把counter信息写入gcda文件。

#### 生成gcov的原理？
根据BB和分支记录的原始信息，和运行时的counter信息（或加之源文件），算出每行被执行的次数。

#### gcov记录的信息？
BB和分支。

#### gcda记录的信息？
执行时每个arc被运行的次数，推算出每个BB执行的次数，从而算出每个函数执行次数。

如何分析代码产生BB，即确定插桩stub位置？
此问题即对应bb图和源码行，从二维到一维的关系。


#### gcno中block，arcs，line？
实际BB，虚拟BB。
虚拟BB包括start和end，中间也可能存在虚拟BB。

LINE数目为真实BB的个数，记录的是每个BB在哪个文件所对应的行数。（因为有可能编译的时候包括进来其他文件）
ARCS数目为所有BB个数-1，即每个有出度的BB，所指向的BB号码。

每个ARCS长度为（1+出度*2），表示的是，自身BB标号+指向BB标号和?
（源码里是dest和flag）


#### 如果根据gcda记录的 arc counter信息确定每个bb执行次数？
arc counter的个数为插桩点的个数，从源码线性确定每个分段被执行过的路径。

#### 何为branch？
传统理解的分支，一个逻辑块。

[我的代码实现](https://github.com/byyam/yamcov)
