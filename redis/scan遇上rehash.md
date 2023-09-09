--- 
title: scan遇上rehash 
date: 2020-03-27
categories: 
- redis 
---

# 缘起
面试时被问到，redis如果你想要找到所有test_开头的键值时，一般使用什么命令
我当时想也不想，keys命令啊
面试官一脸失望的问我，有没有用过scan命令
我赶紧说用过用过
但是当时心里特别害怕，生怕问这两者的区别，不过好在没有继续往下问，嘿嘿
不过面试后，我隐隐好奇，这两者到底有什么区别，为什么都推荐用scan这个命令？

# 背景
## keys和scan的区别
```
KEYS parttern：查找所有符合给定模式pattern的key。
```
1. KEYS指令一次性返回所有匹配的key。
2. key的数量过大会使服务卡顿。

```
SCAN cursor [MATCH pattern] [COUNT count]：查找给定数量内个数的符合给定模式pattern的key。
```
1. 基于游标的迭代器，需要基于上一次的游标延续之前的迭代过程。
2. 以0作为游标开始一次新的迭代，直到命令返回游标0完成一次遍历。
3. 不保证每次执行都返回给定数量的元素，支持模糊查询。
4. 一次返回的数量不可控，只能是大概率符合COUNT参数。
5. 返回的游标不一定是递增的，可能会获取到重复key，需要在外部程序去重。

## keys会导致的问题
首先我们看下redis的网络模型IO多路复用

## IO多路复用
简单来说，IO多路复用是一场一对多的群架，相较于多进程与多线程网络模型，io多路复用使用一个进程（一个线程）遍历处理多个socket的请求，准确来说，是交由操作系统来进行socket的遍历操作
![20190528115137282_1712659801](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20200401224925413_878421241.png)
总的来说分为三步：
1. fd拷贝（用户空间 -> 系统空间）
2. 遍历fd
3. 返回就绪的fd
4. 进程处理就绪fd请求




首先我们知道，redis是一个基于内存的单进程单线程（基于IO多路复用）的kv数据库
在redis2.8之前，能满足这一需求的是keys命令，但是他会导致两个问题：

1. 没有limit，只能一次性的获取所有符合条件的结果，换句话说如果你的命令输入错误，可能会有成百上千的输出
2. 单进程单线程的redis，一个O(N)的算法，如果执行时间很长，很可能会直接导致redis服务的阻塞

这谁顶得住啊，我如果需要在线上执行一个查询命令，即使命令没有输错，但是如果结果集很大，都有可能导致redis的一个节点阻塞

# 问题
## 为什么实现不了limit （时间复杂度）
那么我就会问，为什么不能支持limit操作呢，这样即使我的操作是O(N)的，我可以通过把limit设置的比较小，循环多次取，这样可以减少对服务的影响。但是为什么实现不了呢，这与redis的底层结构有关，我们知道redis的键值对底层是以dict这一内部数据结构组织的，那我们就看看dict这一结构为什么不好实现这一功能
## dict结构
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181115173716041_1984082947.png)
```c
dict结构
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
> 可以看到，一个dict存在两个hash表结构（本质是数组）

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

```c
hash节点结构
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
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181115172834333_457855334.png)
> 这里可以看到，redis使用了拉链法来处理hash冲突

我们可以看到上面的结构，每个dict都会存在两张hash表。由于存在哈希冲突，另外一张表的作用是在装载因子（used/size）过大或过小时，进行rehash时使用的。
首先，rehash会对hash表的size进行调整（增大或者缩小，但都保持2^n）, 且这个过程是渐进式的.
也就是说，这个过程可能会持续很长一段时间，问题就出在这个地方。

## rehash导致的问题
如果我们的keys同学支持了limit，上一次游标如果是100.下次从100开始继续获取时，在这个间隔中dict可能正在进行rehash操作，101对应的键值对可能已经迁移到了另一张hash表中，如果仍然按照现有的游标来进行便利，会发生什么呢
接下来我们就来复现一下这个问题：
首先我们知道，一般的hash表实现方式，

````go
//键值
string key = "USER_PHONE_123";
//根据键值计算hash值
hash_id = hashFunc(key);
//计算hash表掩码
mask = size(hash_table) - 1;
//获取hash值对应的索引值
idx = hash_id % mask;
````
可以看到，基本操作是hash值与掩码值取模，换算为二进制，即时hash_id |= ~mask
举个例子，如果table_size为8时，则所有的hash_id二进制形式的后三位则为其索引值。
如果按照正常的顺序进行迭代，哈希表容量分别为8和16时，idx的迭代过程如下：
```c
000 -> 001 -> 010 -> 011 -> 100 -> 101 -> 110 -> 111 -> 000
0000 -> 0001 -> 0010 -> 0011 -> 0100 -> 0101 -> 0110 -> 0111 -> 1000 -> 1001 -> 1010 -> 1011 -> 1100 -> 1101 -> 1110 -> 1111 -> 0000

```
### 哈希表的扩展
字典扩展情况下，当前字典哈希表的容量为8，假设在迭代完索引010的bucket之后，按理说下一个应该迭代011，
此时注意了，如果这个时候我去上了个厕所，在这期间，字典容量扩容成了16。而011这个索引，在扩容为16的情况下，变为了0011或者1011，
而之前已经迭代过得索引是
000，001，010
扩展到16之后，分别对应的是，
0000，1000
0001，1001
0010，1010
此时如果将游标移到0011和1011分别进行迭代的话，会依次迭代
0100，0101，0110，0111，1000，1001，1010，1011，1100，1101，1110，1111，0000
可以看到，其中0000，1000，1001，1010的bucket被重复迭代了
### 哈希表的收缩
字典收缩情况下，当前字典哈希表容量为16，假设在迭代完0100之后，我又去上了个厕所，这期间字典收缩为了8，下一个要迭代的bucket为0101，在容量为8时，下一个要迭代的bucket是101.
容量为16时，尚未迭代的索引为：
0101，0110，0111，1000，1001，1010，1011，1100，1101，1110，1111
而这些索引收缩后，分配到新的bucket中的索引为：
000，001，010，011，100，101，110，111
目前的索引为101，那么这个索引之前的
000，001，010，011，100
这些索引就不会迭代到了。因此某些节点被漏掉了。

这下我们就知道了，为什么keys同学没有办法支持limit了

那么，scan命令是如何保证在rehash过程中，迭代返回的数据不会遗漏呢？


# 解决
直到有一天，有个叫Pieter Noordhuis的男人，提出了一个叫做reverse binary iteration的算法，简单来说就是反转二进制迭代。
这个算法，其实作者本人也没有明确的证明出这个算法的正确性。

> antirez: Hello @pietern! I'm starting to re-evaluate the idea of an iterator for Redis, and the first item in this task is definitely to understand better your pull request and implementation. I don't understand exactly the implementation with the reversed bits counter...
I wonder if there is a way to make that more intuitive... so investing some more time into this, and if I fail I'll just merge your code trying to augment it with more comments...
Hard to explain but awesome.
pietern： Although I don't have a formal proof for these guarantees, I'm reasonably confident they hold. I worked through every hash table state (stable, grow, shrink) and it appears to work everywhere by means of the reverse binary iteration (for lack of a better word).


具体可以参考[Add SCAN command #579](https://github.com/antirez/redis/pull/579)
这一长篇的讨论，详细阐述了这个反转二进制迭代的算法：

## 反转二进制算法
首先可以看下scan命令的主要源码：
```c
unsigned long dictScan(dict *d,
                       unsigned long v,
                       dictScanFunction *fn,
                       void *privdata)
{
    dictht *t0, *t1;
    const dictEntry *de;
    unsigned long m0, m1;

    if (dictSize(d) == 0) return 0;

    if (!dictIsRehashing(d)) {
        t0 = &(d->ht[0]);
        m0 = t0->sizemask;

        de = t0->table[v & m0];
        while (de) {
            fn(privdata, de);
            de = de->next;
        }

    } else {
        t0 = &d->ht[0];
        t1 = &d->ht[1];

        if (t0->size > t1->size) {
            t0 = &d->ht[1];
            t1 = &d->ht[0];
        }

        m0 = t0->sizemask;
        m1 = t1->sizemask;

        de = t0->table[v & m0];
        while (de) {
            fn(privdata, de);
            de = de->next;
        }

        do {
            de = t1->table[v & m1];
            while (de) {
                fn(privdata, de);
                de = de->next;
            }

            v = (((v | m0) + 1) & ~m0) | (v & m0);  // 就是对 v 的低 m1-m0 位加 1，并保留 v 的低 m0 位

        } while (v & (m0 ^ m1));  // 循环条件 v &(m0 ^ m1)，表示直到 v 的低 m1-m0 位到低 m1 位之间全部为 0 为止。
    }

    v |= ~m0;

    v = rev(v);
    v++;
    v = rev(v);

    return v;
}
```
可以把源码中的主要逻辑拆出来，进行一个简单的测试，先观察下其主要行为
```c
#include <stdio.h>

// 对 v 进行二进制逆序操作
unsigned long rev(unsigned long v) {
    unsigned long s = 8 * sizeof(v);
    unsigned long mask = ~0;
    while ((s >>= 1) > 0) {
        mask ^= (mask << s);
        v = ((v >> s) & mask) | ((v << s) & ~mask);
    }
    return v;
}

//打印二进制
void printBits(unsigned long v, int tablesize)
{
    int cnt = 0;
    while(tablesize >> ++cnt);
    for(int i = cnt-2; i >= 0; --i)
        printf("%lu", (v >> i) & 1);
}

void test_dictScan_cursor(int tablesize)
{
    unsigned long v;
    unsigned long m0;

    v = 0;
    m0 = tablesize-1;
    printBits(v, tablesize);

    do
    {
        printf(" -> ");
        //保留游标v的低m0位，其余位全为1
        v |= ~m0;
        //二进制进行翻转
        v = rev(v);
        //对二进制进行加1操作
        v++;
        //再次翻转
        v = rev(v);
        printBits(v, tablesize);
    }while (v != 0);
    printf("\n");
}


int main()
{
    test_dictScan_cursor(8);
    test_dictScan_cursor(16);
    return 0;
}
```
打印出迭代过程为：
```c
$ gcc main.c -o main
$ ./main
000 -> 100 -> 010 -> 110 -> 001 -> 101 -> 011 -> 111 -> 000
0000 -> 1000 -> 0100 -> 1100 -> 0010 -> 1010 -> 0110 -> 1110 -> 0001 -> 1001 -> 0101 -> 1101 -> 0011 -> 1011 -> 0111 -> 1111 -> 0000
```
可以看到，reverse binary iteration算法的主要思想是，每次想游标的最高位加1，并向地位方向进位。
比如0010的下一个数是1001，1101的下一位是0011。
为什么要采用这种操作呢？我们还是看下采用这种迭代方法过程中，dict进行rehash的情况

## 哈希表的扩展
当hash表的容量为8时，进行迭代操作，当迭代了010游标之后，我起身去上了个厕所，这个时候dict进行了rehash，且容量扩展到了16。正常来说，下一个要迭代的索引是011，该索引扩展到新的哈希表中对应的索引为0011，1011，且这两个索引是相邻的。
容量为8时，迭代过的索引为
000，100，010
扩展容量为16时，之前迭代过的索引会转移到以下索引的bucket中：
0000，1000，0100，1100，0010，1010
而这些索引，都是在0011和1011之前，换句话说，按照现在得到的索引向下迭代时，不会有遗漏的bucket

## 哈希表的收缩
当hash表的容量为16时，我们同样进行迭代操作，当迭代了0100索引之后，我起身上了个厕所，这个期间dict进行了rehash，且容量收缩为了8，下一个要迭代的索引为1100，而容量为8时，下一个要迭代的索引为100.
容量为16时，已经迭代过的索引为
0000，1000，0100
对应容量为8的hash表，其索引为
000，000，100
且这些索引都在100这个索引之前，换句话说，按照现在得到的索引向下迭代时，也不会漏掉bucket。

# 总结
目前来看，scan命令可以保证返回结果的完整性，但是少数情况下会存在元素的重复，这一点可以由应用层处理

> 本文源码基于redis3.0

至此，我们了解到了这种reverse binary iteration算法的原理，撒花~


两者区别
rehash时机和频率 惰性rehash
