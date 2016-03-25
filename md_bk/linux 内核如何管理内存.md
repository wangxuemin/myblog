---
title: linux 内核如何管理内存
date: 2015-07-30 01:17:37
tags:
- linux
- 内存
categories: linux
---

翻译自 http://duartes.org/gustavo/blog/post/how-the-kernel-manages-your-memory/ 感觉作者的精美图片
建议读者对一遍英文原文
&emsp; &emsp; 在介绍完了进程虚拟地址空间的布局后， 我们来看一下内核是如何管理内存的：

![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/linux-kenel-mange-mem-1.png) 
 <!-- more --> 

&emsp; &emsp; linux的进程在内核中是由一个task_struct结构体描述的, 其中task_struct里面有一个mm_struct结构体，该结构
体是进程内存管理的主要结构体.  其中mm_struct 存放了每一个虚拟地址段的起始地址；进程使用的真正的物理
内存的页数；进程使用虚拟空间的数量;其他额外的信息. 进程内存管理主要包括两部分：虚拟地址块的集合和页
表, 下图显示了进程虚拟地址的管理.

![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/linux-kenel-mange-mem-2.png) 

&emsp; &emsp; 每一个虚拟地址空间(virtual memory area 简称VMA)是一段连续的虚拟地址区间, 这些区间不会重叠. 每一个
VMA由一个vm_area_struck来描述,包括VMA的起始和结束地址, 访问权限等.  其中的vm_file字段表示了该区域
映射的文件(如果有的话). 有些不映射文件的VMA是匿名的，例如上图中的heap/stack都分别对应于一个单独的
匿名的VMA. 进程的VMA存放在一个list和一个红黑树中,  该list根据VMA的起始地址排序. 存放在红黑树中是为
了加快查找速度可以很快的查找某一地址是否在进程的某一个VMA中. 通过命令读取/proc/pid_of_process/maps
文件查看进程的内存映射时,  其实内核只是简单便利了存放VMA的list然后打印出来.

&emsp; &emsp; 在windows中, EPROCESS块大致是task_struct和mm_struck的结合.  存放一个VMA的结构叫做VAD( Virtual 
Address Descriptor)，VAD存放在一个AVL树中. 
&emsp; &emsp; 进程4G的虚拟地址空间被划分为页. x86架构32位模式支持页的大小为(4K, 2M, 4M ).  linux和windows默认使
用4k大小的页. 一个VMA的大小必须是页大小的整数倍. 下图展示了进程虚拟地址空间中用户的3G空间页的表示
方式:
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/linux-kenel-mange-mem-3.png) 

&emsp; &emsp; CPU使用页表将虚拟地址转化为真正的物理地址,  每一个进程都有自己的页表, 当进程切换时当前进程对应
的页表也会跟着切换.linux进程内存管理里有一个pgd的字段指向该进程的页表. 虚拟地址的每一页都在页表中
存在一个记录称作页表项（PTE）用来指向真是的物理页, 在X86系统里每一个PTE大小为4个BYTE如下图：
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/linux-kenel-mange-mem-4.png) 
&emsp; &emsp; PTE中有很多flag,linux有专门的函数用来设置或读取这些flag. 其中P标志位表示该虚拟内存页是否已经映射
物理内存如果P标志为0时访问该页会触发一个page fault. R/W标志则表示了该页的读写属性. U/S则表示该页的
访问权限，只能由内核空间访问，还是进程的用户空间也可以访问.  这些标志位实现了内存的读写保护和内核
空间内存的访问保护.D/A标志表示了一个页面是否为脏页面和是否被访问过. 如果一个页面被写过那么该页面D
标志为1,   如果一个页面被读写过那么A标志为1. CPU只会设置这些标志位, 这些标志位只能由内核去清理. 最
后. PTE存放了它所指向的物理页面的真实地址,该地址4k对齐. 通常页表最大映射的内存为4G， 但是可以通过
PTE的一些标志位进行扩展.

&emsp; &emsp; 一个虚拟的内存也是内存保护的基本单位，因为页里面的内存共享U/S. R/W标志.  但是同一个物理内存页可
能被映射到不同的虚拟内存页, 这些虚拟内存也有可能有不同的保护标志.  在PTE中没有内存是否可执行的标
志，所以在X86系统里允许栈上的代码被执行, 这也是很容易被缓冲区溢出攻击的原因. PTE没有可执行标志导
致了虽然VMA有很多权限控制标志, 但是都不能够传递给硬件(CPU). 内核尽可能的做些保护但是因为CPU架
构的一些关系, 不会发生太大作用.

&emsp; &emsp; 虚拟内存不会存放任何东西,  只是指向了真实物理内存的地址,该物理地址是CPU真真可以访问的,  又称为
物理地址空间.在总线上的内存操作比较复杂, 我们可以假设物理地址区间是从0到可用内存按照字节自增的.物
理地址空间被内核以页大小为单位划分页帧. CPU不知道页帧的存在, 但是页帧是对内核却很重要, 也是内核内
存管理最基本的单元. Linux和Windows都使用4KB大小的页帧(32位模式); 下图是一个拥有2G内存的例子：

![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/linux-kenel-mange-mem-5.png) 
&emsp; &emsp; 在Linux中通过一个描述符和一些标志位来表示一个物理页， 所有的描述符和标志位加在一起标识了全部的
物理内存, 一个页帧的状态总是确定并可知的. 物理内存管理系统通过伙伴算法来分配和回收物理页面, 通过伙
伴系统分配的物理页面一开始的状态是free的, 该页面有可能用作存放程序数据(匿名的), 也可能被用作page
 cache存放文件或磁盘的数据. 在Windows中有一个PFN(Page Frame Number)数据库用来记录跟踪物理内存
的状态.接下来我们看一下虚拟内存区域， 页表项， 物理页帧是怎样一起工作的, 下图为一个用户的堆栈:

![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/linux-kenel-mange-mem-6.png) 

&emsp; &emsp; 蓝色矩形表示虚拟内存区域，箭头表示虚拟内存通过页表映射的物理内存, 图中没有箭头的虚拟内存表示还
没有映射真实的物理内存, 也就是说这些虚拟内存对应的页表项(PTE)的P标志为空. P标志为空表示这些虚拟内
存从未被进程访问过或者这些页面可能被交换出(swap out). 不管哪中情况当P标志为空时,  访问这想和页面会
触发page fault, 虽然访问这些地址在虚拟页内,  但是因为没有相应的物理页所以会触发页中断.
&emsp; &emsp; VMA是进程和内核之间的桥梁,当进程需要某些资源(malloc, mapp a file 等),  内核很快完成，但是此时内核
只修改了VMA部分,    并没有真正的去分配这些资源,  仅当进程真正使用这些资源时，会触发page fault来真正
完成资源的分配. 内核的这种机制称作延迟分配, 是虚拟内存管理的基本准则. VMA记录了已经同意分配的资源,
PTE真实反映了内核确切已经分配的资源, 这两个结构体共同完成了进程的内存管理； 在page fault处理,释放
内存, 内存的换出中都扮演了重要的角色. 让我们来看一个内存分配的例子:
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/linux-kenel-mange-mem-7.png) 
&emsp; &emsp; 当进程需要更多内存是会调用brk()系统调用, 内核只是在VMA区中标识了一块内存，真是的物理内存页此
时并没有分配. 一旦用户进程试图访问VMA区中的这些内存时,会触发 page fault, 处理page fault的是
do_page_fault()函数. do_page_fault()函数会首先通过find_vma()来确定触发page fault的内存是否在进程的
VMA区间中, 如果在, 则进一步查看该VMA区间的读写和访问权限是否匹配。 如果没有找到合适的VMA或者
此次读写访问不符合VMA的权限，则会给进程发生Segmentation Fault消息.

&emsp; &emsp; 当找到所属的VMA时, 内核会根据VMA对应的PTE内容做相应的处理, 本例中PTE显示了物理页面不存在.
实际上我们的PTE全部为0, 这表示我们的虚拟内存页从来没有被映射过, 因为这是一个匿名的VMA.内核会调用 
do_anosnymous_page()分配一个物理内存页， 并通过PTE映射到虚拟内存页上.

&emsp; &emsp; 还有其他情况, 例如：PTE表示一个已经换出的页面, P标志为0, 但是PTE非空，此时PTE指向了存放该页面的
磁盘地址信息, 这时会通过 do_swap_page从磁盘读出页面的内容copy到一个新分配的页面并更新PTE信息.


转载请注明出处，谢谢。。


