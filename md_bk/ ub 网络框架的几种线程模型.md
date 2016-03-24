---
title:  ub 网络框架的几种线程模型
date: 2015-08-27 00:17:37
tags:
- 网络
- ub
- 线程模型

categories:
- linux
---

ub是公司不错的网络框架, 使用C语言开发，清晰易懂，不像sofa-rpc使用c++ 开发，语言层面的技巧较多.
个人还是喜欢ub的简单. 本文通过ub框架介绍一下server端开发的常见的几种线程模型.

ub包含5种线程模型，我们挑选了三个比较典型和简单的来讲解一下
```
xpool   \\ 最简单同步模型

cpool    \\ 生产者消费者模型

appool  \\ 异步模型
```

 xpool最简单的线程模型:
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/ub_1.png) 
 <!-- more --> 
在xpool连接模型中，主线程创建listen socket, 多个工作线程同时accept该listen socket竞争一个新的
连接, 拿到连接的线程就进行IO读写和业务逻辑处理. 为了避免惊群现象，多个线程的accept会加锁处理
(互斥锁mutex)。  xpool一般认为在连接较少时候效果比较好,但如果同一时候连接数过多会造成没有工作线
程与客户端进行连接, 客户端会出现大量的连接失败.这种模型的最大优点在于编写简单，目前使用的较少.

cpool生产者消费者模型:
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/ub_2.png) 
cpool是一种生产者消费者模型, 是对xpool的很大改进,也是我们现在许多服务比较常用的模型. 在cpool连接模
型中，由一个线程去accept新的连接，然后将新的连接FD放入等待队列(这里有个特别的地方就是当新建的连接
有数据到来时才放入等待队列)， 多个工作线程从这个队列里取出新连接的FD, 进行I/O读写和逻辑处理操作. 在
大压力下队列有一定的缓冲作用，虽然有些请求会出现延时, 很少出现像xpool那样出现连接失败的问题.

xpool/cpool本质都是同步的处理业务逻辑,在一个线程中处理了读请求,逻辑处理和发送结果三个过程, 但是读和
发送这两个IO的处理往往会阻塞工作线程, 如果数据收发非常的慢IO阻塞线程时间会很长, 这个时候就造成了线程
的浪费. 可以增加线程数来解决问题,但是过多的线程会导致cpu上下文切换频繁切换.

appool异步模型:    
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/ub_3.png) 
该图为原理图，appool实际处理中I/O线程和工作线程耦合的还是比较严重的,并没与采用双队列的方式.
appool异步模型, 就是用一个线程使用epoll专门进行数据收发, 工作线程只做业务逻辑处理,这样使的工作线程
只处理业务逻辑部分,可以提高CPU的使用率，减少等待的时间. 而I/O线程通过epoll同时处理多个客户端的数据收发
可以应付很大的流量请求.
SSDB跟该模型很类似参见我的另一篇博客。

转载请注明出处，谢谢。。

