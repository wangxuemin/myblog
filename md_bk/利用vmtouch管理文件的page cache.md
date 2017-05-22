---
title:  利用vmtouch管理文件的page cache
date: 2016-02-15 07:59:34
tags:
- liunx
- 文件
- page cache

categories:
- linux
---



利用vmtouch管理文件的page cache,   vmtouch主页和使用说明：https://hoytech.com/vmtouch/
源码也比较简单 https://github.com/hoytech/vmtouch/blob/master/vmtouch.c

```c

void usage() {   //使用说明
  printf("\n");
  ....
  printf("  -t touch pages into memory\n");
  printf("  -e evict pages from memory\n");
  ....
  printf("  -v verbose\n");
  exit(1);
}

```
vmtouch的功能主要分三部分:
```c
  1. 查看一个文件被缓存了多少
  2. 把文件从磁盘缓存到内存中
  3. 把内存中的缓存数据驱逐到硬盘，释放page cache
```
具体实现：
--1--查看一个文件被缓存了多少即:文件的哪些部分在page cache中，是通过mincore系统调用实现的:

```c
    if (mincore(mem, len_of_range, (void*)mincore_array)) fatal("mincore %s (%s)", path, strerror(errno));
    for (i=0; i<pages_in_range; i++) {
      if (is_mincore_page_resident(mincore_array[i])) {
        total_pages_in_core++;
      }
    }
```
mincore在内核中的实现代码也比较简单: https://github.com/torvalds/linux/blob/v2.6.12-rc3/mm/mincore.c

--2--把文件从磁盘缓存到内存中:
```c
    if (o_touch) {
      for (i=0; i<pages_in_range; i++) {
        junk_counter += ((char*)mem)[i*pagesize];   //mmap映射文件后以page size为单位，读取每个page的第一个字节，
                                                    //从而触发内核把该page从硬盘读取到内存中 
                                                    // 
        mincore_array[i] = 1;

        ....
    }
```
--3--把内存中的缓存数据驱逐到硬盘，释放page cache：

```c
  if (o_evict) {
    if (o_verbose) printf("Evicting %s\n", path);

//#if defined(__linux__) || defined(__hpux)
    if (posix_fadvise(fd, offset, len_of_range, POSIX_FADV_DONTNEED))
      warning("unable to posix_fadvise file %s (%s)", path, strerror(errno));
//#elif defined(__FreeBSD__) || defined(__sun__) || defined(__APPLE__)
//    if (msync(mem, len_of_range, MS_INVALIDATE))
//      warning("unable to msync invalidate file %s (%s)", path, strerror(errno));
//#else
//    fatal("cache eviction not (yet?) supported on this platform");
//#endif
  }
```

在linux中是通过fadvise系统调用实现的最终调用invalidate_mapping_pages释放page. 
fadvise在内核中的实现同样比较简单： https://github.com/torvalds/linux/blob/v2.6.12-rc3/mm/fadvise.c




转载请注明出处，谢谢。。
