---
title: NGINX最新版正式增加了对reuseport (SO_REUSEPORT) 的支持,正式补丁代码分析
date: 2015-07-25 02:17:37
tags:
- nginx
categories: nginx
---

NGINX release 1.9.1 introduces a new feature that enables use of the SO_REUSEPORT 

https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/
http://forum.nginx.org/read.php?29,252762,253401

实现思路补丁里已经说明，跟很早之的一个简单方案的差别比较大
 <!-- more --> 
Here, I am proposing a simpler way to enable the SO_REUSEPORT support. It is just to create
and configure certain number of listen sockets in the master process with SO_REUSEPORT enabled.
All the children processes can inherit. In this case, we do not need to worry about ensuring 1 available 
listen socket at the run time. The number of the listen sockets to be created is calculated based on the 
number of active CPU threads. With big system that has more CPU threads (where we have the scalability
issue), there are more duplicated listen sockets created to improve the throughput and scalability. With system
that has only 8 or less CPU threads, there will be only 1 listen socket. This makes sure duplicated listen sockets
only being created when necessary. 

在这里我们使用一个简单的方法来支持SO_REUSEPORT, 当enable SO_REUSEPORT时在主进程中去配置
和创建一定数量的listen socket， 然后让子进程去继承。 在这种情况下我们不用担心至少有一个listen socket存活
(配置升级).  创建监听socket  的数量是通过CPU的数量决定的.  CPU数目多则多创建socket. CPU数目少于8核
则只会创建一个listen socket.
关于创建listen socket 的数目从最终的diff看 listen socket数量跟工作进程的数目一致

最终修改代码diff：
http://trac.nginx.org/nginx/changeset/4f6efabcb09b693ac5461f2b1d05a526f9710137/nginx
```cpp
//****************************************************************
//复制listen socket的配置， 后续的创建流程会根据listen socket 配置进行创建
//假设原来监听listen socket个数为 1 , 工作线程数为10, 则复制后需要创建的listen 
//socket的配置变为10*1个， 后续主进程会根据配置个数也就是10个来创建listen socket
//****************************************************************

ngx_int_t ngx_clone_listening(ngx_conf_t *cf, ngx_listening_t *ls)
{
#if (NGX_HAVE_REUSEPORT)

    ngx_int_t         n;
    ngx_core_conf_t  *ccf;
    ngx_listening_t   ols;

    if (!ls->reuseport) {
        return NGX_OK;
    }

    ols = *ls;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cf->cycle->conf_ctx,
                                           ngx_core_module);

    for (n = 1; n < ccf->worker_processes; n++) {

        // create a socket for each worker process /
                                                    //
        ls = ngx_array_push(&cf->cycle->listening); //根据工作进程的数目复制listen socket配置 
        if (ls == NULL) {
            return NGX_ERROR;
        }

        *ls = ols;
        ls->worker = n;
    }

#endif

    return NGX_OK;
}


//****************************************************************
//子进程继承父(主)进程的listens socket 
//****************************************************************
static ngx_int_t ngx_event_process_init(ngx_cycle_t *cycle)
{
    ngx_listening_t     *ls;

    ls = cycle->listening.elts;
    
    for (i = 0; i < cycle->listening.nelts; i++) {

#if (NGX_HAVE_REUSEPORT)
        if (ls[i].reuseport && ls[i].worker != ngx_worker) {
            continue;  
        }    //继承父进程的listen socket 
#endif

        c = ngx_get_connection(ls[i].fd, cycle->log);
     
        ............
        
        ngx_add_event(rev, NGX_READ_EVENT, 0);  //将继承过来的listensocket加入到epoll中
     
     }
}

``` 
