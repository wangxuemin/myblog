---
title: FreeBSD Kqueue的实现原理
date: 2015-07-30 10:17:37
tags:
- freebsd
- kqueue
categories:
- freebsd
---


kqueue/epoll 是两个网上出现频率比较高的关键字，epoll实现原理及源码网上已经有很多blog分析，关于
select/poll/epoll、kqueque的优缺点也不再解释。kqueue实现原理的文章网上资料比较少， 基本上就Jonathan
Lemon的一篇论文， Jonathan Lemon也是Kqueue的发明者。文章链接: 

-[Kqueue:A generic and scalable event notification facility](http://people.freebsd.org/~jlemon/papers/kqueue.pdf)-


本文主要说明一下其中的kqueue的实现部分,从网络收发部分看kqueue和epoll的使用方式和实现原理都比较类似.
这篇文章介绍了kqueue的使用方法：使用 kqueue 在 FreeBSD 上开发高性能应用服务器 . kqueue的是主要实现逻辑
在kern_event.c 里面(freebsd 4.1)

在kqueue实现中，比较关键的是一个knote结构体，该结构体在内核空间对应于应用层的kevent结构体.  knote
将事件源(被监控节点tcp/socket), 事件源是否有事件发生和knote所在的kqueue联系起来.  knote之间也有联系这个
后面具体分析.另一个比较关键的数据结构是kqueue自己，包含两个功能：1)包含一个有事件发生需要通知应用
层的knotes队列，也就是已完成事件队列. 2) 保存并跟踪应用层注册的的需要监听的事件和描述符.kqueue有三个
子结构体来实现上面的功能：
      1. 一个队列，用来保存active的knotes节点

      2. 一个小hashtable 用来查找那些没有对应描述符的knotes节点

      3， 一个线性的描述符array，这个array和进程打开的文件描述符表一致

上述的hashtable 和线性数组都是延迟分配的,  存放描述符array可以自动扩展, kqueue必须记录这些所有用户注册
的knotes, 当kqueue被close，属于它的knotes都会被释放. 描述符array可以保证当用用户关闭一个描述符如(socket 
fd）时, kqueue对应的knotes 节点会别释放.
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/kqueue-1.png) 

事件注册：
    最初， 应用程序调用kqueue() 来分配一个新的kqueue(一下简称KQ), 涉及到了分配一个新的kqueue描述符、
kqueue结构体、和一个指向已打开文件描述符table的指针, 这个时候并没有给这个给array和hashtable分配空间.
应用然后调用kevent()传递一个changelist指针(参见kevent使用说明)，changelist中的kevents从用户空间copy到
内核空间, 然后对每一个kevents调用register().  register()先在KQ中查找是否有匹配的knotes, 如果过没有，表明
第一次添加，分配一个新的knotes(有EV_ADD标记).根据传递来的kevent信息对新建的knotes进行初始化，并调
用attacth()将knote连接到事件源(如tcp收包).这个连接操作是通过一个叫filter(不明白为啥叫filter)的attach函数.  
之后将knote添加到KQ的hashtable或array中.如果在处理changelist发生错误， 发生错误的kevent则会copy到evenlist
返回给应用层.当changelist的所有事件都处理成功后 kqueue_scan()才会开始检查是否有active的事件.

Filters：
    每一个filter都包含三个函数{attach, detach, filter}， filter根据事件源类型决定(如tcp/udp/file分别有自己的filter)

      attach() : 将knote加入到要监听的事件源中.
      deattch(): 将knote从某一事件源中取消.
      filter()：当事件源(TCP)有事件发生时，如收到数据会先调用该函数先检查当前事件
                是否需要通知到应用层.  filter()返回boolean类型表明事件是否需要通知应用层.
                kqueue_sacn()也会调用该函数来检测某一事件是否已满足


跟事件源连接，事件源是否有事件发生，发生时通知哪些knotes节点, 都通过这三个函数来实现.

事件源有事件发生：
当有事件发生时（收到一个数据包、文件被修改、一个进程退出），最终会将事件通知给应用层. 收到事件时
事件源会对attach到自己的knotes链表调用knote()函数.knote()扫描所有link到该事件源的knotes，检测事件是否满足
通知条件(filter()函数）.如果事件条件满足则将该knote放入到kqueue的active list队列里，最终会传递给应用层.
投递：

主要将kqueue active list里的事件copy到应用层下面是调用栈示简单示意图(freebsd 4.1 ),以TCP为例:


```cpp
//***************************************************************************    
    
//注册    
kevent()    
  ->kqueue_register()  //注册要监听的TCP的事件    
    ->knote_attach      //分配note结点，并和相对应的socket 连接    
    ->fops->f_attach()==filt_sorattach()  //调用TCP socketfilter里的attach函数    
       ->SLIST_INSERT_HEAD()   // 将knotes结点挂入该socket的knotes 列表    
           
    
//检测是否有event发生    
kevent()    
  ->kqueue_scan()  // 检查是否有时间按发生     
    ->tsleep()      //事件为空则sleep 等待事件通知    
            
    
//TCP 收报流程    
tcp_input()    
  ->sowwakeup()    
    ->sowakeup()        
      ->knote()   //遍历连接到自己的knotes结点list    
        ->kn_fop->f_even()  //事件满足唤醒条件    
        ->knote_enqueue()    //将knote插入kqueue的active队列    
          ->wakeup();   //唤醒kqueue    
    
//************************************************************************  

```
转载请注明出处，谢谢。。

