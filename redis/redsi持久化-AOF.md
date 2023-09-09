--- 
title: redsi持久化-AOF 
date: 2019-08-09
categories: 
- redis
---
> redsi持久化-AOF
- 除了RDB持久化之后，redis还提供了AOF持久化功能（append only file）
- AOF持久化使用过保存redis写命令来记录数据库状态的
- 在AOF文件中，除了指定数据库的select语句之外，其他的都是通过客户端发生的命令
# AOF持久化实现
> AOF持久化的实现是通过命令追加，文件写入，文件同步三个步骤实现的
## 命令追加
```c
struct redisServer {
    //aof缓冲区
    sds aof_buf;
}
```
- 在redisServer中保存了aof_buf的缓冲区，改缓冲区是由sds实现的
- 每次redis的写命令之后，都会讲命令追加到这个缓冲区中
## 文件写入和文件同步
- 对于redisserver来说，redis的服务进程就是一个事件循环
- 在每个时间循环的最后，都会调用flushAppendOnlyFile函数并评估是否需要将aof_buf中的内容写入文件
- 对于flushAppendOnlyFile函数来说，有三种行为：
     ![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118211414821_1262405935.png)
    - 现代操作系统中，文件写入操作并不会立即写入磁盘，而是会在文件缓存中，以一定频率写入到磁盘
    - 默认是everysec，每次循环结束，会将aof_buf写入文件缓冲区，当上次flush时间已经距现在超过一秒，就将缓冲区同步到磁盘
## 文件同步的安全性
- always 最多会丢失一个时间循环的命令
- everysec 会丢失一秒的命令
- no 以来操作系统何时将文件同步到磁盘
# AOF的载入
- 当redis服务启动时，服务器会读入AOF文件，并还原数据库状态
- 步骤
    - 由于命令需要运行在client的上下文，所以redis会创建一个伪客户端来执行命令
    - 从AOF文件读取一行
    - 伪客户端执行读出的命令
    - 知道AOF被读完
# AOF文件的重写
- 随着命令的执行，AOF文件会越来越大，会占用过多的服务器资源，如果不加以控制的话，会对宿主机造成影响
- 所以需要对AOF文件进行重写，redis服务器创建一个新的AOF文件替代就得AOF文件，两个文件包含的数据库状态完全相同，只是新的AOF文件不包含冗余的命令
- 实际上，redis并不会根据就得AOF文件进行重写，而是根据现有的数据库状态进行重写
## 后台重写任务
- 前台的AOF文件重写任务会带来大量的文件写入操作，所以导致单线程的redis阻塞
- 所以将AOF重写任务放入后台，bgRewriteAof会新起一个子进程来进行重写
    - 这样在重写期间redis服务可以正常响应请求
    - 而且相对于多线程来说，可以避免锁的使用
## AOF重写缓冲区
- 后台重写任务期间，redis也会产生新的AOF文件，这一部分命令需要一块AOF重写缓冲区进行存储
- 在后台重写任务期间，redis服务器每条写命令执行完之后，会将命令分别写入AOF缓冲区和AOF重写缓冲区
- 当AOF重写完成之后，会想redis发送一个信号，父进程会将AOF重写缓冲区中的命令写入新的AOF文件中
- 之后原子性的将新的AOF文件替换就得AOF文件
- 整个过程中，只有信号处理函数会打断redis对请求的响应，其他都不会阻碍redis正常的响应
# 重点回顾
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118213744086_118117185.png)











