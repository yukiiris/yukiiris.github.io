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

