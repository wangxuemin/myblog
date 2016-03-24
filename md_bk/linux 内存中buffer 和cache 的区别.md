---
title: linux 内存中buffer 和cache 的区别
date: 2015-08-04 04:17:37
tags:
- linux
- buffer
- cache
categories: linux
---

page cahce 缓存了页面用来优化文件I/O, buffer cache 缓存了磁盘块用来优化 block I/O.在linux kernel
2.4之前，这两个缓存是不同的： 文件在page cache里， 磁盘块在buffer cache里. 这样某些系统调用(mmap
)数据需要在两层cache中都保存了一份. 许多unix系统都遵循类似的模式. 这样很容易实现， 但是看起来很不
优雅和低效, 从linux 2.4开始两种cache开始统一起来. VM子系统接管了I/O读写和页面缓存, 如果缓存的数据
是一个文件内容（大多数据都是这样）这时buffer cache指向了page cache，也就是说数据只存在一份，buffer 
cache向硬盘发起读请求时数据会直接写入到 page cache中,因为指向了同一地址.  page cache 就可以简单看
做磁盘的一层缓存.

buffer cache 也会独立存在， 因为内核有时候需要普通小的block I/O 而不是以页为单位读写，硬盘大
多数是文件内容的读写(以页为单位)， 此时buffer cache复用了page cache的内存, 但是有些少量的数据并不
是文件内容例如：文件元数据(inode,dentry等)和原始块 I/O, 这种情况下的磁盘缓存则通过buffer cache 实现.
关于上面的一段历史可以参照 : 

https://www.usenix.org/legacy/event/usenix2000/freenix/full_papers/silvers/silvers_html/

简单说来page cache缓存了文件内容而buffer cache 缓存了一些文件系统需要用的元数据包括磁盘中的inode
目录等信息. Linux 内核情景分析中，有些章节有说明这个问题，如下：

![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/buf-cache-1.png) 

参考资料:http://www.quora.com/Linux-Kernel/What-is-the-major-difference-between-the-buffer-cache-and-the-page-cache


转载请注明出处，谢谢。。