---
title: lnmp报错too-big-header错误探究
date: 2018-01-08
categories: 
- php
---

> 之前遇到的一个偶发的错误，探讨下到底是怎么肥四

# 1.背景

遇到nginx报出了如下的error，导致服务为502 bad gateway, 而且某些请求稳定复现，但是其他请求缺没有问题，正常返回,正常情况下，php的warning并不会导致流程的中断

```php
2018/01/08 15:50:20 [error] 15477#0: *2941700922 upstream sent too big header while reading response header from upstream, client: xx.xx.xxx, server: test.com, request: "POST /test/test HTTP/1.1", upstream: "fastcgi://127.0.0.1:xxxx", host: "xxxxxx", referrer: "xxxxxx"
```

一时之间不知道原因，唯一有异常的是warninig日志较多，推测可能是php-fpm会把warning日志加到response header头里,导致头部过大报错

php代码中存在类似与以下的错误片段：

```php
<?php
    $a = [
        'a' => '123',
        'b' => '123',
        'c' => '123',
        'd' => '123',
    ];

    foreach ($a['nber1994'] as $item) {
        /**
            do something...
        */    
    }
```

此时会报warning的错误，类似于

```php
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
php message: php Warning:  Invalid argument supplied for foreach() in XXXXX.php on line 206
```

**大量的warning如果会被加到header头里的话，会触发nginx的相关配置(如果有的话),导致报错,看起来解释的通**
但是是否真的是因为header头过大导致的呢，探究一下

# 2.简单的分析一次请求的日志行为

该情况是线上问题，无法抓取到问题发生时，php-fpm与nginx之间到底发生了什么，所以准备在自己的开发环境复现, 不过首先需要能够对nginx与php-fpm进程间通信的数据进行抓取

## 2.1 抓取php-fpm与nginx之间的通信数据

### 2.1.1 php监听端口通信方式的监听

速度较unix socket较慢，但是支持跨机器调用，我们可以通过tcpdump进行抓取

```shell
tcpdump -X -i any  port 9999 > /tmp/tcpdump.log
```

#### ps:开放php端口对外提供fastcgi服务的方法

```shell
1.设置防火墙,开放你的端口
2.php-fpm配置增加：
listen = 公网IP:9085
listen.allowed_clients = 192.168.10.66 #设置允许连接到 FastCGI 的服务器 IPV4 地址。如果允许所有那么把这条注释掉即可
```

### 2.1.2 以unix socket方式通信数据的抓取

该方法速度较快，性能高,可以通过监听socket通道得到数据

```shell
cd /path/to/php.sock
mv php.sock php.sock.orig
socat -t100 -x -v UNIX-LISTEN:php.sock,mode=777,reuseaddr,fork UNIX-CONNECT:php.sock.orig
将socket通道的内容转发一份到另一个socket,进行监听
```

### 2.1.3 抓取到的数据大概的样子

![抓到的数据](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/tcpdump.png)

## 2.2 简单分析

让我们观察下日志，
首先可以看到tcp的建立链接数据，可以顺便直观了解下tcp的三次握手:P

```shell
12:09:25.581210 IP 127.0.0.1.58984 > 127.0.0.1.9999: Flags [S], seq 3550696989, win 65495, options [mss 65495,sackOK,TS val 1133088550 ecr 0,nop,wscale 7], length 0
        0x0000:  4500 003c d93b 4000 4006 637e 7f00 0001  E..<.;@.@.c~....
        0x0010:  7f00 0001 e668 270f d3a3 561d 0000 0000  .....h'...V.....
        0x0020:  a002 ffd7 4619 0000 0204 ffd7 0402 080a  ....F...........
        0x0030:  4389 8f26 0000 0000 0103 0307            C..&........
12:09:25.581249 IP 127.0.0.1.9999 > 127.0.0.1.58984: Flags [S.], seq 457907882, ack 3550696990, win 65483, options [mss 65495,sackOK,TS val 1133088550 ecr 1133088550,nop,wscale 7], length 0
        0x0000:  4500 003c 0000 4000 4006 3cba 7f00 0001  E..<..@.@.<.....
        0x0010:  7f00 0001 270f e668 1b4b 1eaa d3a3 561e  ....'..h.K....V.
        0x0020:  a012 ffcb 396f 0000 0204 ffd7 0402 080a  ....9o..........
        0x0030:  4389 8f26 4389 8f26 0103 0307            C..&C..&....
12:09:25.581261 IP 127.0.0.1.58984 > 127.0.0.1.9999: Flags [.], ack 1, win 512, options [nop,nop,TS val 1133088550 ecr 1133088550], length 0
        0x0000:  4500 0034 d93c 4000 4006 6385 7f00 0001  E..4.<@.@.c.....
        0x0010:  7f00 0001 e668 270f d3a3 561e 1b4b 1eab  .....h'...V..K..
        0x0020:  8010 0200 602b 0000 0101 080a 4389 8f26  ....`+......C..&
        0x0030:  4389 8f26                                C..&
```

### 2.2.1 一次请求的nginx-error日志

![nginx-error](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/one-hit-error-log.png)

### 2.2.2 一次请求的tcpdump的日志

![tcpdump-log](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/one-hit-tcpdump-log.png)

### 2.2.3 简单结论

#### 观察到的现象

* php-fpm确实将warning与error日志都传给了nginx
* tcpdump的日志，明显响应头大于fastcgi_buffer_size设置的大小，但是并未报错
* nginx-error日志，可以明显的发现，日志是一段一段的，大小和fastcgi_buffer_size的大小相当，

#### 得到的初步结论

对比nginx日志与tcpdump的日志的量，我们能看到:

1.**php-fpm确实会将warning与error传递给nginx**

2.**不管fastcgi_buffer_size设置了多少，其实是根据fastcgi_buffer_size的大小来一段一段的读取php-fpm的响应头，不管传了多少的header，都会按照fastcgi_buffer_size的大小，一段一段的读取，然后写入到nginx的日志**

> 得到了上述结论，我们大致可以确定，并不是因为header头过大，或者说php-fpm传递给nginx的数据超过fastcgi_buffer_size的值才报出的502,那到底是为什么会报相关错误呢？这就需要探究下nginx内部了

# 3.nginx内部发生了什么

首先我们知道：

* nginx发送请求数据与接收来自后端服务器的响应可以同时进行，是一种全双工方式；
* nginx接收到了后端服务器的响应，什么时候向客户端转发响应呢? 对于http响应头部，nginx只有在接收到了所有来自后端服务器的响应头部后，才会转发给客户端。对于http响应包体，则nginx边接收后端服务器的响应包体，边发给客户端

## 3.1 fastcgi_buffer_size

我们知道，fastcgi_buffer_size与proxy_buffer_size这两个参数会影响到响应头, 其中proxy_buffer_size影响的是nginx作为反向代理时的参数
对于nginx配置文件中的fastcgi_buffer_size，文档中是这么写的

![文档](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/fastcgi_buffer_size.png)

其中 **the first part of the response received from the FastCGI server.**,这个参数指定的是接受到fastcgi-server端的第一部分的响应（一般是response header），在lnmp的场景里，fastcgi-server就是php部分
其中这个first part的含义是这样子的，由于upstream是一个通用的组件，它不知道后端的协议，而对于http场景来说，由于http是需要header的，而后端的协议不一定有头，此时就需要我们通过解析后端的协议，然后来设置好发送给client的头，最终发送给client，通过上面的观察，我们发现php的错误信息也会包含其中 

以下是所谓的正常情况下的**the first part of the response**的结构

![header](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/fastcgi-header.png)

* 对于接收到的来自后端服务器的响应头部。以上是一个fastcgi格式的响应报文，需要从fastcgi格式的报文中提取出http响应头部，因此需要经过状态机处理，从而解析出http响应头部。
* nginx接收后端服务的响应也应该是一种fastcgi报文格式。从图中可以看出，每一个http响应包头前面都加上了fastcgi头部，而每一个http响应包头都有可能由多条http响应头部组成。
* 我们暂且称每一个fastcgi头部 + http响应包头为一个组， nginx使用扫描法，扫描每一组。每一个组使用两个状态机，一个状态机解析这个组内的fastcgi头部， 另一个状态机解析这个组内的http响应包头，从而获取到每一个http响应头部。

## 3.2 nginx代码报错分析

源码中报错的位置如下

![error](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/code.png)

```c
//nginx接收来自上游服务器的响应头部
static void ngx_http_upstream_process_header(ngx_http_request_t *r, ngx_http_upstream_t *u)
{
    //循环接收来自后端服务器的响应头部，并使用状态机解析
    for ( ;; )
    {
        n = c->recv(c, u->buffer.last, u->buffer.end - u->buffer.last);

        //表示还需要再次接收来自上游服务器的响应,重新注册读事件到epoll中
        if (n == NGX_AGAIN)
        {
            ngx_handle_read_event(c->read, 0);
            return;
        }

        //更新接收缓冲区
        u->buffer.last += n;

        //调用http模块的方法，解析响应头部
        //fastcgi为: ngx_http_fastcgi_process_header
        rc = u->process_header(r);

        //表示包头还没有完全接收到
        if (rc == NGX_AGAIN)
        {
            //已经读取到末尾
            if (u->buffer.last == u->buffer.end) {
                ngx_log_error(NGX_LOG_ERR, c->log, 0,
                        "upstream sent too big header");

                ngx_http_upstream_next(r, u,
                        NGX_HTTP_UPSTREAM_FT_INVALID_HEADER);
                return;
            }
            continue;
        }
    }
}
```

可以看到，当处理头部的会调函数process_header返回NGX_AGAIN,同时已经读取到了response的末尾，则会报错

### 3.2.1处理头部的回调函数process_header伪代码

```c
//解析来自后端服务器发来的响应头部，从fastcgi格式转为nginx格式。
//采用两个状态机，一个解析fastcgi头部，一个解析fastcgi响应包头
static ngx_int_t ngx_http_fastcgi_process_header(ngx_http_request_t *r)
{
    //循环解析每一个组(fastcgi头部 + 多个http响应头部组成的数据)。因为每次从内核接收到的数据有可能包含多个组
    for ( ;; )
    {
        //解析fastcgi的头部
        if (f->state < ngx_http_fastcgi_st_data)
        {
            f->pos = u->buffer.pos;
            f->last = u->buffer.last;

            //使用状态解析fastcgi的头部
            rc = ngx_http_fastcgi_process_record(r, f);
            u->buffer.pos = f->pos;
            u->buffer.last = f->last;
        }

        //这个循环用于解析这个组内的每一个http响应头部。因此该组内的http响应包头可能由多个http响应头部组成。
        //而ngx_http_parse_header_line状态机每次只能解析一个http响应头部，因此需要循环解析
        for ( ;; )
        {
            //使用状态机解析来自后端服务器发来的http响应头部
            rc = ngx_http_parse_header_line(r, &u->buffer, 1);
            if (rc == NGX_OK)
            {
                //解析某个http响应头部成功后，保存到数组链表中
                /* a header line has been parsed successfully */
                h = ngx_list_push(&u->headers_in.headers);

                //保存http响应头部
                ngx_memcpy(h->key.data, r->header_name_start, h->key.len);
                //保存http响应头部的值
                ngx_memcpy(h->value.data, r->header_start, h->value.len);
                h->hash = r->header_hash;

                //如果是负责均衡模块支持的头部，则把常用头部的指针指向数组链表中相应位置
                hh = ngx_hash_find(&umcf->headers_in_hash, h->hash,
                                   h->lowcase_key, h->key.len);
                hh->handler(r, h, hh->offset);
            }

            //解析完成所有的响应头部，则保存响应状态码
            if (rc == NGX_HTTP_PARSE_HEADER_DONE)
            {
                if (u->headers_in.status)
                {
                    //后端服务器返回了状态码，则优先使用后端服务的响应状态码
                }
                else if (u->headers_in.location)
                {
                    //后端服务器返回了location,则需要给客户端返回302重定向
                }
                else
                {
                    //否则nginx构造200 0k返回给客户端
                    u->headers_in.status_n = 200;
                    ngx_str_set(&u->headers_in.status_line, "200 OK");
                }
            }
        }
    }
}
```

以上是流程代码，具体细节未展示
以上代码会对http与fastcgi的header分别进行解析

**当两个状态机对fastcgi与http头解析如果出错的话（比如header头缺失）,也会返回NGX_AGAIN，如果此时读取到php-fpm返回的header的尾部时，则会报错**
那到底是否是这个原因呢？php-fpm传送了残缺的或者没有传递response header，才导致的报错？我们复现下观察下

# 4.尝试复现

如果可以复现出502，我们就能验证我们的猜想
不断的增大warning的数目，并监听端口与日志，截取502发生时，php-fpm到底传了什么给nginx
出问题时，是因为过多的warning写入到了fastcgi_buffer_size中，我们可以按着这个思路，进行强行复现

## 4.1 php脚本

我们可以尝试着一步步加大error的size，来观察发送502时的日志情况
以下是脚本:

```php
<?php
for ($i = 0; $i<$_GET['iterations']; $i++)
        error_log(str_pad("a", $_GET['size'], "a"));
echo "got here\n";
```

## 4.2 日志监听

同时监听php绑定的端口，得到日志(此时，fastcgi_buffer_size 4K)

## 4.3 压测方法

使用循环对脚本进行请求，不断的增加单条日志的size与迭代次数
，使其向日志里写入更多的字节，同时过滤返回的502以及报错时的迭代次数与每次日志的大小

```shell
bash~ for it in {30..200..3}; do for size in {100..250..3}; do echo "size=$size iterations=$it $(curl -sv "http://localhost/debug.php?size=$size&iterations=$it" 2>&1 | egrep '^< HTTP')"; done; done | grep 502 | head

size=121 iterations=30 < HTTP/1.1 502 Bad Gateway
size=109 iterations=33 < HTTP/1.1 502 Bad Gateway
size=241 iterations=48 < HTTP/1.1 502 Bad Gateway
size=226 iterations=51 < HTTP/1.1 502 Bad Gateway
size=190 iterations=60 < HTTP/1.1 502 Bad Gateway
size=223 iterations=69 < HTTP/1.1 502 Bad Gateway
size=127 iterations=87 < HTTP/1.1 502 Bad Gateway
size=199 iterations=96 < HTTP/1.1 502 Bad Gateway
size=151 iterations=99 < HTTP/1.1 502 Bad Gateway
size=106 iterations=102 < HTTP/1.1 502 Bad Gateway 
```

**顺便说下，unix socket方式性能真的很高，这种unix socket通信的情况下，测试语句运行完了也没有报502**
由上可知，我们复现了502,下面我们对日志进行紧张的分析

## 4.4 结果分析

### 报错的nginx日志

```shell
2018/01/09 11:39:01 [error] 4207#0: *2045 FastCGI sent in stderr: "php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
2018/01/09 11:39:01 [error] 4207#0: *2045 upstream sent too big header while reading response header from upstream, client: 10.30.128.251, server: testtest, request: "GET /debug.php?size=121&iterations=30 HTTP/1.1", upstream: "fastcgi://127.0.0.1:9999", host: "test.com"
```

可以看到我们成功复现了too big header报错:P

### 正常的nginx-error日志

```shell
2018/01/09 11:39:02 [error] 4207#0: *2353 FastCGI sent in stderr: "php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
php message: aaaaaaaaaaaaa
```

### 紧张分析

上面的日志，一个是39:01，一个是39:02的，按照我们的测试脚本，时间越靠后，传递的日志量会越大(至少是之前的3倍)，但是为什么数据量较小的39:01秒的日志触发了502，而数据量较大的39:02没有报错，而且，日志的量也是相同的，明显有截断，明显超出了4K，这也能验证我们的推测，其实fastcgi_buffer_size并不是一个检查值，超过这个值并不会报错。而更像是一个暂存的空间

### 进一步紧张分析

我们回去看下之前监控的tcpdump的日志,发现了一些有趣的现象

一次正常的日志

```shell
        0x0cf0:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x0d00:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x0d10:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x0d20:  6161 6161 610a 5048 5020 6d65 7373 6167  aaaaa.php.messag
        0x0d30:  653a 2061 6161 6161 6161 6161 6161 6161  e:.aaaaaaaaaaaaa
        0x0d40:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x0d50:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x0d60:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x0d70:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x0d80:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x0d90:  6161 6161 6161 610a 0000 0000 0106 0001  aaaaaaa.........
        0x0da0:  0024 0400 436f 6e74 656e 742d 7479 7065  .$..Content-type
        0x0db0:  3a20 7465 7874 2f68 746d 6c0d 0a0d 0a67  :.text/html....g
        0x0dc0:  6f74 2068 6572 650a 0000 0000 0103 0001  ot.here.........
        0x0dd0:  0008 0000 0000 0000 0061 6161            .........aaa
        12:09:25.585035 IP 127.0.0.1.9999 > 127.0.0.1.58984: Flags [F.], seq 3497, ack 721, win 523, options [nop,nop,TS val 1133088554 ecr 1133088551], length 0
        0x0000:  4500 0034 918f 4000 4006 ab32 7f00 0001  E..4..@.@..2....
        0x0010:  7f00 0001 270f e668 1b4b 2c53 d3a3 58ee  ....'..h.K,S..X.
        0x0020:  8011 020b 4fa2 0000 0101 080a 4389 8f2a  ....O.......C..*
        0x0030:  4389 8f27                                C..'
```

一次502的日志

```shell
        0x1f30:  6d65 7373 6167 653a 2061 6161 6161 6161  message:.aaaaaaa
        0x1f40:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x1f50:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x1f60:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x1f70:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x1f80:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x1f90:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x1fa0:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x1fb0:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x1fc0:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x1fd0:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x1fe0:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x1ff0:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x2000:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x2010:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x2020:  6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
        0x2030:  6161 610a                                aaa.
12:09:43.315839 IP 127.0.0.1.36564 > 127.0.0.1.9999: Flags [.], ack 40961, win 768, options [nop,nop,TS val 1133106285 ecr 1133106285], length 0
        0x0000:  4500 0034 c6d5 4000 4006 75ec 7f00 0001  E..4..@.@.u.....
        0x0010:  7f00 0001 8ed4 270f 363a 1573 0df7 5365  ......'.6:.s..Se
        0x0020:  8010 0300 e2df 0000 0101 080a 4389 d46d  ............C..m
        0x0030:  4389 d46d                                C..m
```

发现了吗，502报错的情况下，tcpdump抓到的包中，末尾缺少了http的response header,这正验证了我们之前的推测，php-fpm在返回body之前，并没有传递完整的response header给nginx,导致nginx报出来错误

## 4.5 暂时的结论

观察上面的日志，报错的最后的响应头不完整或者根本就没有,这个正好会进入到nginx中未解析到header头并且已到header尾部的情况，触发报错
所以，暂时现在可以推断的是，

**php-fpm会在一定的情况下向nginx传送不完整的响应头数据，导致nginx解析fastcgi与http的header出错，导致报出502**
这看起来不可思议，竟然是下游服务出了差错，而不是因为nginx内的fastcgi_buffer_size太小导致的错误,不过该问题并不是不可能发生：
![哈哈](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/gis.png)

# 5. 探究下不同fastcgi_buffer_size下的502发生情况

## 5.1 fastcgi_buffer_size 1k

### nginx日志

不会报502的错误日志
![error-log](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/1k-error-log.png)

会报502的错误日志
![error-log](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/1k-502-error-log.png)

### tcpdump的日志

不会报502的错误日志
![tcp-error](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/1k-tcpdump-log.png)

会报502的错误日志
![tcp-error](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/1k-tcpdump-502-log.png)

### 测试结果

```shell
bash~ for it in {30..200..3}; do for size in {100..250..3}; do echo "size=$size iterations=$it $(curl -sv "http://test/debug.php?size=$size&iterations=$it" 2>&1 | egrep '^< HTTP')"; done; done | grep 502 | head
size=121 iterations=30 < HTTP/1.1 502 Bad Gateway
size=190 iterations=30 < HTTP/1.1 502 Bad Gateway
size=109 iterations=33 < HTTP/1.1 502 Bad Gateway
size=202 iterations=33 < HTTP/1.1 502 Bad Gateway
size=127 iterations=36 < HTTP/1.1 502 Bad Gateway
size=184 iterations=36 < HTTP/1.1 502 Bad Gateway
size=241 iterations=36 < HTTP/1.1 502 Bad Gateway
size=169 iterations=39 < HTTP/1.1 502 Bad Gateway
size=205 iterations=42 < HTTP/1.1 502 Bad Gateway
size=229 iterations=42 < HTTP/1.1 502 Bad Gateway
```

## 5.2 fastcgi_buffer_size 2k

### nginx日志

不会报502的错误日志
![error-log](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2k-error-log.png)

会报502的错误日志
![error-log](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2k-502-error-log.png)

### tcpdump的日志

不会报502的错误日志
![tcp-error](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2k-tcpdump-log.png)

会报502的错误日志
![tcp-error](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2k-tcpdump-error-502-log.png)

### 测试结果

```shell
bash~ for it in {30..200..3}; do for size in {100..250..3}; do echo "size=$size iterations=$it $(curl -sv "http://test/debug.php?size=$size&iterations=$it" 2>&1 | egrep '^< HTTP')"; done; done | grep 502 | head
size=121 iterations=30 < HTTP/1.1 502 Bad Gateway
size=190 iterations=30 < HTTP/1.1 502 Bad Gateway
size=109 iterations=33 < HTTP/1.1 502 Bad Gateway
size=229 iterations=42 < HTTP/1.1 502 Bad Gateway
size=241 iterations=48 < HTTP/1.1 502 Bad Gateway
size=106 iterations=51 < HTTP/1.1 502 Bad Gateway
size=226 iterations=51 < HTTP/1.1 502 Bad Gateway
size=175 iterations=54 < HTTP/1.1 502 Bad Gateway
size=190 iterations=60 < HTTP/1.1 502 Bad Gateway
size=148 iterations=63 < HTTP/1.1 502 Bad Gateway
```

## 5.3 fastcgi_buffer_size 4k

### nginx日志

不会报502的错误日志
![error-log](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/4k-error-log.png)

会报502的错误日志
![error-log](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/4k-error-502-log.png)

### tcpdump的日志

不会报502的错误日志
![tcp-error](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/4k-tcpdump-error-log.png)

会报502的错误日志
![tcp-error](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/4k-tcpdump-502-error-log.png)

### 测试结果

```shell
bash~ for it in {30..200..3}; do for size in {100..250..3}; do echo "size=$size iterations=$it $(curl -sv "http://test/debug.php?size=$size&iterations=$it" 2>&1 | egrep '^< HTTP')"; done; done | grep 502 | head
size=121 iterations=30 < HTTP/1.1 502 Bad Gateway
size=109 iterations=33 < HTTP/1.1 502 Bad Gateway
size=241 iterations=48 < HTTP/1.1 502 Bad Gateway
size=226 iterations=51 < HTTP/1.1 502 Bad Gateway
size=190 iterations=60 < HTTP/1.1 502 Bad Gateway
size=223 iterations=69 < HTTP/1.1 502 Bad Gateway
size=127 iterations=87 < HTTP/1.1 502 Bad Gateway
size=199 iterations=96 < HTTP/1.1 502 Bad Gateway
size=151 iterations=99 < HTTP/1.1 502 Bad Gateway
size=106 iterations=102 < HTTP/1.1 502 Bad Gateway
```

## 5.4 观察发现

由上1k，2k，4k的比对结果，我们可以得出以下结论

综合以上502报错时的size与iteration的值的乘积，我们可以发现，不同fastcgi_buffer_size下，其实发生502错误时的php-fpm传给nginx的数据的大小基本是一样的
**提高fastcgi_buffer_size的大小，并不能减少too big header的发生风险**
但是建议是设置成操作系统分页的大小

# 6. 结论

经过探究，我们最终可以得到:

1.**php-fpm确实会将warning与error传递给nginx**

2.**不管fastcgi_buffer_size设置了多少，其实是根据fastcgi_buffer_size的大小来一段一段的读取php-fpm的响应头，不管传了多少的header，都会按照fastcgi_buffer_size的大小，一段一段的读取，然后写入到nginx的日志**

3.**php-fpm会在一定的情况下向nginx传送不完整的响应头数据，导致nginx解析fastcgi与http的header出错，导致报出502**

4.**提高fastcgi_buffer_size的大小，并不能减少too big header的发生风险**

# 7. 疑问

经过以上的探究，我们可以看到以上的结论，但是仍有很多疑问

1.为什么php-fpm发送的数据会缺少header

2.有没有其它情况会导致too big header报错

3.线上问题是仅有特定的请求参数会稳定复现，其他的则不会复现，到底发生了什么？是我们探究的原因导致的吗？

以上的疑问待进一步的探究

# 8. 对疑问进一步分析

经过老司机的指点，对以上压测得出的数据进行进一步的分析，
上面我们探究了在不同fastcgi_buffer_size下的502出错情况

1K

```shell
bash~ for it in {30..200..3}; do for size in {100..250..3}; do echo "size=$size iterations=$it $(curl -sv "http://test/debug.php?size=$size&iterations=$it" 2>&1 | egrep '^< HTTP')"; done; done | grep 502 | head
size=121 iterations=30 < HTTP/1.1 502 Bad Gateway
size=190 iterations=30 < HTTP/1.1 502 Bad Gateway
size=109 iterations=33 < HTTP/1.1 502 Bad Gateway
size=202 iterations=33 < HTTP/1.1 502 Bad Gateway
size=127 iterations=36 < HTTP/1.1 502 Bad Gateway
size=184 iterations=36 < HTTP/1.1 502 Bad Gateway
size=241 iterations=36 < HTTP/1.1 502 Bad Gateway
size=169 iterations=39 < HTTP/1.1 502 Bad Gateway
size=205 iterations=42 < HTTP/1.1 502 Bad Gateway
size=229 iterations=42 < HTTP/1.1 502 Bad Gateway
```

2k

```shell
bash~ for it in {30..200..3}; do for size in {100..250..3}; do echo "size=$size iterations=$it $(curl -sv "http://test/debug.php?size=$size&iterations=$it" 2>&1 | egrep '^< HTTP')"; done; done | grep 502 | head
size=121 iterations=30 < HTTP/1.1 502 Bad Gateway
size=190 iterations=30 < HTTP/1.1 502 Bad Gateway
size=109 iterations=33 < HTTP/1.1 502 Bad Gateway
size=229 iterations=42 < HTTP/1.1 502 Bad Gateway
size=241 iterations=48 < HTTP/1.1 502 Bad Gateway
size=106 iterations=51 < HTTP/1.1 502 Bad Gateway
size=226 iterations=51 < HTTP/1.1 502 Bad Gateway
size=175 iterations=54 < HTTP/1.1 502 Bad Gateway
size=190 iterations=60 < HTTP/1.1 502 Bad Gateway
size=148 iterations=63 < HTTP/1.1 502 Bad Gateway
```

4k

```shell
bash~ for it in {30..200..3}; do for size in {100..250..3}; do echo "size=$size iterations=$it $(curl -sv "http://test/debug.php?size=$size&iterations=$it" 2>&1 | egrep '^< HTTP')"; done; done | grep 502 | head
size=121 iterations=30 < HTTP/1.1 502 Bad Gateway
size=109 iterations=33 < HTTP/1.1 502 Bad Gateway
size=241 iterations=48 < HTTP/1.1 502 Bad Gateway
size=226 iterations=51 < HTTP/1.1 502 Bad Gateway
size=190 iterations=60 < HTTP/1.1 502 Bad Gateway
size=223 iterations=69 < HTTP/1.1 502 Bad Gateway
size=127 iterations=87 < HTTP/1.1 502 Bad Gateway
size=199 iterations=96 < HTTP/1.1 502 Bad Gateway
size=151 iterations=99 < HTTP/1.1 502 Bad Gateway
size=106 iterations=102 < HTTP/1.1 502 Bad Gateway
```

## 8.1 观察

对比上述结果，有没有发现什么诡异？
为什么三次压测的结果，得到的502次数都是10次
为什么第一次报错都是在size=121，iteration=30的情况下发生的？

## 8.2 进一步分析结果

我们把每一次的size与iteration进行相乘，我们可以得到如下的一些结果：(添加上开头的php message: 这13个字符)

```shell
(121 + 13) * 30 = 4020
(109 + 13) * 33 = 4026
(241 + 13) * 48 = 12192
(226 + 13) * 51 = 12189
(190 + 13) * 60 = 12180
.....
```

我们知道1024是程序员日，那上面的数据有什么敏感的吗? 

```shell
1024 * 4 = 4096,
4096 * 3 = 12288
....
```

# 甩锅

对比上面的数据，似乎是php的日志在某一个nk大小的时候(我的测试机器上很可能是4k), 会稳定复现这个问题呢？
这也很好的解释了为什么线上的某些请求会稳定复现这种问题，而其他请求，即使打了更多的php warning日志也不会触发502报警的现象

这回，我们可以把锅甩给php-fpm，我们可以不负责任的胡乱猜测一下，

**是否php-fpm内部也存在类似的buffer(在我的测试机器上也许是4k)，也许这段buffer是tcp发送消息的缓冲区，当发送的字节很接近4k时, 就会触发某些bug，导致某些尾部数据的丢失, 而当大小与4k的整数倍的数据相差比较大的数据，则不会丢失数据, emmm，也许是在重新分配buffer的时候出了问题，导致的数据丢失:P**

# 参考

部分内容参考自以下内容，感谢以下作者对知识的分享
http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html
http://www.pagefault.info/?p=273
http://blog.csdn.net/ApeLife/article/details/78001508?ref=myrecommend
http://blog.csdn.net/midion9/article/details/51384002
https://www.imooc.com/article/19278
https://stackoverflow.com/questions/23844761/upstream-sent-too-big-header-while-reading-response-header-from-upstream

> 以上探究仅是个人观点，可能存在错误，欢迎交流~           eamil:m18710895391@163.com
