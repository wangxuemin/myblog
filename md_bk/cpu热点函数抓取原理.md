---
title:  CPU热点函数抓取原理
date: 2020-06-27 00:17:37
tags:
- Linux
- 动态跟踪技术

categories:
- linux
---


cpu热点抓取原理，怎么才能知道是进程的哪一个函数消耗了cpu资源呢？目前gperftools,async-profile,perf 都针对不同的语言提供了抓取cpu热点函数的功能，他们抓取的原理都很类似，如果不依赖内核支持的话，简单来说就是在用户空间设置一个timer定时器，timer以一定的频率向进程发送信号，在信号处理函数中可以拿到进程正在执行的调用栈，将采集到这些调用栈统计分析一下，数量最多的那个及时占用cpu最高的热点函数
gperftools的实现代码在[profiler.cc](https://github.com/gperftools/gperftools/blob/51b4875f8ade3e0930eed2dc2a842ec607a94a2c/src/profiler.cc), 中定时器回调函数处理流程调用栈如下图所示:

```c
//    GetStackTrace() and store in the Bucket
//             |
//    ProfileData::Add(unsigned long pc)   //在信号处理函数中获取当前的调用栈地址信息并存储
//             |
//    ProfileData::prof_handler()     // SIGPROF信号处理函数
//             |
//    < SIGPROF signal >      

```
在profile.cc中生成的CPUPROFILE只是存储了调用栈的二进制地址信息. 至于对应符号表的对应关系则是通过pprof这个perl脚本来实现
里面用到了nm/objdump/addr2line等一些工具来帮助找到地址和函数名称的对应关系

async-proflie的itimer引擎使用了跟gperftools一样的原理，代码更加简单和清晰:
```c
#include <sys/time.h>
#include "itimer.h"
#include "os.h"
#include "profiler.h"

long ITimer::_interval;

// SIGPROF信号处理函数，在函数中获取进程的当前调用栈并存储
void ITimer::signalHandler(int signo, siginfo_t* siginfo, void* ucontext) {
    Profiler::_instance.recordSample(ucontext, _interval, 0, NULL);
}

Error ITimer::start(Arguments& args) {
    if (args._interval < 0) {
        return Error("interval must be positive");
    }
    _interval = args._interval ? args._interval : DEFAULT_INTERVAL;

    OS::installSignalHandler(SIGPROF, signalHandler);

    long sec = _interval / 1000000000;
    long usec = (_interval % 1000000000) / 1000;
    struct itimerval tv = {{sec, usec}, {sec, usec}};
    setitimer(ITIMER_PROF, &tv, NULL);
    //设置timer timer 到期后向当前进程发送SIGPROF信号
    return Error::OK;
}

void ITimer::stop() {
    struct itimerval tv = {{0, 0}, {0, 0}};
    setitimer(ITIMER_PROF, &tv, NULL);
}
```

原理比较清楚了，但是async-profile默认的引擎和perf工具则是依赖perf_event来抓取热点的并没有使用timer.
perf是一个功能强大的性能统计和分析工具 https://perf.wiki.kernel.org/index.php/Tutorial
perf_event是perf相关的一个系统调用,由内核提供给进程使用功能强大，其中的抓取cpu热点分支相对于上述timer方式存在下面几个优点:
1. 由硬件和内核触发，更加精确 在最初版本中可以看到当前运行函数的调用栈由intel_pmu_handle_irq（）触发，This handler is triggered by the local APIC
2. 因为代码在内核中，抓取热点函数调用栈非常的高效，对应用程序性能几乎没有影响
3. 不同于gperftools和async profile需要再目标程序中加载额外代码， perf_event对于目标程序没有任何入侵性。
4. perf_event可抓取的信息非常丰富，cpu热点只是其中之一






转载请注明出处，谢谢。。
