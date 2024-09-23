---
title: "Lab-1 MapReduce"
description: 描述Lab1中MapReduce的原理以及实现
date: 2024-09-17
image: matt-le-SJSpo9hQf7s-unsplash.jpg
categories:
    - 分布式
---

# MapReduce: Simplified Data Processing on Large Cluster
## 模型
MapReduce由Map+Reduce两部分构成

### Map
将输入的pair转化为中间的k/v对

![](https://cdn.jsdelivr.net/gh/Messiz/image-bed/test/20240924001534.png)

### Reduce
将中间的k/v对转化为最终的输出

![](https://cdn.jsdelivr.net/gh/Messiz/image-bed/test/20240924001544.png)



### 例子
![](https://cdn.jsdelivr.net/gh/Messiz/image-bed/test/20240924001552.png)



## 实现
### 流程
!![](https://cdn.jsdelivr.net/gh/Messiz/image-bed/test/20240924001601.png)
1. 将输入文件分为M个小块，然后在集群的所有机器上启动程序
2. 用户程序分为master和worker
    1. master负责分发任务
    2. worker负责处理具体工作，例如一个map任务的worker负责从输入数据中提取k/v对，并使用Map函数处理，得到中间k/v对，存储在内存缓冲区
3. 内存缓冲区中的数据周期性地写入磁盘，并且由分区函数分为R个partition，并将结果交给master，master可以将这些位置信息交给负责reduce任务的worker
4. reduce worker接到磁盘上partition的位置信息后，使用RPC来读取这些数据，然后按照key对数据进行排序
5. reduce worker然后遍历已排序的数据，将每一个不同的key以及其对应的value集合传递给Reduce函数进行处理，并输出此分区最终的输出文件
6. 当所有的map和reduce函数完成工作后，master唤醒用户程序，返回数据



### Master
master负责调度worker，因此对于每一个map task和reduce task，都保存了其状态信息（空闲、正在处理、处理完毕），以及对于正在处理或处理完的任务，保存其工作机器的信息



### 容错
由于MapReduce本身设计基于大的机器集群，因此机器故障一定会发生

#### Worker Failure
+ master定期ping各个worker，如果没有回应则认为该worker挂掉，对于出问题的worker，其<font style="color:#DF2A3F;">完成的所有map任务</font>以及<font style="color:#DF2A3F;">正在进行的map或reduce任务</font>，都将被标记为idle状态（未完成）以供重新调度。
+ 之所以完成的map任务需要重新执行，是因为由于该worker联系不上，说明对应的机器也联系不上，而完成的map任务，中间结果是保存在本机的磁盘的，因此需要重新调度执行；而reduce任务不需要是因为其输出是存储于全局的文件系统中。
+ 当A执行map失败而由B重新执行时，所有后续执行reduce的worker都会接到信息，尚未从A读取数据的worker会转而从B读取数据。



#### Master Failure
+ 通过周期性记录checkpoint，当master宕机后，可以从最近的checkpoint恢复



#### Semantics in the Presence of Failures
当map和reduce函数都是确定性函数时，提供与整个程序无故障顺序执行相同的输出。

当map和reduce函数为非确定性函数时，提供较弱但仍然合理的语义。

<font style="color:rgb(5, 7, 59);background-color:rgb(253, 253, 254);"></font>

### <font style="color:rgb(5, 7, 59);background-color:rgb(253, 253, 254);">后备任务</font>
当出现拖后腿的节点时（由于某些原因导致执行慢），会造成MapReduce整体变慢。

当MapReduce操作接近完成时，master会调度空闲节点执行当前剩余的正在执行的任务，当任意节点完成了该任务后，就将其标记为完成。



## 改进
### 分区函数 Partitioning Function
常见的分区函数：hash(key) mod R，哈希后取模

对于某些特殊目标，也可以定制特殊的分区函数



### 排序保证 Ordering Guarantees
对于一个给定的分区，其中键值对按照key按顺序排列



### 组合函数 Combiner Function
某些情况下，map操作产生的中间键存在大量重复，例如对于统计词频的任务，map操作会产生若干<key, 1>的键值对，我们可以引入组合函数，在将其发送至reduce函数前先进行合并，得到<key, n>这样的中间产物。注意，这里的组合函数的实现实际上与reduce函数一致，只是他们的输出不同，组合函数输出仍是中间产物，而reduce函数输出为最终结果。



### 输入输出类型 Input and Output Type
例如对于文本数据，输入kv对，key表示offset，value则为对应行的数据

用户可以自己定义一些输入输出格式



### 副产物 Side-effects
<font style="color:#DF2A3F;">没太懂</font>

<font style="color:#DF2A3F;"></font>

### 跳过错误记录 Skipping Bad Records
跳过错误记录从而避免程序崩溃，能够跳过继续执行



### 本地执行 Local Execution
MapReduce工作在分布式系统中，难以调试，提供了简单的单体执行机制，方便用户调试



### 状态信息 Status Information
Master节点提供任务整体进度信息



### 计数器 Counters
提供了计数器，各worker节点统计一些计数数据，将数据汇总给master节点



## 性能
两个任务上进行了测试

1. 1tb数据排序
2. 1tb数据搜索

