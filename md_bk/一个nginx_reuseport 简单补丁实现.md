---
title: 一个nginx_reuseport 简单补丁实现
date: 2015-07-25 01:17:37
tags:
- nginx
categories: nginx
---

补丁的diff文件在这里:
http://leaf.dragonflybsd.org/~sephe/ngx_soreuseport.diff

该补丁只是一个简单实现演示，很多东西没有考虑到，作者也只是简单验证了使用REUSEPOORT后的效果(正式补丁
后来正式提交了 见我的博客)

The basic idea of the above patch is:
- Defer the listen socket creation until work processes are forked
- Work process creates listen socket, and set SO_REUSEPORT before bind(2)
- Accept mutex is no longer needed, since worker process is not
contended on the single listen socket anymore

基本实现思想：

----等工作进程创建后才开始监听listen socket
----工作进程创建listen socket,  在bind调用之前设置SO_REUSEPORT选项
----Accpet_mux互斥锁已经不需要了，因为所有工作进程不在使用同一个listensocket

下面主要对该diff文件加了一下注释:
```cpp
# HG changeset patch
# User Sepherosa Ziehau <sepherosa@gmail.com>
# Date 1374824628 -28800
#      Fri Jul 26 15:43:48 2013 +0800
# Node ID 55ad072b8934d3eea6d84c3c694c5f8bd7b37a70
# Parent  6d73e0dc4f647afd13a9daafc7cc7b061b2689dc
#Initial SO_REUSEPORT support


//**配置相关，不详细解释**
static ngx_conf_enum_t  ngx_debug_points[] = {
       0,
       NULL },
 
+    { ngx_string("so_reuseport"),
+      NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_TAKE1,
+      ngx_set_so_reuseport,
+      0,
+      0,
+      NULL },
+
     { ngx_string("debug_points"),
       NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_TAKE1,
       ngx_conf_set_enum_slot,
 
     return NGX_CONF_OK;
 }
+
+
+static char *
+ngx_set_so_reuseport(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
+{
+    ngx_str_t        *value;
+    ngx_core_conf_t  *ccf;
+
+    ccf = (ngx_core_conf_t *) conf;
+
+    value = (ngx_str_t *) cf->args->elts;
+
+    if (ngx_strcmp(value[1].data, "on") == 0) {
+        ccf->so_reuseport = 1;
+    } else if (ngx_strcmp(value[1].data, "off") == 0) {
+        ccf->so_reuseport = 0;
+    } else {
+        return "invalid value";
+    }
+    return NGX_CONF_OK;
+}


diff -r 6d73e0dc4f64 -r 55ad072b8934 src/core/ngx_connection.c
--- a/src/core/ngx_connection.c Thu Jul 25 12:46:03 2013 +0400
+++ b/src/core/ngx_connection.c Fri Jul 26 15:43:48 2013 +0800
 ngx_int_t
 ngx_open_listening_sockets(ngx_cycle_t *cycle)
 {
-    int               reuseaddr;
+    int               reuseaddr, reuseport;
     ngx_uint_t        i, tries, failed;
     ngx_err_t         err;
     ngx_log_t        *log;
     ngx_socket_t      s;
     ngx_listening_t  *ls;
+    ngx_core_conf_t  *ccf;
 
     reuseaddr = 1;
+    reuseport = 0;
 #if (NGX_SUPPRESS_WARN)
     failed = 0;
 #endif
 
+    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
+    if (ccf->so_reuseport) {
+        reuseaddr = 0;
+        reuseport = 1;
+    }
+
     log = cycle->log;
 
                 return NGX_ERROR;
             }
 
-            if (setsockopt(s, SOL_SOCKET, SO_REUSEADDR,
-                           (const void *) &reuseaddr, sizeof(int))
-                == -1)
-            {
-                ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
-                              "setsockopt(SO_REUSEADDR) %V failed",
-                              &ls[i].addr_text);
+            if (reuseaddr) {
+                if (setsockopt(s, SOL_SOCKET, SO_REUSEADDR,
+                               (const void *) &reuseaddr, sizeof(int))
+                    == -1)
+                {
+                    ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
+                                  "setsockopt(SO_REUSEADDR) %V failed",
+                                  &ls[i].addr_text);
 
-                if (ngx_close_socket(s) == -1) {
+                    if (ngx_close_socket(s) == -1) {
+                        ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
+                                      ngx_close_socket_n " %V failed",
+                                      &ls[i].addr_text);
+                    }
+
+                    return NGX_ERROR;
+                }
+            }
             //***********************************************************
+            if (reuseport) {  //在listen socket增加了设置SO_REUSEPORT的选项
             //***********************************************************
+                if (setsockopt(s, SOL_SOCKET, SO_REUSEPORT,
+                               (const void *) &reuseport, sizeof(int)) 
+                    == -1)
+                {
                     ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
-                                  ngx_close_socket_n " %V failed",
+                                  "setsockopt(SO_REUSEPORT) %V failed",
                                   &ls[i].addr_text);
+
+                    if (ngx_close_socket(s) == -1) {
+                        ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
+                                      ngx_close_socket_n " %V failed",
+                                      &ls[i].addr_text);
+                    }
+
+                    return NGX_ERROR;
                 }
-
-                return NGX_ERROR;
             }
 

diff -r 6d73e0dc4f64 -r 55ad072b8934 src/core/ngx_cycle.c
--- a/src/core/ngx_cycle.c  Thu Jul 25 12:46:03 2013 +0400
+++ b/src/core/ngx_cycle.c  Fri Jul 26 15:43:48 2013 +0800
         }
     }
     //*************************************************
     //此处为master 主进程调用init_cycle中不再创建监听socket
     //*************************************************
-    if (ngx_open_listening_sockets(cycle) != NGX_OK) {
-        goto failed;
-    }
+    if (!ccf->so_reuseport) {
+        if (ngx_open_listening_sockets(cycle) != NGX_OK) {
+            goto failed;
+        }
 
-    if (!ngx_test_config) {
-        ngx_configure_listening_sockets(cycle);
+        if (!ngx_test_config) {
+            ngx_configure_listening_sockets(cycle);
+        }
     }
 
 
diff -r 6d73e0dc4f64 -r 55ad072b8934 src/core/ngx_cycle.h
--- a/src/core/ngx_cycle.h  Thu Jul 25 12:46:03 2013 +0400
+++ b/src/core/ngx_cycle.h  Fri Jul 26 15:43:48 2013 +0800
      ngx_array_t              env;
      char                   **environment;
 
+     unsigned                 so_reuseport:1;
+
 #if (NGX_THREADS)
      ngx_int_t                worker_threads;
      size_t                   thread_stack_size;


diff -r 6d73e0dc4f64 -r 55ad072b8934 src/event/ngx_event.c
--- a/src/event/ngx_event.c Thu Jul 25 12:46:03 2013 +0400
+++ b/src/event/ngx_event.c Fri Jul 26 15:43:48 2013 +0800
     ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
     ecf = ngx_event_get_conf(cycle->conf_ctx, ngx_event_core_module);
     //***************************************************
     //增加逻辑判断，如果使用了reuseport 则 accetp_mutx锁不生效
     //***************************************************
-    if (ccf->master && ccf->worker_processes > 1 && ecf->accept_mutex) {
+    if (ccf->master && ccf->worker_processes > 1 && ecf->accept_mutex &&
+        !ccf->so_reuseport) {
         ngx_use_accept_mutex = 1;
         ngx_accept_mutex_held = 0;
         ngx_accept_mutex_delay = ecf->accept_mutex_delay;

         
diff -r 6d73e0dc4f64 -r 55ad072b8934 src/os/unix/ngx_process_cycle.c
--- a/src/os/unix/ngx_process_cycle.c Thu Jul 25 12:46:03 2013 +0400
+++ b/src/os/unix/ngx_process_cycle.c Fri Jul 26 15:43:48 2013 +0800
     ngx_core_conf_t  *ccf;
     ngx_listening_t  *ls;
 
+    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
     //**************************
     // 在各个子进程中创建监听socket
     //**************************
+    if (ccf->so_reuseport) {
+        ngx_open_listening_sockets(cycle);
+        ngx_configure_listening_sockets(cycle);
+    }
+
     if (ngx_set_environment(cycle, NULL) == NULL) {
         /* fatal */
         exit(2);
     }
 
-    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

``` 
