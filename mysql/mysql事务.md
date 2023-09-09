--- 
title: mysql事务 
date: 2019-08-09
categories: 
- mysql 
---
# 事务概述
> Innodb中的事务隔离级别和锁的关系 https://tech.meituan.com/innodb_lock.html

- A：原子性
一个事件中的几个步骤，要么不做，要么都做
- C：一致性
事务将数据库从一个状态转化为另一个一致性状态，例如外键约束没有遭到破坏
一致性关注的是数据，用户只对事务开始前的数据和事务结束后的数据可见，中间状态的数据对用户是不可见的
- I：隔离性
隔离性也叫做并发控制，可串行化，锁等。总的来说就是事务提交之前对其他事务是不可见的
隔离性是当多个用户并发访问数据库时，比如操作同一张表时，数据库为每一个用户开启的事务，不能被其他事务的操作所干扰，多个并发事务之间要相互隔离。
- D：持久性
事务一旦提交，则其结果就是永久性的
持久性是指一个事务一旦被提交了，那么对数据库中的数据的改变就是永久性的，即便是在数据库系统遇到故障的情况下也不会丢失提交事务的操作。


# 事务的实现
redo log 用来保证事务的持久性  
undo log用来保证事务的一致性  

redo log 和undo log并不是互为逆过程  
redo log记录的是页的改动  
undo log记录的是行的改动  

# redo log
分为两种，一种是redo缓冲区中的redo log，其是易失的    
另一种是redo日志文件，是持久的    
## redo log作用
innodb是事务型存储引擎，通过force log at commit机制实现事务的持久化，即在事务提交之前将所有日志写入到日志文件中    
redo log基本上是顺序写的，而且只有在redo log日志写入之后，调用fsync命令。所以磁盘的性能决定了sync的性能，进而决定了事务的性能  
在innodb中，该日志包含两部分，redo log和undo log，redo log来实现事务的持久性，undo log来实现事务的回滚和mvcc机制    


## binlog与redo log区别

- 层次不同 binlog是记录的mysql行为的，而redo log只是innodb中的
- 日志内容不同，binlog记录的是逻辑日志，记录的内容是sql，而redo log记录的是页面的变动
- 写入时机不同，binlog是在事务提交只有写入，redolog在事务中就写入

## redolog恢复
当mysql启动时，不论上次关闭是否正常，都会重做redolog，将页面改动做回磁盘，根据checkpoint表示的LSN，重做之后的redolog    

# undo log
undolog用于事物的回滚，将数据改动为事务之前的状态
与redolog不同的是，undolog是存在于共享表空间的undo段
## 误解
undolog是逻辑日志，只记录变化量
## undolog作业
一个是进行事务的回滚    
另一个是实现mvcc    
当一个事务读取一行信息，如果发现正在被某一事务占用，则会根据undolog读取到之前版本的信息    
同时undolog也会导致redolog的产生，因为undolog也需要持久化保护    

# 事务隔离级别
> 事务的四个隔离级别

- read uncommitted
- read commited
- repeatble read
- serializable
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108214404214_256950961.png)

innodb支持的隔离级别是repeatable read，但是innodb通过next-key lock解决了幻读的问题    
所以innodb已经完全实现了事务的ACID，实现了serializeble    

## read commited
所有的读取都是不加锁的，数据写入，修改删除是加锁的

## repeatable read
## 不可重复读&幻读
对于这两个概念，其实是有一定区别的，不可重复读是对于修改和删除来说的，而幻读是对于inser来说的。    
sql第一次读取某行之后，将改行记录加锁，阻止其他事务读取，即可实现可重复读，但是此时如果有insert操作，    
是无法阻止的，还是会出现幻读的现象，这种情况下只能使用悲观锁，读加上读锁，写加上写锁，读写互斥，这样可以避免    
脏读，不可重复读，幻读的问题，这种方法会导致并发性能降低，而innodb采用了mvcc机制避免了幻读的问题
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108215740807_2132671057.png)

## mvcc的实现
innodb会为每条记录增加一个版本信息，对应的是事务的版本号，每开启一个事务，事务的版本号就会递增    
在MVCC中，

- select 读取的是<=当前事务号的数据。
- insert 保存当前事务版本号为当前插入数据版本号
- delete 保存当前事务版本号为行的删除版本号
- update 新插入一条记录，保存当前事务版本号为行创建版本号，同时将原来的版本号保存到原来的行
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108220727145_152902922.png)

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108221515967_1448968242.png)

## 快照度和当前度
快照度读取的是历史的数据    
当前度读取的是当前最新版本的数据    

1. 在mvcc中，快照读就是

    - select * from table_t where ...
2. 当前读
    - select ？ from table_t for update
    - select ？ from table_t lock in share mode
    - insert
    - update
    - delete
**当前读其实是对于写操作来说的，当前读会存在幻读的问题，所以innodb引入了next-key锁来解决这个问题**
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108222755951_1379483439.png)

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108222812780_744706261.png)

> 在mysql中，此时其实会在（5，30】上加next-key锁，从而阻止了事务B的插入操作，另外innodb其实会在(30 ， -∞)  
上也加上gap锁，使其他事务无法在这两个区间插入操作，受限于这种方式，innodb会锁住不需要的区间    

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108223515915_743433911.png)

对于update操作，如果where没有命中索引，则会在全表上加上gap锁，此时所有的操作都会被阻塞    
行锁防止别的事务修改或删除，GAP锁防止别的事务新增，行锁和GAP锁结合形成的的Next-Key锁共同解决了RR级别在写数据时的幻读问题。
