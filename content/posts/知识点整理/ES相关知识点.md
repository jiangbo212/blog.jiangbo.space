---
title: "ES相关知识点"
date: 2022-02-07T1549:14+08:00
draft: false
categories: 知识点整理
keywords: ES Elastic 面试 索引 translog segment flush fsync Replica
---
+ ### es索引文档过程

1. es节点写入请求过程。write→refresh→flush→merge 

2. es节点接受新的请求(写入数据)时，会首先将数据写入内存缓存，同时定时(默认1s)写入到磁盘的缓存区。以上过程称之为refresh。refresh后数据已经进入segment, 此时可以被搜索到。准实时的误差其实就是refresh的刷新时间。

3. 特殊情况下，refresh过程中可能会存在丢失数据，这时候就需要es的translog机制。也就是在收到请求的同时写入到translog中。当磁盘缓存的数据写入到磁盘中时，translog会清除，这个过程为flush。flush的触发时机为定时出发(默认30分钟)或者translog变得太大时(默认512M)

4. refresh过程中，每次会产生一个小的segment。由于是每秒刷新一次，过多的小的segment会影响系统性能，因此 es会不断的通过定时任务去合并小的segment。此过程可称之为merge。

+ ### es的translog机制
  
  flush过程中，内存中的缓存也会被清理，内容会被写入一个新段(segment), 段的fsync将创建一个新的提交点并将内容刷新到磁盘，旧的translog将被删除并重新开始一个新的translog。translog本质是一个日志文件，通过检查点来进行日志恢复。translog默认为每个请求时候进行一次fsync写入磁盘，这个可以强一致性的保证数据不丢失,但如此会影响性能. 如果对数据丢失有一定容忍，那么可以配置async，对其进行异步刷新。

+ es中的Replica(副本)机制
  
  es包含一个主分片及多个副本分片，由此来保证分布式。索引文档时，写入主片及多个副本分片(副本分片为并发写入)；读取时，只需其中一个返回即可返回结果。

+ 