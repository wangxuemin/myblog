---
title: LEVELDB(SSDB)关于读操作两种CACHE的作用和配置
date: 2015-07-24 01:05:03
tags: 
- leveldb
categories: leveldb
---
SSDB及LEVELDB的用来优化查找Cache分为两种，分别是table_cache和block_cache。
``` 
  table_cache用来缓存的是sstable的索引数据，也可以理解为mysql中得二级索引在内存中得缓存， 及通常所
  说的元数据的缓存，bloom_fileter就放在table_cache,来快速定位一个key是否在该table中.

  block_cache用来缓存的block数据，即文件内容的缓存;
``` 
table_cache 在ssdb中通过leveldb.max_open_files 来设置，最大1000个table及1000个sstable文件;
block_cache 通过leveldb.block_size 设置；
关于两种cache在查找流程中得位置如下图所示:
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/leveldb-cache-1.png) 

转载请注明出处 谢谢~~
