---
title: LVS RS节点健康检查
date: 2015-07-28 07:17:37
tags:
- linux
- lvs
categories: linux
---

&emsp; &emsp; LVS RS健康节点检查一般交由keepalived来做. 当然也可以自己写一个脚本来检查，通过tcp_connnect
或者curl get 方式定期检测RS节点，如果检测失败则在LVS上删除该RS节点.
&emsp; &emsp; 下面介绍一下百度内部的LVS(又叫做BVS) RS默认健康检测方式.
&emsp; &emsp; 服务上线到BVS后，BVS会维护一份VIP-RS对应关系的配置，通过健康检查的机制来及时发现和屏蔽
故障的RS，使业务上的故障或调整对用户透明.
&emsp; &emsp; BVS以VIP为单位，根据指定的间隔时间对各个RS的服务端口发起一次tcp请求，若RS端口正常回复
SYN+ACK，即认为健康检查成功。反之认为检查失败。健康检查仅对RS发起tcp请求，RS响应 SYN+ACK
之后，BVS随即发送RST包。因此对RS的运行及处理能力几乎没有影响。
若RS健康检查失败次数达到指定的retry次数，BVS系统会自动将该RS从VIP下摘除。在摘除前新进入流量
还会分配到该RS上。当RS机器恢复后健康检查会发现并自动将RS机器再加回到VIP下，BVS将开始为其分配
流量。不需要人工干预
&emsp; &emsp; 健康检查默认配置为： 间隔6S，重试2次，连接超时为5S.  (感觉有点久...最好在秒级) RS随便重启
肯定会丢流量的哦...
&emsp; &emsp; 没有查到BVS是否是通过keepalived来实现健康检查的. 还是自己实现了一套代码,keepalived据说性能很
一般。 

转载请注明出处，谢谢。。


