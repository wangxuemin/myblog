---
title: 关于SSDB的网络模型
date: 2015-07-23 19:16:49
tags: 
- leveldb 
- ssdb

categories: leveldb
description: 
---

SSDB 是 完整的LevelDB实现， 因为google的LevelDB并没有实现网络功能， 只封装了数据库的基本操作.

SSDB地址和介绍 详见： https://github.com/ideawu/ssdb.

SSDB作者代码更新的还是比较快的.. 最新的版本比以前的改动好大.
从网络模型上说 SSDB跟 Memcache比较像，但是还有一些 细微的差别，如下图所示:
![](http://raw.githubusercontent.com/wangxuemin/myblog/master/pic_bak/ssdb-1.png) 

Network 网络现成通过epoll 进行网络数据的收发，每当收到一个完整的命令时，生成一个JOB放入workpool的处理
队列，workpool 的线程池进行数据库的get、set操作等。 处理完毕后将处理结果通过另外一个对列发发送给
Network线程，由Network线程将结果数据发回给客户端
``` cpp
void NetworkServer::serve(){
	writer = new ProcWorkerPool("writer");    //生成leveldb写操作的线程池
	writer->start(num_writers);
	reader = new ProcWorkerPool("reader");    //生成leveldb读操作的线程池
	reader->start(num_readers);

	ready_list_t ready_list;
	ready_list_t ready_list_2;
	ready_list_t::iterator it;
	const Fdevents::events_t *events;

	fdes->set(serv_link->fd(), FDEVENT_IN, 0, serv_link);
	// workpool每处理完毕后，向该管道写一个字符
	fdes->set(this->reader->fd(), FDEVENT_IN, 0, this->reader); 
	
	//通知network线程有数据到来
	fdes->set(this->writer->fd(), FDEVENT_IN, 0, this->writer);
	
	uint32_t last_ticks = g_ticks;
	
	while(!quit){
		// status report
		if((uint32_t)(g_ticks - last_ticks) >= STATUS_REPORT_TICKS){
			last_ticks = g_ticks;
			log_info("server running, links: %d", this->link_count);
		}
		
		ready_list.swap(ready_list_2);
		ready_list_2.clear();
		
		if(!ready_list.empty()){
			// ready_list not empty, so we should return immediately
			events = fdes->wait(0);  //epoll_wait 等待事件的发生
		}else{
			events = fdes->wait(50);
		}
		if(events == NULL){
			log_fatal("events.wait error: %s", strerror(errno));
			break;
		}
		
		for(int i=0; i<(int)events->size(); i++){
			const Fdevent *fde = events->at(i);
			//listen socket 有时间发生表示有新的client connect
			// 调用accept 接收新的连接
			if(fde->data.ptr == serv_link){
				Link *link = accept_link();   

				if(link){
					this->link_count ++;				
					log_debug("new link from %s:%d, fd: %d, links: %d",
						link->remote_ip, link->remote_port, link->fd(), this->link_count);
					fdes->set(link->fd(), FDEVENT_IN, 1, link);
				}
			}else if(fde->data.ptr == this->reader || fde->data.ptr == this->writer){
				// 表示workpool 有新的job处理完成
				ProcWorkerPool *worker = (ProcWorkerPool *)fde->data.ptr;
				ProcJob job;
				//取出处理完成的JOB
				if(worker->pop(&job) == 0){
					log_fatal("reading result from workers error!");
					exit(0);
				}
				// 主要的处理逻辑将job的处理结果发送给客户端
				if(proc_result(&job, &ready_list) == PROC_ERROR)

				}
			}else{
				//此分支为普通的用户读写事件
				proc_client_event(fde, &ready_list);
			}
		}

		for(it = ready_list.begin(); it != ready_list.end(); it ++){
			Link *link = *it;
			if(link->error()){
				this->link_count --;
				fdes->del(link->fd());
				delete link;
				continue;
			}

			const Request *req = link->recv();
			if(req == NULL){
				log_warn("fd: %d, link parse error, delete link", link->fd());
				this->link_count --;
				fdes->del(link->fd());
				delete link;
				continue;
			}
			if(req->empty()){
				fdes->set(link->fd(), FDEVENT_IN, 1, link);
				continue;
			}
			
			link->active_time = millitime();
			//走到此处表明，已经有一个完成的命令读取完毕
			ProcJob job;
			job.link = link;
			this->proc(&job);
			//生成一个新的JOB，并抛给后端工作线程处理
			if(job.result == PROC_THREAD){
				fdes->del(link->fd());
				continue;
			}
			if(job.result == PROC_BACKEND){
				fdes->del(link->fd());
				this->link_count --;
				continue;
			}
			
			//发送结果处理函数
			if(proc_result(&job, &ready_list_2) == PROC_ERROR){
				//
			}
		} // end foreach ready link
	}
}
```
如果仔细看代码的话发现，上面的两个队列是不同类型的， 一个为Queue<JOB>, 一个为SelectableQueue<JOB>
Queue<JOB> 比较普通的一个多线程队列的实现 通过pthread_mutex_lock 和 pthread_cond_signal进行线程
间的同步SelectableQueue的同步方式比较特别 使用了pthread_mutex和pipe进行多线程间的同步， 很奇怪是吧

为什么要引入SelectableQueue呢，我们不妨思考下，当工作线程把JOB处理完了以后怎么样告诉NetWork线程呢因
为Network线程一直在异步epoll收发用户的请求和结果，该线程不能被阻塞，所以一个比较巧妙的方式是NetWork线
程和工作线程创建一个管道，Network线程将该管道加入到epoll监听fd中，  每当工作线程有结果是工作线程就会往
该管道写一个"1"字符触发Network epoll读事件。这样Network线程就可以将网络事件和job完成事件进行统一在
epoll中处理

``` cpp
template <class T>
int SelectableQueue<T>::push(const T item){
	if(pthread_mutex_lock(&mutex) != 0){
		return -1;
	}
	{
		items.push(item);
	}
	if(::write(fds[1], "1", 1) == -1){
	    // 工作线程调用， 向管道中写一个字符， 通知网络线程有结果完成
		fprintf(stderr, "write fds error\n");
		exit(0);
	}
	pthread_mutex_unlock(&mutex);
	return 1;
}


template <class T>
int SelectableQueue<T>::pop(T *data){
	int n, ret = 1;
	char buf[1];

	while(1){
	    // 网络线程调用， 读取管道中的字符
		n = ::read(fds[0], buf, 1);
		if(n < 0){
			if(errno == EINTR){
				continue;
			}else{
				return -1;
			}
		}else if(n == 0){
			ret = -1;
		}else{
			if(pthread_mutex_lock(&mutex) != 0){
				return -1;
			}
			{
				if(items.empty()){
					fprintf(stderr, "%s %d error!\n", __FILE__, __LINE__);
					pthread_mutex_unlock(&mutex);
					return -1;
				}
				//取出结果
				*data = items.front();
				items.pop();
			}
			pthread_mutex_unlock(&mutex);
		}
		break;
	}
	return ret;
}
``` 
该方式很常见 包括 libevent libev 等都使用了类似技术.

转载请注明出处,谢谢~~