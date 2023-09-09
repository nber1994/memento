--- 
title: InnoDB锁 
date: 2019-01-08
categories: 
- mysql 
---
锁用于控制对共享资源的并发访问
对于innodb来说，支持行锁
对于myisam，支持表锁
表锁一般ddl时会用到

# 锁概述
lock的对象是事务，用来锁定数据库的对象，表，页，行。一般是在事务的commit和rollback之后释放
innodb中锁有两种类型

- s锁（共享锁），允许事务读取一行数据
- x锁（排他锁），允许事务删除或更新一行数据
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108014110859_1371949708.png)

## 意向锁
> InnoDB支持多粒度的锁定，这种锁定允许事务在行级锁上和表级锁上同时存在
为了支持这种操作，innodb支持一种额外的枷锁方式，成为意向锁    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108014515750_826824517.png)

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108014806106_72977482.png)

对于最细粒度进行上锁，首先需要对粗粒度的对象进行上锁，如果需要对行r上X锁，则需要对数据库A，表，页上意向所IX，然后再对行r上X锁    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108015204637_319377537.png)

对于innodb来说，意向锁设计的比较简单，意向锁即是表锁。设计目的是为了在一个事务中揭示下一行将要被请求的锁类型

- 意向共享锁（IS lock）事务想要获得一张表中几行的共享锁
- 意向排它锁（IX lock）事务想要获得一张表中几行的排他锁
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108015714483_1216630506.png)
可以看到，意向锁和s和x锁都是兼容的，而只有s和IS锁之间兼容    
**IS和IX都是表级锁，对于行级锁的X和S锁并不会冲突，只会和表级的S和X锁冲突**
**而对于行级锁的S和X锁，还是按照基本的原则**

## 意向锁的作用
1. 如果不存在意向锁，加锁时需要便利所有行来判断是否会有冲突
2. 存在意向锁时，只需要判断意向锁与枷锁的兼容性即可。因为存在意向锁代表有行级锁的存在

## 一致性非锁定读
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108020918717_1717111255.png)

多版本控制技术MVCC，innodb是基于undo log来实现的    
对于read commited隔离级别，每行数据读的是最新版本的数据    
而对于repetable read隔离级别，没行数据读的都是事务开始时数据版本数据    
对于read commited隔离级别来说，违反了一致性准则

## 一致性锁定度
两种一致性锁定度的语句

- select * from A ... for update 加X锁
- select * from A ,,,     lock in share mode 加S锁
对于innodb，在没有引入next-key锁之前，即使加上以上语句，即使读取的行加了for update，其他的事务特使可以读取的    

## 锁的算法
行锁的三种算法：

- ## record lock 单个记录的锁
- ## gap lock 间隙所，一个范围上的锁，但是不包括记录本身
- ## next-key lock gap lock+record lock锁，锁定一个范围，而且包括记录本身
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108143938266_1391350083.png)

**next-key lock是为了解决幻读的问题**
## next-key 行为分析
1. 当索引是唯一索引的情况下，next-key会被降级为record lock
例如表中存在1，2，5为主键，当查询5时，会把next-key锁降级为record lock，而不是锁住（2,5]
2. 如果是辅助索引，情况则不一样
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108150455217_471412798.png)

对于这个查询，由于b=3对于的主键索引来说是5，所以对于聚簇索引来说，只会在a=5加上record lock，但是对于辅助索引来说    
会添加(1,3]的next-key lock， 需要特别注意的是，innodb还是会对（3，6）加上gap锁    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108151713682_967722203.png)
此时，第一条会被阻塞，第二条会被阻塞，第三条不会被阻塞
从上面的例子可以看到，gap lock是为了阻止多个记录在同一个范围之内插入数据，从而引起幻读的问题    
**特别提醒的是，next-key lock降级为record lock仅仅存在于所有唯一索引列，如果是唯一索引是联合索引，则不会降级**
**幻读：指在同一事务下，连续两次执行同样的sql可能导致不同的结果，第二次可能返回之前不存在的行**

## 锁问题
## 脏读
脏读指的是读到了未提交的数据，这一行为违反了事务的隔离性
## 不可重复读
不可重复读，一个事务进行中，另一个事物页访问该数据且做了相应的修改，并提交了事务。    
这时第一个事务再去读数据时读到的数据是不一样的        
不可重复读和脏读的区别是不可重复读读到的是已经提交的数据，而脏读读到的是未提交的数据    
不可重复读违反了事物的一致性    
不可重复读也叫做幻读    
### 不可重复读解决
对于幻读，innodb的解决问题的方法是使用next-key lock锁，对于数据扫描时，会对范围加锁，在该范围内insert操作是不允许的    
innodb默认隔离级别是repetable read

## 丢失更新
丢失更新是一个锁导致的问题，但是innodb的行锁或者意向锁会解决这个问题    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181108171732786_1613407090.png)

## 阻塞
因为不同的锁的兼容性，某一时刻，一个资源被占用，另一个事务只能等待资源被释放后才能继续    
该问题并不是坏事，因为他会保证事务的正常运行

## 死锁
死锁是指两个或两个以上事务在执行过程中，因争夺资源而造成的互相等待的现象    
## 解决方法
最简单的解决方法就是超时
