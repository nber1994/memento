--- 
title: redis数据结构-跳跃表 
date: 2019-01-08
categories: 
- redis 
---
- 跳跃表通过每个节点位置多个指向其他节点的指针来实现快速访问
- 跳跃表可以实现最好O(logN)，最坏O(N)的节点查找，同时可以顺序操作批量处理
- redis的应用，一个是在有序集合中使用了跳跃表，同时在集群节点中作为内部数据结构使用

# 结构
跳跃表由zskiplistNode和zskiplist结构组成
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181116215118081_834546591.png)
## 对于跳跃表结构:
```c
typedef strut zskiplist {
    //表头节点和表尾节点
    struct zskiplistNode *header, *tail;;
    //表节点的数量
    unsigned long length；
    //表中层数最大值
    int level
} zskilplist;
```
- header: 指向表头节点的指针
- level：跳跃表中level最大的level值，不包括表头结点
- length：跳跃表中包含节点的个数，不包含表头

## 对于跳跃表节点结构：
```c
typedef struct zskiplistNode {
    //层
    struct zskiplistLevel {
        //前进指针
        struct zskiplistNode *forward;
        //跨度
        unsigned int span;
    } level[];
    //后退指针
    struct zskiplistNode *backward;
    //分值
    double score;
    //成员对象
    robj *obj;
} zskiplistNode;
```
- level：每个节点在初始化时，会分配给其一个1-32之间的level，每个level包含一个前进指针和跨度，前进指针用于访问队尾其他的节点，而跨度记录了本节点到前进指针所指节点的距离；一般来说层数越多，访问其他节点的速度越快
    - 对于跨度，其实是用来计算某个节点在列表中的排位的
- bw：后退指针，后退指针用于访问该节点的前一个节点。
- score：分值，保存各个节点的分值。跳跃表中的节点是按照分值从小到大排列的。score是一个double型
- obj：保存成员对象， 成员对象指向一个字符串对象，其保存一个SDS
- 注意：头节点同样存在后退指针，分值和成员对象，但是一般不使用
- 注意：当节点的score值相同时，但是成员变量不能重复，会按照成员对象在字典中的大小进行排序

# 重点回顾
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181117154357647_1008856380.png)
