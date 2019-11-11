---
title:  leveldb的seek_compaction
date: 2016-10-16 07:59:34
tags:
- leveldb

categories:
- leveldb
---

Seek Compaction：

如果在key在某个文件的key range的范围内，但是总是找不到，这时需要去高一级的level查找。显然该文件和高一级的文件key的范围重叠很很严重，会导致读效率的下降。因此，需要对该文件发起一次major compaction，减少该level 和level ＋ 1的key的重叠。
某一文件允许seek miss的的最大值为：

```c
f->allowed_seeks = (f->file_size / 16384)    //  文件长度／16K 
```
seek compaction 功能在rocksdb给删除掉了.原因是RocksDB先是改进了一下seek compaction条件
https://github.com/facebook/rocksdb/commit/c1bb32e1ba9da94d9e40af692f60c2c0420685cd
在最初的版本 allowed_seeks ++的条件非常宽松， 只要Get()在两个文件以上进行过查询都会增加allowed_seeks
的值，这是不合理的会增加不必要的compaction，因为有可能Get()操作只是在block cache中查询了一下， 
或者 bloom filter 直接返回没有该值了，并没有真正的文件seek.  RocksDB限制了allowed_seeks ++条件
(只有 从文件中read block操作了才会触发).

后续的版本发现改进后seek compaction触发的情况比较少见，seek compaction增加了代码的复杂度而且有时候
会减慢读取速度，索性rocksdb将该功能给删除了, 以后可能针对某一文件seek miss过多在 compaction策略中
统一处理，

转载请注明出处，谢谢。。
