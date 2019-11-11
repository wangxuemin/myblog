---
title: Epoll 新增 EPOLLEXCLUSIVE 选项解决了新建连接的’惊群‘问题
date: 2016-01-25 08:15:03
tags:
- epoll
categories:
- linux
---



epoll最终和accept一样解决了新建连接的惊群问题 patch地址：
 https://github.com/torvalds/linux/commit/df0108c5da561c66c333bb46bfe3c1fc65905898
patch比较简单， 下面摘录了一部分关键修改~~
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/epoll-exculsive-1.png) 
 <!-- more --> 
 
在加入listen socket的sk_sleep队列 的唤醒队列里使用了 add_wait_queue_exculsive()函数，  当tcp 收到 三次
握手最后一个 ack 报文时调用sock_def_readable时，只唤醒一个等待源， 从而避免’惊群‘.
调用栈如下：
```c
//  tcp_v4_do_rcv()
//  -->tcp_child_process()
//  --->sock_def_readable()
//  ---->wake_up_interruptible_sync_poll()
//  ----->__wake_up_sync_key()
```

相关链接:
https://github.com/torvalds/linux/commit/df0108c5da561c66c333bb46bfe3c1fc65905898
https://lwn.net/Articles/667087/
http://netdev.vger.kernel.narkive.com/RWScCeSc/patch-net-next-epoll-add-epollexclusive-support
https://lwn.net/Articles/632590/

转载请注明出处，谢谢。。
