---
title: Tokyo Cabinet Hash数据库存储和索引结构
date: 2015-07-31 22:59:34
tags:
- Tokyo Cabinet
- 数据库
categories:
- Tokyo Cabinet
---


先看图:
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/tc-1.png) 
 <!-- more --> 

```
head:              数据库头文件.
bucket Array:      hash索引数组，存放对key进行hash之后得到的hash值所对应的第一个
                   元素在数据库文件中的偏移量.  比如当第一个数据来的时候请求存储，这时计算到该key的
                   hash index值是1，在bucket array数组中找到1对应的位置，发现是NULL，就将这个值
                   存在目前数据存储区的第一个可用位置。并将这个位置记录到bucket array里.

freepool array:    数据文件中空闲的区域，在新插入数据时先在里面查找是否有合适的位置，当删除一个record
                   时，会把该record的偏移和大小记录到里面.

record:            存放record的数据区.

MSIZ:              最小mmap大小，即： head+bucket array的大小， hash数组的索引区必须使用mmap映射到
                   内存

XMSIZ:             内存比较充足的情况下， 除了mmap索引区域外还可以mmap一部分rcord区域的内容，加快读写
                   速度
```

hash数据库的原理是通过key值用一个hash算法算出一个bidx值，然后在这个表(bucket array)里查这个bidx对应的
key-value值在文件中的偏移，再在文件中查找record记录.

当然hash算法是会冲突的，当不同的key值算到了同样的hash，那我们仅用上面的一个bucket array是不能区分的, 
Tokyo Cabinet采用一个二叉树来管理冲突的key,  hash相等的所有记录都是互相用数据单元的left，right指针
（其实就是一个offset值）连接起来的 

如下图 :

![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/tc-2.png) 

转载请注明出处，谢谢。。

