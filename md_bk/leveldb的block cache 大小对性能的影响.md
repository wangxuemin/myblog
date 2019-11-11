---
title:  leveldb的block cache 大小对性能的影响.md
date: 2016-06-09 07:59:34
tags:
- leveldb

categories:
- leveldb
---

leveldb使用的缓存：
```c
//   indexes and  bloom filters(table索引信息缓存)
//   block cache (解压后的数据)
//   page cache （压缩的数据）
```
在非DirectIO下，block cache作为文件数据的cache层，下层还有os 的page cache 缓存了
硬盘中的文件, 所以block cache大小对系统性能的影响不大. block cache 因为是解压
后的数据，所以可以降低一下cpu的解压时间. 过大的 block cache 以为会导致page cache
数据减少,因为block cache 和 page cache 缓存的部分数据重复导致整体数据缓存量减少
hit rate 还有可能下降，通常来说leveldb 依赖 os的 page cache来减少文件I/O。

rocksdb统计信息比较丰富,通过以上参数信息可以知道block cache 和 page cache的命中情况:
```c
//   block_cache_hit_count 
//   block_read_count
//   block_read_byte 
```
在DirectIO下，文件缓存全部依赖 block cache, 所以block cache越大越好
通常设置为系统内存的60%-80%


转载请注明出处，谢谢。。
