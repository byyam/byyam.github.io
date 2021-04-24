---
layout: post
title:  "Job scheduling"
date:   2017-12-24 23:00:00
categories: algorithms
---
**Job scheduling**

**作业调度**

## 场景

多个作业进程（或服务器，后称为process）可处理相同的任务，并具有不同的处理能力。

### 假设

进程（服务器）|处理能力
------------|------
A|4
B|3
C|2

在一轮任务（Job）的分配中，均衡的分配给每一个process。

处理能力可认为是分配权重weight。

## 算法

### 加权轮询调度

```
Supposing that there is a server set S = {S0, S1, …, Sn-1};
W(Si) indicates the weight of Si;
i indicates the server selected last time, and i is initialized with -1;
cw is the current weight in scheduling, and cw is initialized with zero; 
max(S) is the maximum weight of all the servers in S;
gcd(S) is the greatest common divisor of all server weights in S;

while (true) {
    i = (i + 1) mod n;
    if (i == 0) {
        cw = cw - gcd(S); 
        if (cw <= 0) {
            cw = max(S);
            if (cw == 0)
            return NULL;
        }
    } 
    if (W(Si) >= cw) 
        return Si;
}
```

输出的结果是`AABABCABC`

这个算法的两个技巧：

1. gcd最大公约数：相当于算法的步长。比如权重为1000，100，10的三个process，分配时可以认为它们是权重为100，10，1.最大公约数为10，循环执行10次的效果是一样的。
2. 当W(Si) >= cw时，才返回Si。这样做的效果是，不同process中优先调用高于其他weight的部分，当所有process剩下的weight相等时，按顺序循环调用。


### 实现代码
[python实现](https://gist.github.com/byyam/6fe49cc9861b9134025c8234517f778a)


### 更为随机的加权轮询调度

设想`A，B，C`的权值分别为`5，1，1`的情景下，以上的调度算法输出为`AAAAABC`

定义：

1. 每个process的动态当前权值current weight为cw
2. 每个process的固有权值weight为w
3. 参加轮询的process的weight之和为sw

算法：

1. 每个cw初始值为0
2. 每个cw = cw + w
3. 被选中的process为cw最大的，被选中后cw的值被惩罚为该process的cw减去sw，而其他process的cw保持不变。即max(cw) - sw
4. 若所有process的cw都为0，则一轮结束。否则，进行下一步
5. 跳到第2步

运算：

奇数行根据最大的current weight选择process

被选中的cw被高亮标出

A|B|C
---|---|---
0|0|0
`5`|1|1
-2|1|1
`3`|2|2
-4|2|2
1|`3`|3
1|-4|3
`6`|-3|4
-1|-3|4
4|-2|`5`
4|-2|-2
`9`|-1|-1
2|-1|-1
`7`|0|0
0|0|0

技巧：

1. 算法里循环执行的`cw = cw + w`和`max（cw）-sw`两个步骤确保了cw之和等于sw，循环次数为`sw`次
2. 和上一个算法相比，该算法每次让被选出来的process尽可能的排在最后面，确保了不会有连续的被选中


### 参考资料

加权轮询调度算法：

[http://kb.linuxvirtualserver.org/wiki/Weighted_Round-Robin_Scheduling](http://kb.linuxvirtualserver.org/wiki/Weighted_Round-Robin_Scheduling)