---
title: Tokyo Cabinet 的一个bug
date: 2015-07-24 04:17:37
tags:
- Tokyo Cabinet
- mmap
categories: Tokyo Cabinet
---

Tokyo Cabinet 的代码......真是草泥马啊.... 跟LevelDB简直没发比啊...
手机某些机型中Tokyo Cabinet Lib出现了好几次crash报告, 出问题的地方在  2145行:

![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/tc-bug-1.png) 
 <!-- more --> 
通过google breakpad 抓到了每次crash是都同一个非法地址0X0000021, 这个地址太小了肯定非法:
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/tc-bug-2.png) 

这个地址的值恰巧是 HDFLAGSOFF的值， 说明hdb->map == NULL;  看了些调用栈关系和源码发现hdp->map 还真有
可能为空
调用关系如下:

tchdbcloseimp() ---> tchdbsetecode()  ---> tchdbsetflag()

在函数中 3539行，将hdb->map=NULL , 但是后面还调用了三次tchdbsetecode， 有些分支是可以走进到tchdbsetflag
的逻辑的,在tchdbsetflag中对已经为空的hdb->map偏移33的地址赋值，导致crash
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/tc-bug-3.png) 
Tokyo Cabinet  代码可读性真的不怎么样..

转载请注明出处，谢谢。。


