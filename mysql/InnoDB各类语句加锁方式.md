---
title: InnoDB各类语句的加锁方式与应用
date: 2017-10-20
categories: 
- mysql
---
> 胡乱探讨了mysql的InnoDB的简单的加锁机制与使用场景，同时也有加锁等级

# 加锁机制


 锁定读、UPDATE、DELETE通常在处理SQL语句的过程中在扫描到的每个索引记录上加锁，不关心WHERE条件中可能排除行的非索引条件。比如，A表有两列i和j，i列有索引，j列没索引，当前存在(1,1)，（1,2），（1,3），（1,4），（2,1），（2,2），（2,3），（2,4）……等记录，语句SELECT * FROM A WHERE i=1 AND j=3;会在所有i=1的索引记录上加锁，而不考虑j=3这个条件。如果查询中使用了辅助索引，InnoDB除了给扫描到的辅助索引加锁，还会查找到对应的聚集索引并在其上加锁。若语句用不到合适的索引，则MySQL会扫描整个表，每个表行都会被加锁，会阻塞其他用户的插入操作。 

# innoDB对不同的语句加不同的锁
* SELECT...FROM读数据库快照，不对记录加锁，除非使用的是SERIALLIZABE隔离级别，此时对索引记录加S Next-key Lock。
* SELECT...FROM...IN SHARE MODE加S Next-key Lock。
* SELECT...FROM...FOR UPDATR /  UPDATE ... WHERE ...  / DELETE FROM ... WHERE ... /加X Next-key Lock。
* INSERT在插入的索引记录上加X锁，不会阻止其他事物在插入的记录前的“间隙”插入新的记录。插入记录前，会设置一把 insertion intention gap lock用以表明：不同的事务可以向同一索引“间隙”插入记录而无需相互等待，只要其插入的位置不同。一个事务中insert语句会在插入的行的索引记录上设置一把排它锁。如果有键重复的错误发生，则会在重复的索引记录上设置一把共享锁。在多个session同时插入同一行，且另外的某个session已经持有了该索引记录的排它锁时，共享锁的使用可能导致死锁的出现。

# 加锁方式
innoDB预设的事务隔离机制为REPEATABLE READ，在SELECT 的读取锁定主要分为两种方式:
* SELECT ... LOCK IN SHARE MODE
* SELECT ... FOR UPDATE
**这两种方式在事务(Transaction) 进行当中SELECT 到同一个数据表时，都必须等待其它事务数据被提交(Commit)后才会执行。而主要的不同在于LOCK IN SHARE MODE 在有一方事务要Update 同一个表单时很容易造成死锁 。**
简单的说，如果SELECT 后面若要UPDATE 同一个表单，最好使用SELECT ... UPDATE。

# 应用场景
**假设商品表单products 内有一个存放商品数量的quantity ，在订单成立之前必须先确定quantity 商品数量是否足够(quantity>0) ，然后才把数量更新为1。**

## 不安全的做法:

```sql
SELECT quantity FROM products WHERE id=3; 
UPDATE products SET quantity = 1 WHERE id=3;
```
**会出现的问题**
* 包并发的情况下一定会出现错误!
* 如果我们需要在quantity>0 的情况下才能扣库存，假设程序在第一行SELECT 读到的quantity 是2 ，看起来数字没有错，但是当MySQL 正准备要UPDATE 的时候，可能已经有人把库存扣成0 了，但是程序却浑然不知，将错就错的UPDATE 下去了
* 必须透过的事务机制来确保读取及提交的数据都是正确的。

## 解决方案

```sql
SET AUTOCOMMIT=0; BEGIN WORK; 
SELECT quantity FROM products WHERE id=3 FOR UPDATE;  
```
**此时products 数据中id=3 的数据被锁住(注3)，其它事务必须等待此次事务 提交后才能执行**
SELECT * FROM products WHERE id=3 FOR UPDATE 如此可以确保quantity 在别的事务读到的数字是正确的

```sql
UPDATE products SET quantity = '1' WHERE id=3 ; 
COMMIT WORK;
```
* 提交(Commit)写入数据库，products 解锁。
* BEGIN/COMMIT 为事务的起始及结束点，可使用二个以上的MySQL Command 视窗来交互观察锁定的状况
* 在事务进行当中，只有SELECT ... FOR UPDATE 或LOCK IN SHARE MODE 同一笔数据时会等待其它事务结束后才执行，一般SELECT ... 则不受此影响
* FOR UPDATE 只适用于InnoDB，且必须在事务中
* FOR UPDATE时正常查询不会阻塞，只有事物的查询会被阻塞，但是写表操作不论是否在事务里，都会阻塞

# Row Lock 与 Table Lock

**锁定(Lock)的数据是判别就得要注意一下了。由于InnoDB 预设是Row-Level Lock，所以只有「明确」的指定主键，MySQL 才会执行Row lock (只锁住被选取的数据) ，否则MySQL 将会执行Table Lock (将整个数据表单给锁住)**

## 例子
### 明确指名主键,并且有此数据,row lock

```sql
SELECT * FROM products WHERE id='3' FOR UPDATE;
```

### 明确指明主键，但无数据，无lock

```sql
SELECT * FROM products WHERE id='-1' FOR UPDATE;
```

### 无主键table lock

```sql
SELECT * FROM products WHERE name='Mouse' FOR UPDATE;
```

### 主键不明确，table lock

```sql
SELECT * FROM products WHERE id<>'3' FOR UPDATE;
SELECT * FROM products WHERE id LIKE '3' FOR UPDATE;
```

