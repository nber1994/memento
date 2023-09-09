--- 
title: nginx平滑重启 
date: 2019-08-09
categories: 
- nginx 
---
# nginx-平滑重启
> 本文转载自软件编程之路公众号文章：[深入剖析nginx平滑重启](https://mp.weixin.qq.com/s?__biz=MzIxNzg5ODE0OA==&mid=2247483660&idx=1&sn=b78b07de401eb7c08b6b7ba70c68cf00&chksm=97f38cc7a08405d18ecca5cf9af523b1d2f3294a603a08d810664aa4986b481eda82eee78626&utm_medium=hao.caibaojian.com)

# 一、背景
在服务器开发过程中，难免需要重启服务加载新的代码或配置，如果能够保证server重启的过程中服务不间断，那重启对于业务的影响可以降为0。最近调研了一下nginx平滑重启，觉得很有意思，记录下来供有兴趣的同学查阅。

# 二、重启流程
* 重启意味着新旧接替，在交接任务的过程中势必会存在新旧server并存的情形，因此，重启的流程大致为：
    1. 启动新的server
    2. 新旧server并存，两者共同处理请求，提供服务
    3. 旧的server处理完所有的请求之后优雅退出

* 这里，最主要的问题在于如何保证新旧server可以并存，如果重启前后的server端口一致，如何保证两者可以监听同一端口。

# 三、nginx实现
1. 为了验证nginx平滑重启，笔者首先尝试nginx启动的情形下再次开启一个新的server实例，结果如图：
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20190524111302828_2075783423.png)

很明显，重新开启server实例是行不通的，原因在于新旧server使用了同一个端口80，在未开始socket reuseport选项复用端口时，bind系统调用会出错。nginx默认bind重试5次，失败后直接退出。而nginx需要监听IPV4地址0.0.0.0和IPV6地址[::]，故图中打印出10条emerg日志。    

2. 接下来就开始尝试平滑重启命令了，一共两条命令：
````
kill -USR2 `cat /var/run/nginx.pid`
kill -QUIT `cat /var/run/nginx.pid.oldbin`
````
* 第一条命令是发送信号USR2给旧的master进程，进程的pid存放在/var/run/nginx.pid文件中，其中nginx.pid文件路径由nginx.conf配置。
* 第二条命令是发送信号QUIT给旧的master进程，进程的pid存放在/var/run/nginx.pid.oldbin文件中，随后旧的master进程退出。

那么问题来了，为什么旧的master进程的pid存在于两个pid文件之中？事实上，在发送信号USR2给旧的master进程之后，旧的master进程将pid重命名，原先的nginx.pid文件rename成nginx.pid.oldbin。这样新的master进行就可以使用nginx.pid这个文件名了。    

先执行第一条命令，结果如图：     
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20190524111338510_269875207.png)

不错，新旧master和worker进程并存了。 再来第二条命令，结果如图:     
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20190524111356626_307141585.png)

如你所见，旧的master进程8527和其worker进程全部退出，只剩下新的master进程12740。    

不由得产生困惑，为什么手动开启一个新的实例行不通，使用信号重启就可以达到。先看下nginx log文件：     
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20190524111415412_1725415354.png)

除了之前的错误日志，还多了一条notice，意思就是继承了sockets，fd值为6，7。 随着日志翻看nginx源码，定位到nginx.c/ngx_exec_new_binary函数之中，    

````c
ngx_pid_t
ngx_exec_new_binary(ngx_cycle_t *cycle, char *const *argv)
{
    ...
    ctx.path = argv[0];
    ctx.name = "new binary process";
    ctx.argv = argv;
    n = 2;
    env = ngx_set_environment(cycle, &n);
...
    var = ngx_alloc(sizeof(NGINX_VAR)
                    + cycle->listening.nelts * (NGX_INT32_LEN + 1) + 2,
                    cycle->log);
...
    p = ngx_cpymem(var, NGINX_VAR "=", sizeof(NGINX_VAR));
    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {
        p = ngx_sprintf(p, "%ud;", ls[i].fd);
    }
    *p = '\0';
    env[n++] = var;
...
    env[n] = NULL;
...
    ctx.envp = (char *const *) env;
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    if (ngx_rename_file(ccf->pid.data, ccf->oldpid.data) == NGX_FILE_ERROR) {
       ...
        return NGX_INVALID_PID;
    }
    pid = ngx_execute(cycle, &ctx);
    if (pid == NGX_INVALID_PID) {
        if (ngx_rename_file(ccf->oldpid.data, ccf->pid.data)
            == NGX_FILE_ERROR)
        {
            ...
        }
    }
...
    return pid;
}
````

* 函数的流程为

1. 将旧的master进程监听的所有fd，拷贝至新master进程的env环境变量NGINX_VAR。
2. rename重命名pid文件
3. ngx_execute函数fork子进程，execve执行命令行启动新的server。
4. 在server启动流程之中，涉及到环境变量NGINX_VAR的解析，ngx_connection.c/ngx_add_inherited_sockets具体代码为:

````c
static ngx_int_t
ngx_add_inherited_sockets(ngx_cycle_t *cycle)
{
...
    inherited = (u_char *) getenv(NGINX_VAR);
    if (inherited == NULL) {
        return NGX_OK;
    }
    if (ngx_array_init(&cycle->listening, cycle->pool, 10,
                       sizeof(ngx_listening_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }
    for (p = inherited, v = p; *p; p++) {
        if (*p == ':' || *p == ';') {
            s = ngx_atoi(v, p - v);
            ...
            v = p + 1;
            ls = ngx_array_push(&cycle->listening);
            if (ls == NULL) {
                return NGX_ERROR;
            }
            ngx_memzero(ls, sizeof(ngx_listening_t));
            ls->fd = (ngx_socket_t) s;
        }
    }
    ...
    ngx_inherited = 1;
    return ngx_set_inherited_sockets(cycle);
}
````

* 函数流程为：

1. 解析环境变量NGINX_VAR的值，获取fd存入数组
2. fd对应的socket设为ngx_inherited，保存这些socket的信息。

**也就是说，新的server压根就没重新bind端口listen，这些fd状态和值都是新的master进程fork时带过来的,新的master进程监听处理继承来的文件描述符即可，这里比较关键的一点在于listen socket文件描述符通过ENV传递。**

