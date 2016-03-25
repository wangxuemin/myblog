---
title: Thrift 的TNonblockingServer运行原理分析
date: 2016-01-25 06:59:24
tags:
- thrift
- TNonblockingServer
categories:
- thrift
---

整理下thrift TNonblockingServer的工作流程，简单记录下， 因为处理过程比较复杂不具体分析了，
TNonblockingServer的工作流程如下：
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/thrift.png) 

--1-- server 创建 多个 iothread 和  工作线程池 workpool  thread (给iothread发消息的方式主要通过管道的方式进行 )
--2-- 传递listenfd 给 iothread 的 number 为0的线程，该线程监听listen socket的accept事件
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/thrift-2.png) 

--3-- 当0号iothread 监听到accept事件时， 创建connection 并交给相应的iothread处理数据收发(通过管道方式通知相应的iothread)
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/thrift-3.png) 
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/thrift-4.png) 

--4-- iothread收到新的connection 根据 connection的状态 进行数据收发等处理 ，该逻辑由conection的transition()
函数完成, 第一部肯定是请求报文的read操作
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/thrift-5.png) 

--5-- 当connection的一个请求数据read完成时, 封装成任务task交由workpool  thread 的线程池处理
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/thrift-6.png) 

--6-- workpool  thread 任务线程处理完成后会调用notifyIOThread通知connection对应的iothead来发送结果给客户端
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/thrift-7.png) 

--7-- iothread将处理完成结果发送给客户端, 一个请求处理完成.
TNonblockingServer.py的设计思路跟CPP基本是一样的，可以参考着看看~~

转载请注明出处，谢谢。。
