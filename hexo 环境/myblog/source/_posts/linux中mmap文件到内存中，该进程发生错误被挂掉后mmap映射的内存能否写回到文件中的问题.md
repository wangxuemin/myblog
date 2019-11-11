---
title: linux中mmap文件到内存中，该进程发生错误被挂掉后mmap映射的内存能否写回到文件中的问题
date: 2015-07-24 02:17:37
tags:
- linux
- mmap
categories: linux
---

在Tokyo Cabinet中hashDB中的hash索引是通过mmap将数据库文件的一部分映射到内存中的，之前把Tokyo Cabinet 移植到手机淘宝客户端当做一个通用的KV数据库来使用，因为各种手机的环境千差万别，手淘某些机型中得crash率
很高. Tokyo Cabinet数据库文件总是不完整.
因为是手机客户端又不方便像在server端一样使用一个单独的线程定时同步mmap内存到文件中去！手机淘宝的突然
crash是导致mmap的数据不能及时写到文件，使得数据库文件被破坏的原因吗？

突然想到一个问题假设是进程中得其他lib引起crash，当crash时Tokyo Cabinet使用mmap打开的文件在内存中
的修改能否同步到硬盘中呢..搜了一些资料基本得出一个结论：

```
进程崩溃时，mmap的内存内核是会帮你写回到磁盘的
```


参照：
http://stackoverflow.com/questions/5902629/mmap-msync-and-linux-process-termination 
I found a comment from Linus Torvalds that answers this question：

http://www.realworldtech.com/forum/?threadid=113923&curpostid=114068
The mapped pages are part of the filesystem cache,which means that even if the user process that 
made a change to that page dies, the page isstill managed by the kernel and as all concurrent accesses
to that file will go through the kernel, other processes will get served from that cache. In some old
Linux kernels it was different, that's the reason why some kernel documents still tell to force msync.

从内核代码中能基本看到这部分的处理逻辑
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/mmap-question-1.png) 
进程收到信号异常终止是会调用do_exit()释放资源
do_exit() 里面有ummap的调用
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/mmap-question-2.png) 
mongodb貌似也试用了mmap. 总感觉mmap不太可靠.还是建议mmap使用者显示的调用msync进行文件的同步, 不要
过分依赖内核的逻辑吧

转载请注明出处，谢谢。。


