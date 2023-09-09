--- 
title: redis数据结构-压缩列表 
date: 2019-01-08
categories: 
- redis 
---
- 压缩列表是列表建和哈希键的底层实现之一
- 压缩表是一系列有特殊编码组成的顺序型的数据结，可以保存字节数组和整数值

# 压缩表结构
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181117163711230_1673580428.png)
- zlbytes：记录整个压缩表的字节长度
- zltail：记录尾节点的距离列表起始地址有多少字节
- zllen：记录存在节点的数目
- entryX：压缩表包含的节点，长度由保存内容决定
- zlend：特殊值，标明压缩表的结尾

# 压缩表节点结构
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181117164343168_125444990.png)
- prev_entry_length 前一个节点的长度，prev_entry_length可以是1字节或者5字节
    - 当前一个节点的长度小于254字节时，prev_entry_length为一个字节
    - 当前一个节点的长度大于254字节时，prev_entry_length为5字节
    - 遍历：当想要遍历一个压缩表时，会根据zltail首先跳转到最后的节点，然后根据prev_entry_length节点向前遍历
- encoding 编码
    - encoding记录着节点content的长度和数据类型
- content 内容
    - content的值的类型由encoding来决定
# 连锁更新
- 如果存在这一一种情况的数组，每个节点的长度都在250到254之间时，当向前一个节点的插入一个大于254长度的节点，后一个节点会更新期prev_entry_length的长度，使该节点的长度也大于了254，这样将导致一系列的内存重分配操作
- 除了插入操作，删除操作也会导致连续更新的操作
- 在连续更新的情况下，最坏为O（N^2）
- 连续更新操作大量出现的概率比较小，所以压缩表的平均还是为O(N)

# 重点回顾
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181117170719184_1598835606.png)


？？压缩表优点
内存中以连续快存放，载入速度更快

