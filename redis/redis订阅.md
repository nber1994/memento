--- 
title: redis订阅 
date: 2019-08-09
categories: 
- redis 
---
- redis发布订阅相关的命令
    - publish，发布
    - subscribe，订阅
    - psubscribe，模式订阅
# 频道订阅和退订
```c
struct redisServer {
    //保存所有频道的订阅关系
    dict *publish_channels;
}
```
- 频道的订阅信息由publish_channels保存
    - 字典的键为频道名
    - 字典的值为一个客户端的指针的链表
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121172343737_94803585.png)
## 订阅频道
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121172510005_2078470668.png)
## 退订频道
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121172552580_1887514974.png)

# 模式的订阅和退订
- 所有模式订阅信息都保存在redsiServer中的pubsub_patterns属性里
```c
struct redisServer {
    //保存所有模式订阅关系
    list *pubsub_patterns;
}
```
- pubsub_patterns是一个链表，每个链表的节点结构为
```c
struct pubsubPatterns {
    //订阅模式的客户端
    redisClient *client;
    //被订阅的模式
    robj *patterns;
}
```
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121173009519_835734534.png)

# 发送消息
- publish channel message
    - 首先将消息发送给channel的订阅者
    - 表里pubsub_patterns列表，符合条件的客户端发送消息
# 重点回顾
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121173832939_1647261558.png)
