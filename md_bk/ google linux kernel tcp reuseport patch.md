---
title: google linux kernel tcp reuseport patch
date: 2015-07-25 08:17:37
tags:
- tcp
- reuseport
categories: tcp/ip
---
This patch implements so_reuseport (SO_REUSEPORT socket option) forTCP and UDP. For TCP,so_reuseport 
allows multiple listener socketsto be bound to the same port.  In thecase of UDP, so_reuseport allows 
multiple socketsto bind to the same port.  To prevent port hijacking all sockets bound to the same port 
using so_reuseport must have the same uid.  Received packets are distributed to multiple sockets bound 
to the same port using a 4-tuple hash.

该补丁为TCP/UDP增加了了so_reuseport（SO_REUSEPORT socket选项), so_reuseport允许不同的listen
socket绑定同一个端口号(TCP)，不同的socket绑定同一个端口号(UDP),为了防止端口劫持绑定同一端口号的这
些listen socket必须有相同的uid,当数据到来时究竟选择哪一个listen socket是通过一个4元组的hash来实
现的，尽量保证各个listen socket均衡收到请求

The motivating case for so_resuseport in TCP would be something like a web server binding to port 80 
running withmultiple threads, where each thread might have it's own listener socket.  This could be 
done as an alternative toother models: 1) have one listener thread which dispatches completed connections
to workers. 2) accept on a singlelistener socket from multiplethreads.  In case #1 the listener threadcan
easily become the bottleneck with high connection turn-over rate. In case #2, the proportion of connections
accepted per thread tends to be unevenunder high connectionload (assuming simple event loop:while (1) { accept();
process() }, wakeup does not promote fairness among the sockets. We have seen the  disproportion to be as high
as 3:1 ratio between thread accepting most connections and the one acceptingthe fewest.  With so_reusport the 
distribution is uniform.

TCP的so_resueport主要使用在高负载的webserver领域，webserver一般有多个线程，这些线程都需要使用80
端口号, 以前这些线程的listen socket是没有办法同时绑定80端口号的，只能用其他的一些方法：1）使用一个
线程去监听80端口号，将已完成的连接分发给各个工作线程去处理(memcache) . 2）多个线程使用同一个listen
socket 

第一种方式在高并发的情况下监听线程很容易成为瓶颈. 第二种方式在高并发下每个线程accept的连接数目会变的
很不均匀,假设我们线程的处理逻辑很简单
```cpp
while(1){
        accept();   //线程阻塞在accept，等待底层唤醒接收新的连接, 因为
                    // 大家使用的同一listen socket 只有一个进程被唤醒后能
                    //成功接收新的连接

        process();

}
``` 

每个连接被唤醒的概率是不公平的，我们见过最多的线程和最少的之间接收的连接数目相差3倍,使用reuseport后这些
线程将会变得非常均匀

补丁diff文件：
https://gist.github.com/fcicq/3993833
http://patchwork.ozlabs.org/patch/50430/

上面是补丁的介绍，除了扩展一些数据结构外主要更改了:
1）bind逻辑   如果设置了so_reuseport 选项，放开对同一端口绑定的限制,
2） 另外就是TCP/UDP的收报流程，如TCP的SYN建连报文来时，需要在这些绑定的相同的端口的 listen socket 均衡
的选择一个合适的，下面代码增加了查找相同端口号的listen socket 的逻辑:
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/google-reuseport-1.png) 

转载请注明出处 谢谢~~
