---
title: go(golang) dns 解析源码 go/src/net/dnsclient_unix.go 分析
date: 2015-09-04 00:18:37
tags:
- golang
- dns
categories:
- golang
---

关于go dns解析的一些说明参照我的另一篇文章 --go (golang) DNS域名解析实现--
go dns 解析 源码在go/src/net/dnsclient_unix.go,  lookupHost()通过向本地dns server发送请求，获得IP和域名的
对应关系然后返回，函数调用关系如下：
```c
//lookupHost()
//->goLookupHostOrder()
//-->goLookupIPOrder()
//--->tryOneName()
//---->exchange()
```
```golang
func exchange(server, name string, qtype uint16, timeout time.Duration) (*dnsMsg, error) {
	d := Dialer{Timeout: timeout}
	out := dnsMsg{
		dnsMsgHdr: dnsMsgHdr{
			recursion_desired: true,
		},
		question: []dnsQuestion{
			{name, qtype, dnsClassINET},
		},
	}
	for _, network := range []string{"udp", "tcp"} {
		c, err := d.dialDNS(network, server)    //创建UDP
		if err != nil {
			return nil, err
		}
		defer c.Close()
		if timeout > 0 {
			c.SetDeadline(time.Now().Add(timeout))
		}
		out.id = uint16(rand.Int()) ^ uint16(time.Now().UnixNano())
		if err := c.writeDNSQuery(&out); err != nil {   //发送DNS请求
			return nil, err
		}
		in, err := c.readDNSResponse()   //解析DNS请求得到IP
		if err != nil {
			return nil, err
		}
		if in.id != out.id {
			return nil, errors.New("DNS message ID mismatch")
		}
		if in.truncated { // see RFC 5966
			continue
		}
		return in, nil
	}
	return nil, errors.New("no answer from DNS server")
}
```

其中的timeout 是 dns 超时时间 是在dnsconfig_unix.go 文件中读取 /etc/reslove.conf  的配置决定的
net.go中的DialTimeout函数也会走到DNS解析流程中，该函数最终会调用到 lookupIPDeadline 启用一个新的协
程去解析DNS， 具体调用栈如下:
```c
//DialTimeout()
//->resolveAddrList()
//-->internetAddrList()
//--->lookupIPDeadline()
//---->lookupGroup.DoChan() 在新的协程中去做 dns解析
//----->lookupIP()
//------>goLookupIPOrder()
```
总之，纯go语言的 DNS解析流程还是比较完善的~~


转载请注明出处，谢谢。。

