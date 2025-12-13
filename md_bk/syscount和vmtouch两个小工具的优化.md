---
title:  syscount和vmtouch两个小工具的优化
date: 2022-12-15 07:59:34
tags:
- liunx
- 文件
- page cache
- perf


categories:
- linux
---


使用syscount这个perf-tools 跟踪某一进程的系统调用情况时，并不支持-d 选项如下：
```c

USAGE: syscount [-chv] [-t top] {-p PID|-d seconds|command}
       syscount                  # count by process name
                -c               # show counts by syscall name
                -h               # this usage message
                -v               # verbose: shows PID
                -p PID           # trace this PID only
                -d seconds       # duration of trace
                -t num           # show top number only
                command          # run and trace this command
  eg,
        syscount                 # syscalls by process name
        syscount -c              # syscalls by syscall name
        syscount -d 5            # trace for 5 seconds
        syscount -cp 923         # syscall names for PID 923
        syscount -c ls           # syscall names for "ls"

```
-d 只支持整个系统的调用统计，非常不方便，简单修改了syscout 支持 syscount -cp 923 -d 10, 支持统计进程
在-d 时间段 所有syscall调用次数的统计：https://github.com/wangxuemin/linux-ftools
效果如下：
```c

➜  perf-tools sudo ./syscout -cp 15572 -d 10
Tracing for 10 seconds...
Tracing PID 15572 for 10 seconds. .. Ctrl-C to end.
/usr/lib/linux-tools/4.4.0-210-generic/perf stat -o /dev/stdout -e 'syscalls:sys_enter_*'  -p 15572 sleep 10
SYSCALL              COUNT
madvise                  1
mmap                     1
munmap                   1
statfs                   1
getsockopt               5
bind                     8
shutdown                 8
epoll_create1           11
mprotect                17
getsockname             18
newuname                22
sendto                  23
setsockopt              36
connect                 44
fcntl                   75
epoll_ctl               82
recvmsg                171
poll                   212
socket                 243
epoll_wait             326
ioctl                  360
close                  539
write                 6520
newfstat            435492
lseek               442922
read                444750
recvfrom            468426
futex              1853664

```

vmtouch 可以查看某一目录里文件的整体cache情况，很多时候的需求是找出某一个大目录占用page cache 最高的
文件，修改了vmtouch这个小工具，支持打印出某一个目录下的文件占用cache情况，并输出占用cache最高的N个
文件和他们的cache 占用详情:https://github.com/wangxuemin/vmtouch


转载请注明出处，谢谢。。
