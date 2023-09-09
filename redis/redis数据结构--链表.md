--- 
title: redis数据结构--链表 
date: 2019-08-09
categories: 
- redis 
---
- 链表提供了高效的节点重排的能力
- 顺序访问的能力
发布订阅，慢查询，监视器等都用到了链表
# 链表结构
```c
typedef struct listNode {
    //前置节点
    struct listNode *prev;
    //后置节点
    struct listNode *next;
    //节点的值
    void *value;
}
```
多个listNode可以通过prev和next链接为双向链表    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181115164514198_518123244.png)
```c
typdef struct list {
    //表头节点
    listNode *head;
    //表尾节点
    listNode *tail;
    //链表包含的节点数量
    unsigned long len;
    //节点值复制函数
    void *(*dup)(void *ptr);
    //节点值释放函数
    void *(*free)(void *ptr);
    //节点值对比函数
    void *(*match)(void *ptr， void *key);
}
```
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181115165456025_51173323.png)

# redis链表实现的特性：
- 双端：每个节点都存在prev和next指针，获取某个节点前后指针为O(1)
- 无环：头节点和尾节点的prev和next为null，对链表的访问以null终止
- 带有表头和表尾指针，自带表头和表尾指针，查找表头和表尾指针为O(1)
- 待链表长度计数器：获取链表长度为O(1)
- 多态：链表的值为*void，而且可以通过绑定不同的dup，free，match函数，可以保存不同类型的值
# 重点回顾
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181115170437776_1076580861.png)
