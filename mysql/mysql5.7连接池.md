--- 
title: mysql-5.7连接池 
date: 2019-01-08
categories: 
- mysql 
---
> mysql5.7之后，引进了线程池的概念

# 为什么要是有线程池
## 没有线程池的现象
- 随着DB访问的增大，DB的响应时间也会随之增大
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181101110752375_1220949134.png)
- 而当DB访问达到一定程度的时候，DB吞吐量也会出现下降，并且会越来越差
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181101110859422_1674643821.png)

## 原因
## 减少线程重复创建与销毁的开销，提高性能
线程池采用预线程技术，在监听到新的请求到来时，会从线程池中分配一个线程来处理，  
处理后该线程不会销毁而是去处理其他请求
这样避免了频繁的线程的创建和销毁，减少了上线文切换
## 对系统起到保护作用
线程池限制了并发线程数，相当于限制了runing的线程数。  
不论出现多大的请求，超过最大线程都需要排队，所以讲系统保持在高性能的状态， 防止DB雪崩现象

## 线程池和连接池的区别
连接池一般是在客户端设置的，避免连接的频繁销毁和创建，但是并不能起到对DB的保护作用
所以一般线程池和连接池一起使用

# Mysql线程池介绍
## 简介
为了应付频繁创建销毁线程的开销以及高并发导致的雪崩问题
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181101111906731_304857661.png)

## 架构
线程池被划分为多个组group，每个组下面有对应的工作线程
## 架构图
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181101112056619_915771195.png)

## thread pool 组成
线程池由一个timer thread线程和多个线程组组成，而每个线程组又是由两个队列，已和listener线程和多个worker线程组成
### 队列(高优先级队列和低优先级队列)
高优先级队列会优先存放事务中的语句,同时如果一个任务存在于低优先级太长时间也会到高优先级里。如果是非事务队列，  
或者是autocommit的事务，都会直接放在低优先级队列
### listener线程
listener线程监听该group的语句，当队列中存在待处理的任务，说明这时有worker在工作，那么直接把任务放入队列中。  
如果任务队列为空，则将自己转变为worker进程，进行处理。这么做主要是为了减少创建线程的开销
### worker线程
真正干活的线程
### timer线程
### 检查阻塞
- timer线程周期性的检查group是否处理阻塞状态,当出现阻塞，会唤醒线程或者新建线程来解决  
> 检查方法：worker线程去检查队列时，会将queue_event_count+1，而timer线程检查完是否阻塞后会将其置为0  
> 如果此时,queue_event_count=0并且队列不为空，则说明所有的worker都在工作，group阻塞  

## 怎么工作
- 请求连接到mysql， threadid%thread_pool_size确定落在那个group
- group接收到请求后，检查队列是否为空，如果为空，则自己作为worker处理请求，否则将任务放入队列中
- worker检查队列请求，如果有请求的话，首先检查运行的线程数是否超过thread_pool_oversubscribe+1, 如果没有则执行，没有拿到任务的线程会在thread_pool_idle_timeout之后被回收
- timer线程会检查各个group是否存在阻塞出现，一旦出现阻塞，则会创建新的线程去处理请求

## 更多细节
- 一个连接可以被一个group中的一个或多个worker线程处理的
- listener线程不是固定的
- listener线程是通过epoll的方式来监听group中的连接事件的。

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181101115827163_935281056.png)
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181101115848289_1835895799.png)

# 连接推荐
MYSQL线程池总结 https://www.cnblogs.com/cchust/p/4510039.html  
MySQL线程池问题个人整理 https://cloud.tencent.com/developer/article/1068832  
Mysql线程池 淘宝月报 http://mysql.taobao.org/monthly/2016/02/09/
