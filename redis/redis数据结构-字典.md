--- 
title: redis数据结构-字典 
date: 2019-01-08
categories: 
- redis 
---
- 字典是一个用于保存键值对的数据结构
- 字典的键为独一无二的
- redis的数据库就是使用字典实现的
- hash键的底层实现
# hash表结构
```c
哈希表结构
typedef struct dictht {
    //哈希表数组
    dictEntry **table;
    //哈希表大小
    unsigned long size；
    //哈希表大小掩码，用于计算索引值，总之等于size-1
    unsigned long sizemask;
    //该哈希表已有节点的数量
    unsigned long used;
}
```
table是一个数组，数组中的每个元素都是指向dictEntry结构的指针，每个dictEntry保存这一个键值对    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181115172104868_1544804852.png)

# hash节点
```c
typedef struct dictEntry {
        //键
        void *key;
        //值
        union {
            void *val;
            uint64_tu64;
            int64_ts64;
        }v;
        //指向下一个hash节点，形成链表
        struct dictEntry *next;
} dictEntry;
```
**其中next可以连接相同的节点，可以通过这个指针将相同哈希值的多个键值对节点连接起来，来解决哈希冲突的问题**    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181115172834333_457855334.png)

# 字典结构
字典由如下结构表示：    
```c
typedef struct dict {
    //类型特定函数
    dictType *type;
    //私有数据
    void *privData;
    //哈希表
    dictht ht[2];
    //rehash索引
    //当rehash不进行时，值为-1
    int trehashidx;
}
```
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181115173314833_740691337.png)

- ht属性是一个包含两个项的数组，一般情况下，使用ht[0],ht[1]只用来进行rehash操作
- trehashidx记录了rehash的进度
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181115173716041_1984082947.png)

# hash算法
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181116211409933_732195149.png)
hash算法根据**键值**，计算出哈希值，在结合sizemask的值，计算出索引值    
向hash表添加一个键值对时，先根据hash算法计算出键值对的哈希值和索引值    
再根据索引值确定新的哈希表节点放到指定索引上

# 解决键冲突
- 当两个以上的键被分配了相同的索引值，即为哈希碰撞
- redis使用链地址法来处理hash冲突，每个hash表项都有一个next指针，相同的索引值的会组成一个单项链表    
- 为了提高速率，每次发生hash冲突时，会将新的节点放在单向链表的表头    
 ![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181116211119221_356019613.png)

# rehash
当hash表的装载因子大于1时，redis会将hash表进行rehash操作
## 步骤：
- 首先会初始化ht[1]，表的大小由现有键值对的大小等因素决定
- 将ht0表中的所有的键值对重新计算hash值和索引值，写入ht1
- 当完成后，将释放ht0，将ht1作为ht0，并初始化一个空表尾ht1

## rehash的条件
- 不存在bgsave或者bgrewriteaof任务时，当装载因子大于1时
- 存在bgsave和bgrewriteaof任务时，当装载因子大于5时
    - 之所以根据bgsave和bgrewriteaof任务是否存在，来确定不同负载因子的值，是因为，bgsave和bgrewriteaof会产生子进程处理    
    - 由于大多操作系统采用写时复制机制来优化子进程效率，所以在bgsave和bgrewriteaof任务时提高负载因子会避免不必要的内存写入操作

## 渐进式的rehash
当存在大量的键值对时，rehash会带来大量的运算，并且在此期间不能响应请求，所以采用渐进式的rehash方法
- 当redis对键值进行增，删改查时，redis除了执行制定操作时，还会将对应的hashindex上的键值对全部rehash到ht1上
- 对rehash的值会反映在rehashidx上，当该值为-1时，表示已经完成rehash
- 在此期间，对于创建键值操作，会直接在ht1表上创建；查找时，ht0中不存在时，会再查找ht1中

# 重点回顾
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181116214218903_922571933.png)
