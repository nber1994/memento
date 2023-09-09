--- 
title: mysql索引技巧 
date: 2019-01-08
categories: 
- mysql 
---
> ps：平衡查找树

> B B+ https://zhuanlan.zhihu.com/p/27700617

> MySQL索引背后的数据结构及算法原理 http://blog.codinglabs.org/articles/theory-of-mysql-index.html

> 联合索引结构原理 https://blog.csdn.net/weixin_30531261/article/details/79329722  

> 覆盖索引 https://yq.aliyun.com/articles/62419

> 联合索引如何选取wherer http://hedengcheng.com/?p=577


# 索引数据结构基础知识
## 平衡二叉树
平衡二叉树是基于二分法的策略提高数据查找速度的数据结构          
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181106172726705_789983747.png)

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181106172812442_1512930075.png)

## B-tree
与平衡二叉树的区别是B树属于多叉树，每个节点多于两个查找路径    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181106181118682_1672741123.png)
### 规则
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181106173002453_1609390045.png)
## B+树
B+树让查询速度更加稳定，完全趋近于二分查找  
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181106181138368_1051631384.png)
### 与B树的区别
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181106173251310_1461952350.png)
特点：飞叶子节点中不保存记录指针，每个节点可以存更多的关键字，数据只保存在叶子节点，每次查询的速度一定  
每个节点的出度增加，进而树高会降低，提高了查找速度。  
## 为什么使用B/B+树
一般来说，索引本身都很大，所以一般会存在磁盘上，而相对于内存读取，读取磁盘的开销是十分大的，所以衡量一个索引性能的一个标准  
就是一次查询IO的次数
## mysql索引实现
MyIsAM索引结构是通过B+树实现的，叶子节点存放的是数据的地址  
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181106182030630_978256078.png)
而对于辅助索引来说，其结构和主键索引没有任何区别  
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181106182342955_415101514.png)
> 这类索引成为非聚簇索引，即数据和索引文件是分开的
## InnoDB索引实现
> 对于InnoDB来说，数据本身就是主索引文件，

因为索引和数据是被保存在一起的，与MyIsAm不同的是，InnoDB的B+树的叶子节点  
包含完整的数据行记录，因而该索引称为聚簇索引  
因而InnoDB表必须有主键，如果没有的话，InnoDB会自动隐世的创建主键索引  
> 另外对于InnoDB引擎来说，辅助索引叶子节点存储的是主键的值而不是地址

因而InnoDB检索需要经过两边索引：首先通过辅助索引拿到主键的值，在检索一遍主键索引  
因而不建议将索引做的过大，因为会导致过大的索引文件  
不单调的字段作为索引会导致数据插入时频繁的分裂调整  

## 索引使用策略&优化
### 最左前缀原理&优化
```sql
SHOW INDEX FROM employees.titles;
+--------+------------+----------+--------------+-------------+-----------+-------------+------+------------+
| Table  | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Null | Index_type |
+--------+------------+----------+--------------+-------------+-----------+-------------+------+------------+
| titles |          0 | PRIMARY  |            1 | emp_no      | A         |        NULL |      | BTREE      |
| titles |          0 | PRIMARY  |            2 | title       | A         |        NULL |      | BTREE      |
| titles |          0 | PRIMARY  |            3 | from_date   | A         |      443308 |      | BTREE      |
+--------+------------+----------+--------------+-------------+-----------+-------------+------+------------+
```

> 情况一：全列匹配
```sql
EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001' AND title='Senior Engineer' AND from_date='1986-06-26';
+----+-------------+--------+-------+---------------+---------+---------+-------------------+------+-------+
| id | select_type | table  | type  | possible_keys | key     | key_len | ref               | rows | Extra |
+----+-------------+--------+-------+---------------+---------+---------+-------------------+------+-------+
|  1 | SIMPLE      | titles | const | PRIMARY       | PRIMARY | 59      | const,const,const |    1 |       |
+----+-------------+--------+-------+---------------+---------+---------+-------------------+------+-------+
```
where后的字段InnoDB会自动调整顺序
> 情况二：最左前缀匹配
```sql
EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001';
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------+
| id | select_type | table  | type | possible_keys | key     | key_len | ref   | rows | Extra |
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------+
|  1 | SIMPLE      | titles | ref  | PRIMARY       | PRIMARY | 4       | const |    1 |       |
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------+
```
这个时候可以看到key_len长度是4，只用了第一个字段

> 情况三：中间缺乏字段
```sql
EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001' AND from_date='1986-06-26';
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+
| id | select_type | table  | type | possible_keys | key     | key_len | ref   | rows | Extra       |
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+
|  1 | SIMPLE      | titles | ref  | PRIMARY       | PRIMARY | 4       | const |    1 | Using where |
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+
```
这个时候无法命中后续的索引字段，所以key_len使用了4

> 情况四：没有指定第一列
```sql
EXPLAIN SELECT * FROM employees.titles WHERE from_date='1986-06-26';
+----+-------------+--------+------+---------------+------+---------+------+--------+-------------+
| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+----+-------------+--------+------+---------------+------+---------+------+--------+-------------+
|  1 | SIMPLE      | titles | ALL  | NULL          | NULL | NULL    | NULL | 443308 | Using where |
+----+-------------+--------+------+---------------+------+---------+------+--------+-------------+
```
此时不会用到索引
> 情况五：匹配某列的前缀字符
```sql
EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001' AND title LIKE 'Senior%';
+----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table  | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | titles | range | PRIMARY       | PRIMARY | 56      | NULL |    1 | Using where |
+----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
```
如果通配符不是第一个字段的条件的话，可以用到索引

> 情况六：范围查询
```sql
EXPLAIN SELECT * FROM employees.titles WHERE emp_no < '10010' and title='Senior Engineer';
+----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table  | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | titles | range | PRIMARY       | PRIMARY | 4       | NULL |   16 | Using where |
+----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
```
最左前缀的字段可以使用索引，但是后面的字段就无法使用了。
**对于between来说，其实不属于范围操作**

> 情况七：查询条件中存在函数或者表达式
```sql
EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001' AND left(title, 6)='Senior';
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+
| id | select_type | table  | type | possible_keys | key     | key_len | ref   | rows | Extra       |
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+
|  1 | SIMPLE      | titles | ref  | PRIMARY       | PRIMARY | 4       | const |    1 | Using where |
+----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+
EXPLAIN SELECT * FROM employees.titles WHERE emp_no - 1='10000';
+----+-------------+--------+------+---------------+------+---------+------+--------+-------------+
| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+----+-------------+--------+------+---------------+------+---------+------+--------+-------------+
|  1 | SIMPLE      | titles | ALL  | NULL          | NULL | NULL    | NULL | 443308 | Using where |
+----+-------------+--------+------+---------------+------+---------+------+--------+-------------+
```
可以看到当第一个字段如果存在表达式时，则不会命中索引  

## 索引的选择
索引并不是越多越好，
- 索引本身会占用资源
- 索引会增加插入删除修改的额外开销

不建议建立索引的情况：
- 数据量较小
- 所选字段选择性比较低
- 字段选择性可以按照count(distince(x))/count(x)来衡量，该值越高选择性越好

## 逐渐选择
**永远建议使用自增字段作为主键**
因为基于B+树的索引结构，如果使用自增的主键，每次新插入一条数据，InnoDB会选择合适的位置建立索引，如果达到装载因子则会重新开辟页面。      
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181107002022004_1160332232.png)
如果使用非自增的主键，每次新插入的数据可以看为随机的，此时插入数据时，InnoDB寻找随机的位置建立索引，因此不得不为了插入新的数据而把后面的页进行重新的移动，增加了很多开销    

## 辅助索引

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181102170830932_2057287060.png)

对于辅助索引，大致的结构是这样的。
如果存在联合索引，则只会生成一个辅助索引树  
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181107002903925_404824817.png)
### 联合索引
对于联合索引，其大致结构是如下所示    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181107003122254_662505623.png)

大致是在第一个索引的基础上筛选出数据后，在根据后面的字段进行筛选    
## 索引覆盖
**索引覆盖指的是一次查询只需要从辅助索引中就能得到查询信息，而不需要查询聚簇索引中的数据**
### 当没有where条件的优化
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181107153730837_1584260342.png)

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181107160016933_13828841.png)

### 二次索引优化
```sql
 mysql> select sql_no_cache rental_date from t1 where inventory_id<80000;
…
…
| 2005-08-23 15:08:00 |
| 2005-08-23 15:09:17 |
| 2005-08-23 15:10:42 |
| 2005-08-23 15:15:02 |
| 2005-08-23 15:15:19 |
| 2005-08-23 15:16:32 |
+---------------------+
79999 rows in set (0.13 sec)

mysql> explain select sql_no_cache rental_date from t1 where inventory_id<80000\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t1
         type: range
possible_keys: inventory_id
          key: inventory_id
      key_len: 3
          ref: NULL
         rows: 153734
        Extra: Using index condition
1 row in set (0.00 sec)
```

可以看到Extra是using index condition，说明会通过index进行了回表查询
在rental_date字段增加索引之后：
```sql
mysql> explain select tid,return_date from t1 order by inventory_id limit 50000,10\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t1
         type: index
possible_keys: NULL
          key: liu
      key_len: 9
          ref: NULL
         rows: 50010
1 row in set (0.00 sec)
执行计划也可以看到，使用到了复合索引，并且不需要回表
```
