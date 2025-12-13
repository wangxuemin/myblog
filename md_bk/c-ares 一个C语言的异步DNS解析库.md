---
title: c-ares 一个C语言的异步DNS解析库
date: 2015-05-29 08:17:37
tags:
- c-ares
- dns
categories:
- linux
---

c-ares是一个C语言的异步DNS解析库，可以很方便的和使用者的事件循环统一起来，实现DNS的非阻塞异步
解析，libcurl, libevent, gevent, nodejs都在使用。

下面摘自Stack Overflow的一个例子:
```c

#include <time.h>
#include <iostream>
#include <netdb.h>
#include <arpa/inet.h>
#include <ares.h>
//ares  处理完成，返回DNS解析的信息
void dns_callback (void* arg, int status, int timeouts, struct hostent* host) 
{
    if(status == ARES_SUCCESS)
        std::cout << host->h_name << "\n";
    else
        std::cout << "lookup failed: " << status << '\n';
}
void main_loop(ares_channel &channel)
{
    int nfds, count;
    fd_set readers, writers;
    timeval tv, *tvp;
    while (1) {
        FD_ZERO(&readers);
        FD_ZERO(&writers);
        //获取ares channel使用的FD
        nfds = ares_fds(channel, &readers, &writers);   
        if (nfds == 0)
          break;
        tvp = ares_timeout(channel, NULL, &tv);       
        //将ares的SOCKET FD 加入事件循环
        count = select(nfds, &readers, &writers, NULL, tvp);   
        // 有事件发生 交由ares 处理
        ares_process(channel, &readers, &writers);  
     }

}
int main(int argc, char **argv)
{
    struct in_addr ip;
    int res;
    if(argc < 2 ) {
        std::cout << "usage: " << argv[0] << " ip.address\n";
        return 1;
    }
    inet_aton(argv[1], &ip);
    // 创建一个ares_channel
    ares_channel channel;    
     // ares 对channel 进行初始化
    if((res = ares_init(&channel)) != ARES_SUCCESS) {    
        std::cout << "ares feiled: " << res << '\n';
        return 1;
    }
    //传递给c-ares channal 和 回调
    ares_gethostbyaddr(channel, &ip, sizeof ip, AF_INET, dns_callback, NULL); 
    main_loop(channel);   //主程序事件循环
    return 0;
  }
```
如果有需要可以采用c-ares实现一个自己的DNS异步解析的Server，做成独立的dns解析服务

转载请注明出处，谢谢。。


