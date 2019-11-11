---
title: python基于协程的网络库gevent、eventlet
date: 2015-07-29 00:17:37
tags:
- gevent
- event
categories: python
---

python网络库也有了基于协程的实现，比较著名的是 gevent、eventlet 它两之间的关系可以参照
[Comparing gevent to eventlet](http://blog.gevent.org/2010/02/27/why-gevent/)， 本文主要简单介绍一下eventlet一个例子
客户端：
```python
import eventlet
from eventlet.green import urllib2

def myfetch(myurl, i):
	req = urllib2.Request(myurl)
	req.add_header('User-agent', 'Mozilla 5.10')
	res = urllib2.urlopen(req, timeout = 4)
	body = res.read();
	size = len(body);
	print (i, 'body size ' ,size)
	return size


myurl = "http://127.0.0.1:6000"
pool = eventlet.GreenPool(1000)
for i in range(1, 200):
	pool.spawn(myfetch, myurl, i)
#print i
pool.waitall()
print "--finish --GreenPool"
```
服务端：
```python
#! /usr/bin/env python
"""\
Simple server that listens on port 6000 and echos back every input to
the client.  To try out the server, start it up by running this file.

Connect to it with:
  telnet localhost 6000

You terminate your connection by terminating telnet (typically Ctrl-]
and then 'quit')
"""
from __future__ import print_function

import eventlet


def handle(fd):                     #单个协程的处理逻辑
    print("client connected")
    while True:
        # pass through every non-eof line
        x = fd.readline()
        if not x:
            break
        fd.write(x)
        fd.flush()
        print("echoed", x, end=' ')
    print("client disconnected")

print("server socket listening on port 6000")
server = eventlet.listen(('0.0.0.0', 6000))  #监听6000端口
pool = eventlet.GreenPool()  #构造协程池
while True:
    try:
        new_sock, address = server.accept() #accept新的连接
        print("accepted", address)
        pool.spawn_n(handle, new_sock.makefile('rw'))  #将新的连接交由一个新的协程去处理
    except (SystemExit, KeyboardInterrupt):
        break
```
上面的例子可以看出eventlet接口还是非常的简洁和优雅的，至于稳定性和成熟度还待真实的场景去验证，
使用eventlet快速开发一个tcp/http的server还是非常迅速的，因为是基于协程的, 对于网络IO密集型的场景
速度不会太差.
eventlet已知的在openstack项目中有使用.

转载请注明出处，谢谢。。


