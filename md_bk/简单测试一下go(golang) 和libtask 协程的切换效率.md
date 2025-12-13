---
title: 简单测试一下go(golang) 和libtask 协程的切换效率
date: 2015-06-01 22:59:34
tags:
- go
- libtask
- 协程
categories:
- golang
---
简单测试一下go(golang)和libtask协程的切换效率, libtask一个C语言的协程库，是go语言的前身很早期的原型，
测试机器是我的mac air 安装的centos虚拟机(只有一个核)
代码没有采用任何优化，只是使用默认配置

测试结论：
golang 切换100w次 需要 295ms
libtask 切换100w次 需要1446ms



```golang

package main
import (
   "fmt"
   "runtime"
   "time"
   )


var i int = 0

func test(){

	for{
		i++
		if i >1000000{
		  fmt.Printf("end switch \n")
		  break
		}else{
		  //fmt.Printf(" in test i:%d  \n", i)
		}
		runtime.Gosched()

	}

}

func main(){

	fmt.Printf("GOMAXPROCS :%d \n",runtime.GOMAXPROCS(1))
	tv1 := time.Now().UnixNano()

	go test()

	// in main goroutine
	test()

	tv2 := time.Now().UnixNano()

	usetime := (tv2 - tv1)/1000000
	fmt.Printf("use time:  %dms \n", usetime)
	fmt.Printf("end main \n")


}
```
lib task测试代码:
```c
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
#include <stdlib.h>
#include <task.h>
#include<sys/time.h>

extern int tasknswitch;

enum { STACK = 32768 };
int i = 0;
struct timeval tv1, tv2;

void
testswitch(void *v)
{
  int ms;

  while(1)
  {
	  i++;
	  if( i< 1000000)
	  {

	    //printf("int task i:%d \n", i);
	    taskyield();
	  }
	  else
	  { 
	    //  chansendul(c, 0);
	    //taskexitall(0);
	    gettimeofday(&tv2,NULL);
	    ms = ((tv2.tv_sec - tv1.tv_sec)*1000000 + (tv2.tv_usec - tv1.tv_usec))/1000;
	    printf("time use %d\n ", ms);
	    printf("task swithch %d \n", tasknswitch);

	    taskexitall(0); 
	   }
   }
}



void
taskmain(int argc, char **argv)
{
	int  n;
	//c = chancreate(sizeof(unsigned long), 0);
	gettimeofday(&tv1, NULL);
	//for(i=1; i<; i++){
	taskcreate(testswitch, 0, STACK);
	//}
	testswitch(NULL);

}
```


转载请注明出处，谢谢。。
