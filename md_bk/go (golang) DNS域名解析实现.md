---
title: go (golang) DNS域名解析实现
date: 2015-05-29 00:17:37
tags:
- golang
- dns
categories:
- golang
---

之前使用过GO语言写了一个实时图片下载程序，主要考虑到GO语言的DNS解析对协程支持友好， 即
DNS解析时不会阻塞执行线程，只会阻塞当前协程，顺便研究了一下GO的net.LookupHost/ResolveIPAddr
实现方式。下面一段描述翻译自go语言的官方文档 https://golang.org/pkg/net/ 域名解析：
域名解析函数，Dial函数会间接调用到，而LokupHost和LookupAddr则会直接调用域名解析函数，不同
的操作系统实现不同,  在Unix系统中有两种方法进行域名解析：

 1）纯GO语言实现的域名解析,从/etc/resolv.conf中取出本地dns server地址列表， 发送DNS请求(UDP
    报文)并获得结果
2)  使用cgo方式， 最终会调用到c标准库的getaddrinfo或getnameinfo函数(不建议使用对GO协程不友好)


关于 cgo dns 解析的坑 参照以下链接
https://jira.mongodb.org/browse/MGO-41
https://github.com/golang/go/issues/8602#issuecomment-66098142

GO语言默认使用纯GO的域名解析，因为这样一个阻塞的DNS请求只会消耗一个协程， 使 用cgo的方式
则会阻塞一个系统线程, 只有某些特定条件下才会使用系统提供的cgo方式, 例如: 1) 在OS X系统中不允许程序
直接发送DNS请求;  2) LOCALDOMAINH环境变量存在，即使为空;  3) ES_OPTIONS或HOSTALIASES或ASR
_CONFIG环境变量非空; 4)/etc/resolv.conf或/etc/nsswitch.conf指定的使用方式GO解析器没有实现;  5)
当要解析的域名以.local结束， 或者是一个mDNS域名

可以通过GODEBUG环境变量来设置go语言的默认DNS解析方式 纯go或cgo,
```
export GODEBUG=netdns=go    # force pure Go resolver 纯go 方式
export GODEBUG=netdns=cgo   # force cgo resolver   cgo 方式
```
也可以在编译时指定netgo或netcgo的编译tag来设置
在plan 9中 域名解析只能通过 /net/cs和 /net/dns
在windows中 域名解析只能通过windows提供的C标准库函数GetAddrInfo或DnsQuery

OK 官方说明看完了， 我们写一个例子试一下
```golang
package main

import (
	"net"
	"fmt"
	"os"
)

func main() {
	ns, err := net.LookupHost("www.baidu.com")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Err: %s", err.Error())
		return
	}

	for _, n := range ns {
		fmt.Fprintf(os.Stdout, "--%s\n", n) 
	}
}
```

使用strace命令分析一下， 系统调用过程:
```c
openat(AT_FDCWD, "/etc/hosts", O_RDONLY|O_CLOEXEC) = 3
read(3, "127.0.0.1   localhost localhost."..., 4096) = 158
read(3, "", 3938)                       = 0
read(3, "", 4096)                       = 0
close(3)                                = 0

//读取本地dns server 配置
stat("/etc/resolv.conf", {st_mode=S_IFREG|0644, st_size=104, ...}) = 0


//创建UDP socket 发送准备发送DNS请求
socket(PF_INET, SOCK_DGRAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_IP) = 3
setsockopt(3, SOL_SOCKET, SO_BROADCAST, [1], 4) = 0
connect(3, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("172.16.1.3")}, 16) = 0
epoll_create1(EPOLL_CLOEXEC)            = 4
// 将UDP socket 加入到epoll中
epoll_ctl(4, EPOLL_CTL_ADD, 3, {EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, {u32=2130514816, u64=140679489328000}}) = 0
getsockname(3, {sa_family=AF_INET, sin_port=htons(57587), sin_addr=inet_addr("10.0.2.15")}, [16]) = 0
getpeername(3, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("172.16.1.3")}, [16]) = 0

//发送DNS请求
write(3, "\363}\1\0\0\1\0\0\0\0\0\0\3www\5baidu\3com\0\0\34\0\1", 31) = 31
futex(0x645c10, FUTEX_WAIT, 0, NULL)    = 0

read(3, 0xc82007c000, 512)              = -1 EAGAIN (Resource temporarily unavailable)
epoll_wait(4, {{EPOLLOUT, {u32=2130514816, u64=140679489328000}}, {EPOLLOUT, {u32=2130514624, u64=140679489327808}}}, 128, 0) = 2
// epoll等待socket 事件
epoll_wait(4, {{EPOLLIN|EPOLLOUT, {u32=2130514624, u64=140679489327808}}}, 128, -1) = 1
futex(0x645680, FUTEX_WAKE, 1)          = 1
read(5, "g\217\201\200\0\1\0\2\0\r\0\v\3www\5baidu\3com\0\0\1\0\1\300"..., 512) = 474
epoll_ctl(4, EPOLL_CTL_DEL, 5, {0, {u32=0, u64=0}}) = 0
close(5)                                = 0
epoll_wait(4, {}, 128, 0)               = 0
epoll_wait(4, {{EPOLLIN|EPOLLOUT, {u32=2130514816, u64=140679489328000}}}, 128, -1) = 1
futex(0x645680, FUTEX_WAKE, 1)          = 1


// 得到DNS解析结果
read(3, "\363}\201\200\0\1\0\1\0\1\0\0\3www\5baidu\3com\0\0\34\0\1\300"..., 512) = 115
epoll_ctl(4, EPOLL_CTL_DEL, 3, {0, {u32=0, u64=0}}) = 0
close(3)                                = 0
```
转载请注明出处，谢谢。。
