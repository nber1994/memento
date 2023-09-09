---
title: mysql锁表for update
date: 2017-10-20
categories: 
- mysql
---
> 胡乱探讨了for update的语句应用与使用场景

# 锁表 for update
 select for update 是为了在查询时,避免其他用户以该表进行插入,修改或删除等操作,造成表的不一致性.

## 一些例子

```sql

select * from t for update 会等待行锁释放之后，返回查询结果。
select * from t for update nowait 不等待行锁释放，提示锁冲突，不返回结果
select * from t for update wait 5 等待5秒，若行锁仍未释放，则提示锁冲突，不返回结果
select * from t for update skip locked 查询返回查询结果，但忽略有行锁的记录

```

## 语法

```sql
SELECT...FOR UPDATE 语句的语法如下：
　　SELECT ... FOR UPDATE [OF column_list][WAIT n|NOWAIT][SKIP LOCKED];
其中：
　　OF 子句用于指定即将更新的列，即锁定行上的特定列。
　　WAIT 子句指定等待其他用户释放锁的秒数，防止无限期的等待。
```

## 优点

**使用FOR UPDATE WAIT”子句的优点如下：** 
* 防止无限期地等待被锁定的行； 
* 允许应用程序中对锁的等待时间进行更多的控制。
* 对于交互式应用程序非常有用，因为这些用户不能等待不确定 
* 若使用了skip locked，则可以越过锁定的行，不会报告由wait n 引发的‘资源忙’异常报告

> 当我们进行for update的操作时，与普通select存在很大不同。一般select是不需要考虑数据是否被锁定，最多根据多版本一致读的特性读取之前的版本。加入for update之后，Oracle就要求启动一个新事务，尝试对数据进行加锁。如果当前已经被加锁，默认的行为必然是block等待。使用nowait子句的作用就是避免进行等待，当发现请求加锁资源被锁定未释放的时候，直接报错返回。
在日常中，我们对for update的使用还是比较普遍的，特别是在如pl/sql developer中手工修改数据。此时只是觉得方便，而对for update真正的含义缺乏理解。
For update是Oracle提供的手工提高锁级别和范围的特例语句。Oracle的锁机制是目前各类型数据库锁机制中比较优秀的。所以，Oracle认为一般不需要用户和应用直接进行锁的控制和提升。甚至认为死锁这类锁相关问题的出现场景，大都与手工提升锁有关。所以，Oracle并不推荐使用for update作为日常开发使用。而且，在平时开发和运维中，使用了for update却忘记提交，会引起很多锁表故障。

## 使用场景

就是那些需要业务层面数据独占时，可以考虑使用for update。场景上，比如火车票订票，在屏幕上显示邮票，而真正进行出票时，需要重新确定一下这个数据没有被其他客户端修改。所以，在这个确认过程中，可以使用for update。这是统一的解决方案方案问题，需要前期有所准备。
