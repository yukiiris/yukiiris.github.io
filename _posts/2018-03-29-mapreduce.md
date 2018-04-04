---
layout: post
title: "MIT 6.824 Lab1 MapReduce"
date: 2018-03-22
excerpt: "踩了一万个坑才成功"
tags: [go]
comments: true
---



## MIT 6.824 Lab1 MapReduce

### Part I: Map/Reduce input and output

+ domap方法的任务：它读入输入文件，对内容调用用户定义的map方法mapF()，然后将mapF的输出分割成nReduce个中间文件。
+ 每个reduce任务都有一个中间文件，用reduceName()生成文件名。为每个键调用ihash()并模除 nReduce，为键值对选择r。
+ mapF()是用户提供的方法，它为reduce返回一个包含一个键值对的切片。
+ doreduce方法的任务：它为任务读出中间文件，通过键为这些中间键值对排序，为每个键调用用户定义的reduceF方法，将结果写入硬盘。
+ 你必须为每个map任务读出中间文件，reduceName方法为每个map任务产生文件名。
+ 你的domap方法将中间文件的键值对进行了编码，所以你需要解码。
+ reduceF方法由用户提供，你必须为每个不同的键以及它的所有值调用它。这个方法返回这些值应该对应的键。

### Part II: Single-worker word count

+ mapF()方法将每个传入的内容分割为单词，用单词作为键，为每个单词创建一个KeyValue，值为"1”，表示这个单词出现了一次，将所有KeyValue作为一个切片返回。
+ redecuF()方法传入一个键和键对应的值数组，这个方法需要将所以mapF()中统计的次数相加，返回值数组的长度即可。


### Part III: Distributing MapReduce tasks

+ schedule()在所给的阶段（map或reduce）启动和等待所有任。mapFIles参数决定map阶段输入文件的名字。nReduce参数是reduce任务的数量。

+ 所有ntasks个任务都会在worker中被调度，当所有任务都成功结束，schedule()会返回。

+ master线程在mapreduce中调用两次schedule()，map阶段一次，reduce()阶段一次。这个方法的任务是，把任务交给可用的工作线程。因为任务一般比工作线程多，所以schedule()方法要给每个工作线程一系列任务。

+ schedule()方法从rigisterChan参数中知道有多少可用的工作线程。这个channel为每个worker产生一个字符串包含它的RPC地址。一些worker在schedule()方法被调用之前已经存在，另一些会在这个方法运行时生成，schedule()必须将这些worker充分利用。

+ schedule()通过发送一个Worker.DoTask RPC给worker来告诉它要执行任务。

  这里踩了好几个坑，说明一下。一个是关于sync.WaitGroup的add和Done两个方法，add只能在主线程里面用，想要同步线程就要在其他线程里用Done。然后将worker从registerChan中读出来并结束一个任务以后要放回去（来自一个没怎么学go并发编程的兄弟）


### Part IV: Handling worker failures

在这个环节中，你需要处理失败的线程。

一个RPC调用失败不意味着这个线程没有在执行这个任务，这可能是因为返回值丢失或者master线程的RPC调用超时了。因此，这可能会发生于两个worker收到相同任务，处理他，然后生成输出。两个map或reduce调用必须对同一个输入生成相同的输出，所以如果后续处理有时读取一个输出并且有时读取另一个输出，则不会有不一致。除此之外，MapReduce框架确保map方法和reduce方法自动输出：输出文件可能不存在也可能包含map或reduce方法的单个执行的全部输出。