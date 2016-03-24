---
title: lvs 负载均衡fullnat 模式clientip 怎样传递给 realserver
date: 2015-07-26 03:17:37
tags:
- linux
- lvs
categories:
- linux
---

&emsp; 关于LVS和FULLNAT的介绍可以看一下 淘宝吴佳明(普空)的视频  http://blog.aliyun.com/1750 ，FULLNAT
模式很大简化了LVS的配置和部署，目前淘宝和百度基本上都在使用FULLNAT模式来作为接入侧的负载均衡模式.
百度的LVS叫做BVS, Baidu Virtual Server, 是在LVS基础上修改的增加了L3 Though 和 SYN Porxy，貌似也
是吴佳明(普空)在百度搞的, 类似FULLNAT 项目.
&emsp; 下面的图来自吴佳明(普空)的PPT, 自己重画了一遍，关于NAT和FULLNAT的区别如下图所示：
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/net.png) 

&emsp; 看完上图后发现 FULLNAT有一个问题是：RealServer无法获得用户IP；淘宝通过叫TOA的方式解决的，
主要原理是：将client address放到了TCP Option里面带给后端RealServer，RealServer收到后保存在socket
的结构体里并通过toa内核模块hook了getname函数，这样当用户调用getname获取远端地址时，返回的是保
存在socket的TCPOption的IP. 百度的BVS是通过叫ttm模块实现的，其实现方式跟toa基本一样，只是没有开源.
实现原理图如下：
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/lvs-2.png) 


&emsp; 下面看下上面说的逻辑的实现代码 https://github.com/alibaba/LVS

lvs侧在TCP报文的选项中插入clientip代码:  tcp_fnat_in_handler():
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/lvs-3.png) 
RS侧收到建连报文时，取出toa里面的client ip和port 存放在socket的use_data里,toa.c:
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/lvs-4.png) 
HOOK挂载：
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/lvs-5.png) 


&emsp; 当应用层调用getpeername() 或者 getsocketname() 时，会进入到inet_getname_toa,如果存在toa信息则将socket
里存放的真是的clientip 返回给应用层。

转载请注明出处，谢谢。。


