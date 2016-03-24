---
title:  父进程waitpid子进程的一般实现
date: 2015-08-26 07:59:34
tags:
- 进程
- waitpid
categories:
- linux
---



web后台开发中很多框架都是prefork的进程模型的， 包括 php-fpm / flup / gunicorn 等等由主进程 fork 出
一堆工作进程， 主进程监督工作进程的存活状态和数量，按照需要重启工作进程主进程的主要工作： 
1) 回收僵尸子进程 
2）重启子进程（重启策略看需求），逻辑比较简单，我们以gunicorn 为例 简单看一下代码



--1-- 主进程监听SIGCHLD信号， 创建管道, 调用select睡眠等待管道中的消息
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/waitpid-1.png) 


--2-- 收到信号时，调用waitpid回收僵尸进程，存活进程数减一，向管道中写消息，从而唤醒主进程主循环select操作返回
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/waitpid-2.png) 


--3-- 主进程的主循环中收到信号后 select  返回， 检查是否需要重启子进程， 处理完后继续睡眠


![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/waitpid-3.png) 
gunicorn把SIGCHLD和其他信号区分出来，个人感觉统一起来比较好即： 主进程收到信号后，同一将信号写入队列
并唤醒主循环select调用， select 返回后read出信号类型， 根据信号不通做相应的处理，如果信号为SIGCHLD此时
再进行waitpid和重启子进程的处理.  可以参照下php-fpm(fpm_signals.c\fpm_events.c)的处理:

```c
//注册信号，创建管道
int fpm_signals_init_main()  

void fpm_event_loop(int err)
{
   //将管道读时间加入到事件循环中
   //fpm_event_set(&signal_fd_event, fpm_signals_get_fd(), FPM_EV_READ, &fpm_got_signal, NULL);   
   //ret = module->wait(fpm_event_queue_fd, timeout);  //等待事件发生
}

//信号管道可读的的回调,读取事件类型并作相应的处理
static void fpm_got_signal(struct fpm_event_s *ev, short which, void *arg) 
```
转载请注明出处，谢谢。。
