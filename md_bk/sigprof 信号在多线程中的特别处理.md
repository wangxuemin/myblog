---
title:  sigprof 信号在多线程中的特别处理
date: 2020-06-26 00:17:37
tags:
- Linux
categories:
- linux
---


通常信号处理器是进程级别（process-specific）的，一般是进程中某一个线程来处理所发送的信号, sigprof 多用
于程序性能监控和分析，由 sigprof 定时器产生的信号在多线程程序中是如何被处理的呢,这个由单一线程来做肯
定不合理.用下面一个程序来测试下，进程启动了一个大约 100Hz 频率的定时器来发送sigprof信号。
分为两步：
只使用主线程，统计 10 秒内捕获到的信号数量；
使用 2 个独立线程，每个线程各自运行 10 秒，并统计各自捕获到的信号数量。
```c
#include <signal.h>
#include <stdio.h>
#include <pthread.h>
#include <sys/time.h>
#include <string.h>
#include <time.h>
#include <unistd.h>

enum { THREAD_COUNT = 2 };

volatile sig_atomic_t signal_count[THREAD_COUNT];
pthread_t threads[THREAD_COUNT];

static double now_sec(void)
{
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec + ts.tv_nsec / 1e9;
}

static void sigprof_handler(int sig, siginfo_t *info, void *ucontext)
{
    for (int i = 0; i < THREAD_COUNT; i++) {
        if (pthread_equal(threads[i], pthread_self())) {
            signal_count[i]++;
            return;
        }
    }

    signal_count[0]++;
}

static void install_signal_handler(void)
{
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));

    sa.sa_sigaction = sigprof_handler;
    sa.sa_flags = SA_SIGINFO | SA_RESTART;
    sigemptyset(&sa.sa_mask);

    if (sigaction(SIGPROF, &sa, NULL) != 0) {
        perror("sigaction");
    }
}

static void busy_wait(int seconds)
{
    double start = now_sec();
    while (now_sec() - start < seconds) {
    }
}

static void *thread_work(void *arg)
{
    busy_wait(10);
    return NULL;
}

int main(void)
{
    install_signal_handler();

    struct itimerval timer;
    memset(&timer, 0, sizeof(timer));

    timer.it_interval.tv_usec = 1000000 / 100; /* 100Hz */
    timer.it_value = timer.it_interval;

    for (int i = 0; i < THREAD_COUNT; i++) {
        signal_count[i] = 0;
    }

    if (setitimer(ITIMER_PROF, &timer, NULL) != 0) {
        perror("setitimer");
        return 1;
    }

    busy_wait(10);
    printf("signals caught after 10 seconds: %d\n", signal_count[0]);

    for (int i = 0; i < THREAD_COUNT; i++) {
        signal_count[i] = 0;
    }

    printf("creating %d threads...\n", THREAD_COUNT);

    for (int i = 0; i < THREAD_COUNT; i++) {
        pthread_create(&threads[i], NULL, thread_work, NULL);
    }

    for (int i = 0; i < THREAD_COUNT; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("singals caught after 10 seconds \n");
    for (int i = 0; i < THREAD_COUNT; i++) {
        printf("\t thread %d: %d\n",
                i, signal_count[i]);
    }

    return 0;
}

```

```c
signals caught after 10 seconds: 999
creating 2 threads...
singals caught after 10 seconds
	 thread 0: 1025
	 thread 1: 969
root@xmw-cloudcone:~#

```

实验结论可以看出每一个线程都会处理 sigprof 信号，Linux 这么设计，是刻意为了 profiling 工具包含gprof，
perf（早期用户态），采样型 profiler，采样当前真正消耗 CPU 的执行点如果 sigprof 总是打到主线程：多线程
程序的 profile 会完全失真大体翻了下内核的代码，发送SIGPROF信号的执行路径如下：

```c
timer interrupt
  └── tick_sched_timer()
        └── update_process_times()
              └── account_user_time() / account_system_time()
                    └── check_process_timers()
                          └── __group_send_sig_info(SIGPROF)
```
在两个thread都busy的情况下
CPU0 → thread 0 → check_process_timers(0)
CPU1 → thread 1 → check_process_timers(1)

转载请注明出处，谢谢。。
