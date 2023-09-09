--- 
title: redis数据结构-对象 
date: 2019-08-09
categories: 
- redis 
---
- redis并没有直接使用底层的数据结构，而是基于这些数据结构创建了一个对象系统
- 该系统包含了字符串对象，列表，hash表，集合，有序集合五种数据对象
- 带来的收益：
    - 通过对象系统，可以在执行给出的命令来说，执行之前可以判断是否能执行该命令
    - 同时可以根据不同的使用场景，来选择数据类型的不同的实现方式
    - 通过引用计数，可以实现内存回收机制，当不再占用对象时，会对其回收。同时引用计数还可以实现对象共享，来节省内存
    - 对象还记录了访问时间，以此来计算数据库键的空转时长

# 对象的类型与编码
当创建一个键值对时，至少会创建两个对象，一个是键的字符串对象，另一个是值的对象
```c
typedef struct redisObject {
    //类型
    unsigned type:4;
    //编码
    unsigned encoding:4;
    //指向底层实现数据结构的指针
    void *ptr;
} robj;
```

## 类型
对象的type的值：
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181117184139552_1620012426.png)

## 编码和底层实现
encoding来决定ptr指向哪种底层数据结构    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181117184350906_1751167116.png)

不同的type对应的encoding    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181117184549158_602336477.png)

- 对于字符串对象，底层可以有三种实现方式，int，emstr和raw
- 对于列表对象，底层可以有链表，压缩表实现
- 对于hash表对象，底层可以有字典，压缩表实现
- 对于集合对象，底层可以有字典，整数集合实现
- 对于有序集合对象，底层可以有跳跃表，压缩表实现

不为某种对象绑定特定的数据结构，极大的提高了redis对象的灵活性

# 字符串对象
- 字符串对象可以有int，embstr和raw实现
- 当字符串的值可以用long类型标识时，字符串对象会使用int作为底层实现      
# ？？ 为什么
- 当字符串长度小于39字节时，采用embstr的方式来实现
- 当字符串长度大于39字节时，采用raw的方式来实现
- 为什么是39呢，因为redis采用的是jemalloc进行动态内存的申请，会申请8，16，32，64字节的内存，但是sds大小为33，而且redisObject为16字节，所以64-16-4-4-1 = 39
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181117221056567_1885714289.png)

## 采用embstr的优势
- embstr和raw的区别是，redisObject和SDS的内存地址是连续的，这样：
    - 初始化时只需要一次内存分配
    - 删除时也仅仅需要一次内存分配
    - 由于是连续的，读取的时候只需要一次寻址，速度较快
- embstr并没有实现操作函数，所以对embstr进行操作时，会先转化为raw再进行操作
# ？？那为什么不都存为embstr

## 编码的转化
int编码和embstr编码有时会转化为raw，比如append命令

# 列表对象
- 列表对象底层实现是有ziplist和linkedlist
- 压缩表实现时，表中的每一个节点时列表的一项
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181117233931320_636392797.png)
- linkedlist实现时，每个节点包含一个字符串对象
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181117233951025_1385177119.png)
- 字符串对象是唯一一个可以嵌套在其他对象之中的
- 当满足一定条件时，ziplist会转化为linkedlist
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181117234251486_1949105553.png)

# 哈希对象
- 哈希对象是使用ziplist和hashtable实现的
- 当使用压缩表实现时，会先将键值对的键压入压缩表的表尾部，然后将值压入表尾部
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118000138349_901477933.png)
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118000153568_642490346.png)
- 当使用hashtable实现时，字典的每个键都是一个字符串对象，字典的每个值也是一个字符串对象
![hash对象的dict实现，真实结构比这个复杂](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118125331822_34741081.png)
## 编码转化
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118125519764_1925280030.png)

# 集合对象
- 集合对象底层可以由hashtable和intset实现
- 当集合对象由intset实现时，集合的所有元素都保存在intset中
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118125936896_2045354861.png)
- 当集合由hashtable实现时，hashtable的键保存了集合对象的每个元素，对应的值为null
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118130126707_1508411298.png)

## 编码转换
intset只能保存一定数量的整数元素    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118130159940_824232651.png)

## 集合的交集，并集，差集操作
* 交集
    * 将集合按照大小从小到大排序
    * 遍历最小集合的每个元素，查找其他集合是否存在
    * O（n*m)
* 并集
    * 新建一个集合，其他的元素都添加进来即可
* 差集
    * 算法一：将第一个集合所有的元素在其他集合中(按照集合大小排序)寻找，都不存在的则放入结果集O(N*M)
    * 算法二：将第一个集合的所有元素放入一个中间集合，然后将所有的其他集合的元素，如果存在则从中间集合删除O(N)

# 有序集合对象
- 有序集合对象可以由ziplist和skiplist实现
- 使用ziplist实现
    - 当使用ziplist实现时，使用两个紧挨的压缩表节点来标识有序集合的一个元素，有序集合的元素值在第一个，分值在第二个
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118141740596_1608642207.png)
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118141759082_1777940966.png)

- 使用zskiplist实现时
```c
typedef struct zset {
    zskiplist *zsl;
    dict *dict;
} zset;
```
    - 当使用zskiplist实现时，跳跃表的每个节点对应一个有序集合的元素，节点的值存储有序集合的分值，节点的成员对象存储有序集合元素的值的字符串对象
    - 根据跳跃表可以实现有序集合的范围查找
    - 除此之外，有序集合还维护了一个字典结构，以集合元素作为字典的键，分值作为键值。来提供O(1)的根据对象查询分支的能力
    - 有序集合的每个元素对应一个字符串对象，而每个分值对应一个double类型的浮点数
    - zskiplist和dict通过指针来共享集合元素的成员和分值
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118142603900_1766351204.png)

## 编码的转换
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118142700281_250749500.png)

# 类型检查和命令多态
## 类型检查
- redis中存在两类命令
    - 一类是通用的命令，例如del，expire，type等命令
    - 二类是只有特定类型才能执行的命令
     ![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118143034423_983931086.png)
- redis在执行命令之前会根据数据类型判断是否可以执行该命令
- 否则会拒绝执行

## 命令多态
- redis会根据数据类型来判断命令之外，还会根据数据类型的encoding来选择正确的命令代码来执行
- del和expire命令可以实现类型的多态，而llen实现的是基于encoding的多态

# 内存回收
- c语言没有自动内存回收机制，所以redis使用引用计数的方式实现了内存的回收机制
- redisObject中refCount记录了引用的数目
    - 当创建时，refCount=1
    - 被一个程序使用时，refCount+1
    - 不再被一个程序使用时，refCount-1
    - 当refCount为0时，该对象会被回收

# 对象共享
- 引用计数除了实现内存回收机制之外，还实现了对象共享机制
- 键A引用了包含100的字符串对象，键B也声明了一个100的值
    - 将键B的值的指针指向键A的100对象
    - 将100对象的引用计数加一
    ![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118144649442_1305680502.png)
- redis在初始化时，会首先创建很多的字符串对象，其中包含了0到9999的整数值
- 共享对象不单单字符串键可以使用，那些包含字符串对象的对象也可以使用共享对象

# 对象的空转时长
- redisObject中还存在一个lru的属性，以此来记录该对象最后被引用的时间
- 当使用object idletime命令计算时，会以当前时间减去lru的值
# redisObject
```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    void *ptr;
    int refcount;
    unsigned lru:22;
} redisObject;
```

# 重点回顾
- 实现了五种对象：
- 字符串对象
    - int
    - embstr
    - raw
- 列表对象
    - ziplist
    - linkedlist
- 哈希对象
    - ziplist
    - dict
- 集合对象
    - intset
    - dict
- 有序集合对象
    - skiplist
    - ziplist
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118145543259_1584292068.png)
