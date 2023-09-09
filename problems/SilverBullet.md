--- 
title: SilverBullet 
date: 2020-07-09
categories: 
- else 
tags:
- summary
---
# mysql
1. 讲一讲myisam和innodb的区别
    1. 锁
    2. 事务
    3. 。。。
2. 讲讲innodb的索引
> [索引结构](http://nber1994.com/mysql/2019/07/05/mysql-索引结构.html)
[索引](http://nber1994.com/mysql/2019/01/08/mysql-索引.html)
3. 为什么innodb的索引是用B+树实现
    1. 平衡二叉树
    2. 叶子节点相连，范围
    3. 。。。
4. B+树的什么性质影响着查询性能
    1. 层高
5. innodb的辅助索引的叶子节点存储的是什么
6. 为什么这么存储，有什么优劣
    1. 主键索引变动对索引的影响小
    2. 但是查询速度会编码，引入自适应hash索引
7. 索引的最左前缀匹配原则
8. abc联合索引，说明什么查询条件能命中索引
> [事务](http://nber1994.com/mysql/2019/01/08/mysql-事务.html)
9. 讲一讲innodb的事务
10. 事务隔离级别，分别会出现什么问题
    1. 脏读
    2. 幻读
    3. 不可重复读
11. innodb默认是哪种隔离级别，如何解决上述问题
    1. mvcc
    2. next-key锁
12. innodb如何解决幻读，如何解决不可重复读
13. 幻读和不可重复读分别对应什么场景
    1. 幻读insert
    2. 不可重复读update
14. 快照度和当前度，innodb中，事务中的读是快照读，怎么强制读到当前的数据
    1. sql+for update强制读当前
15. innodb中的锁你了解吗
    1. 行
    2. 意向
    3. gap
    4. next-key
16. innodb支持哪种锁
17. 为什么会引入next-key锁，为了解决什么问题
    1. 幻读
18. innodb的持久化你知道吗
19. redo log和undo log分别有什么作用
    1. redo log持久化
    2. undo log快照度，事务回滚
20. innodb的double-buffer-wirte你知道吗，为什么要引入这个机制
21. 你是怎么优化sql的
> [sql军规](http://nber1994.com/mysql/2019/01/08/mysql-查询优化器&sql军规.html)

# redis
1. redis为什么这么快
    1. 基于内存
    2. IO多路复用
    3. 数据结构精简
    4. 。。。
2. redis的缺点
    1. 不能发挥多核多cpu的优点
    2. 事务支持较差
    3. 。。。
3. redis和memecache的对比
4. redis和local cache的对比
5. redis的zset如何实现，他获取单个节点的分值的时间复杂度是多少
    1. 引入hash达到O（1）
> [redis数据类型](http://nber1994.com/redis/2019/01/08/redis数据结构-对象.html)
6. redis的网络模型
7. redis为什么使用IO多路复用
    1. 代码简单，易于维护
    2. 基于内存，速度已经可以得到保障
8. redis常用场景
> [redis使用场景](http://nber1994.com/redis/2019/06/01/redis-应用场景.html)
9. redis 集群了解过吗
    1. 一致性hash
    2. 槽，预分片
10. redis 数据分片方案
11. redis 主从复制
12. redis的持久化机制
> [redis RDB](http://nber1994.com/redis/2019/01/08/redis持久化-RDB.html)
13. redis的bgsave实现
    1. fork
    2. copy on write
14. redis的aof和rdb的区别
15. redis的宕机恢复
16. redis键过期机制
    1. 惰性回收
    2. 定期回收
17. redis的pub和sub

# 网络
1. tcp和udp的区别以及适用的场景
2. tcp的三次握手，四次挥手
3. tcp如何保证可靠性
4. tcp的拥塞控制
5. tcp的慢启动
6. IOS七层模型，分别的作用和协议

# 操作系统
1. 进程和线程的区别
2. 进程fork子进程时，发生了什么操作
3. copy on write机制
4. 讲一讲虚拟内存
> [虚拟内存](http://nber1994.com/os/2019/05/18/os-虚拟内存.html)


# 系统设计

1. 布隆过滤器
> [布隆过滤器](http://nber1994.com/sysdesign/2019/01/10/sysDesign-布隆过滤器.html)
2. 发号器
> [发号器](http://nber1994.com/sysdesign/2019/01/09/sysDesign-发号器.html)
3. 热门微博统计
4. 秒杀系统
* 参照极客时间秒杀
5. 缓存问题
> [缓存问题](http://nber1994.com/sysdesign/2019/01/10/sysDesign-缓存的常见问题.html)
6. 分布式事务
> [分布式事务](http://nber1994.com/sysdesign/2019/05/31/sysDesign-分布式事务.html)
7. 分布式锁
> [分布式锁](https://zhuanlan.zhihu.com/p/41114567)

# 算法
1. 剑指offer
> [剑指offer](https://blog.csdn.net/c406495762/article/details/79247243)
2. leetcode 链表 数组 二叉树 图 dp

> 祝君好运！
