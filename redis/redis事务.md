--- 
title: redis事务 
date: 2019-01-08
categories: 
- redis 
---
- redis通过multi，exec，watch等命令实现事务功能
- 事务提供了一种将多个命令打包，一次性顺序执行的多个命令的机制
## 事务实现
- 事务开始
- 命令入队
- 事务执行
### 事务开始
- multi命令可以将执行该命令的客户端从非事务状态切换为事务状态
    - 将客户端的flag属性变为redis_multi
### 命令入队
- 当处于非事务状态下，客户端发送的命令会立即执行
- 当处于事务状态下，
    - 客户端发送的exec，discard，watch，multi命令会立即执行
    - 除了以上命令，服务器并不执行，而是放入一个事务队列中，返回quneued
#### 事务队列
- 每个客户端都有一个事务状态，
```c
typedef struct redisClient {
    //事务状态
    multiState mstate;
} redisClient;
```
- 每个事务状态包含一个事务队列，一个已入队命令计数器
```c
typedef struct multiState {
    //事务队列，FIFO顺序
    multiCmd *commands;
    //已入队命令计数
    int count;
}
```
事务对列是一个multiCmd的数组
```c
typedef struct multiCmd {
    //参数
    robj **argv;
    //参数数量
    int argc;
    //命令指针
    struct redisCommand *cmd;
}
```
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121175926946_165828182.png)

### 执行事务
- 当服务器收到客户端传来的exec命令时，会立即执行事务队列中的命令
- 并将执行结果返回给客户端
## watch命令
- watch是一个乐观锁，可以在exec执行之前，监视任意数量的数据库键
- 当exec执行时，如果监视的键被修改了，则事务就会拒绝提交
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121181014999_1787964715.png)
### watch命令监听数据库键
```c
typedef struct redisDb {
    //正在被watch命令监听的键
    dict *watched_keys;
} RedisDB;
```
- watched_keys的键为键名，
- 而值则是一个链表，每个链表代表一个监听该键的客户端
### 监听的触发
- 对数据库的修改命令，set lpush sadd等，会检查watched_keys字典
- 如果有客户端正在监听键时，会遍历watched_keys[key]对应的client的REDIS_DIRTY_CAS标识打开，标识事务安全性已被破坏
- 当事务exec时，会根据REDIS_DIRTY_CAS是否打开，来决定事务是否能提交
## 事务的ACID
- 原子性 ok
    - redis的事务不支持回滚，有一条命令出错后，其他的命令继续执行
- 一致性
    - 事务提交之后，不包含非法或者无效的错误数据
    - redis通过错误检查和简单的设计来避免一致性
        - 入队错误：命令入队时，命令不存在或者不正确，redis拒绝执行事务
        - 执行错误：执行中发生了错误，服务器也不会终端事务，其他的命令并不会受错误命令的影响
- 隔离性 ok 单线程
- 持久性 
    - 无持久化：不具有持久性
    - RDB持久化：不具有耐久性
    - AOF持久化，appendsync为always时具有
## 重点回顾
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121183305508_584292534.png)
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121183318766_1909384854.png)
