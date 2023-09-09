---
title: mysql的explain关键字
date: 2017-10-27
categories: 
- mysql
---
> 胡乱的记录了一些explain关键字的一些用法,以及相关的评判参数

# explain概述
EXPLAIN关键字一般放在SELECT查询语句的前面，用于描述MySQL如何执行查询操作、以及MySQL成功返回结果集需要执行的行数。explain 可以帮助我们分析 select 语句,让我们知道查询效率低下的原因,从而改进我们查询,让查询优化器能够更好的工作。

# how to work
MySQL 查询优化器有几个目标,但是其中最主要的目标是尽可能地使用索引,并且使用最严格的索引来消除尽可能多的数据行。最终目标是提交 SELECT 语句查找数据行,而不是排除数据行。优化器试图排除数据行的原因在于它排除数据行的速度越快,那么找到与条件匹配的数据行也就越快。如果能够首先进行最严格的测试,查询就可以执行地更快。

# explain的各个参数的详解

## id

| 项目 | 说明 |
| --- | --- |
| id | MySQL Query Optimizer 选定的执行计划中查询的序列号。表示查询中执行 select 子句或操作表的顺序,id 值越大优先级越高,越先被执行。id 相同,执行顺序由上至下 |

```sql
 **id**SQL执行的顺利的标识,SQL从大到小的执行.

例如:
mysql> explain select * from (select * from ( select * from t3 where id=3952602) a) b;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table      | type   | possible_keys     | key     | key_len | ref  | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  2 | DERIVED     | <derived3> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  3 | DERIVED     | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       |      |    1 |       |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+

很显然这条SQL是从里向外的执行,就是从id=3 向上执行.
```

## select_type

### select_type-SIMPLE
**首先是select__type:将select查询分为简单(simple)和复杂两种类型复杂类型又分为子查询(subquery)和from列表中包含子查询(drived)**

| select_type 查询类型 | 说明 |
| --- | --- |
| SIMPLE | 简单的 select 查询,不使用 union 及子查询 |

```sql
简单SELECT(不使用UNION或子查询等) 例如:
mysql> explain select * from t3 where id=3952602;
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
| id | select_type | table | type  | possible_keys     | key     | key_len | ref   | rows | Extra |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
|  1 | SIMPLE      | t3    | const | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 |       |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
```

### select_type-PEIMARY

| select_type 查询类型 | 说明 |
| --- | --- |
| PRIMARY | 最外层的 select 查询 |

```sql
mysql> explain select * from (select * from t3 where id=3952602) a ;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table      | type   | possible_keys     | key     | key_len | ref  | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  2 | DERIVED     | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       |      |    1 |       |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
```

### select_type-UNION

| select_type 查询类型 | 说明 |
| --- | --- |
| UNION | UNION 中的第二个或随后的 select 查询,不依赖于外部查询的结果集 |

```sql
UNION中的第二个或后面的SELECT语句.例如
mysql> explain select * from t3 where id=3952602 union all select * from t3 ;
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
| id | select_type  | table      | type  | possible_keys     | key     | key_len | ref   | rows | Extra |
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
|  1 | PRIMARY      | t3         | const | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 |       |
|  2 | UNION        | t3         | ALL   | NULL              | NULL    | NULL    | NULL  | 1000 |       |
|NULL | UNION RESULT | <union1,2> | ALL   | NULL              | NULL    | NULL    | NULL  | NULL |       |
+----+--------------+------------+-------+-------------------+---------+---------+-------+------+-------+
```

### select_type-DEPENDENT UNION

| select_type 查询类型 | 说明 |
| --- | --- |
| DEPENDENT UNION | UNION 中的第二个或随后的 select 查询,依 赖于外部查询的结果集 |

```sql
UNION中的第二个或后面的SELECT语句，取决于外面的查询

mysql> explain select * from t3 where id in (select id from t3 where id=3952602 union all select id from t3)  ;
+----+--------------------+------------+--------+-------------------+---------+---------+-------+------+--------------------------+
| id | select_type        | table      | type   | possible_keys     | key     | key_len | ref   | rows | Extra                    |
+----+--------------------+------------+--------+-------------------+---------+---------+-------+------+--------------------------+
|  1 | PRIMARY            | t3         | ALL    | NULL              | NULL    | NULL    | NULL  | 1000 | Using where              |
|  2 | DEPENDENT SUBQUERY | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 | Using index              |
|  3 | DEPENDENT UNION    | t3         | eq_ref | PRIMARY,idx_t3_id | PRIMARY | 4       | func  |    1 | Using where; Using index |
|NULL | UNION RESULT       | <union2,3> | ALL    | NULL              | NULL    | NULL    | NULL  | NULL |                          |
+----+--------------------+------------+--------+-------------------+---------+---------+-------+------+--------------------------+
```

### select_type-SUBQUERY

| select_type 查询类型 | 说明 |
| --- | --- |
| SUBQUERY | 子查询中的第一个 select 查询,不依赖于外 部查询的结果集 |

```sql
子查询中的第一个SELECT.

mysql> explain select * from t3 where id = (select id from t3 where id=3952602 )  ;
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------------+
| id | select_type | table | type  | possible_keys     | key     | key_len | ref   | rows | Extra       |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------------+
|  1 | PRIMARY     | t3    | const | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 |             |
|  2 | SUBQUERY    | t3    | const | PRIMARY,idx_t3_id | PRIMARY | 4       |       |    1 | Using index |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------------+
```

### select_type-DEPENDENT SUBQUERY

| select_type 查询类型 | 说明 |
| --- | --- |
| DEPENDENT SUBQUERY | 子查询中的第一个 select 查询,依赖于外部 查询的结果集 |

```sql
子查询中的第一个SELECT，取决于外面的查询

mysql> explain select id from t3 where id in (select id from t3 where id=3952602 )  ;
+----+--------------------+-------+-------+-------------------+---------+---------+-------+------+--------------------------+
| id | select_type        | table | type  | possible_keys     | key     | key_len | ref   | rows | Extra                    |
+----+--------------------+-------+-------+-------------------+---------+---------+-------+------+--------------------------+
|  1 | PRIMARY            | t3    | index | NULL              | PRIMARY | 4       | NULL  | 1000 | Using where; Using index |
|  2 | DEPENDENT SUBQUERY | t3    | const | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 | Using index              |
+----+--------------------+-------+-------+-------------------+---------+---------+-------+------+--------------------------+
```

### select_type-DERIVED

| select_type 查询类型 | 说明 |
| --- | --- |
| DERIVED | 用于 from 子句里有子查询的情况,MySQL会递归执行这些子查询, 把结果放在临时表里 |

```sql
派生表的SELECT(FROM子句的子查询)

mysql> explain select * from (select * from t3 where id=3952602) a ;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table      | type   | possible_keys     | key     | key_len | ref  | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  2 | DERIVED     | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       |      |    1 |       |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
```

### select_type-UNCACHEABLE SUBQUERY

| select_type 查询类型 | 说明 |
| --- | --- |
| UNCACHEABLE SUBQUERY | 结果集不能被缓存的子查询,必须重新为外 层查询的每一行进行评估 |

### select_type-UNCACHEABLE UNION

| select_type 查询类型 | 说明 |
| --- | --- |
| UNCACHEABLE UNION | UNION 中的第二个或随后的 select 查询,属 于不可缓存的子查询 |


## table

| 项 | 说明 |
| --- | --- |
| table | 输出引用到的表 |

```sql
显示这一行的数据是关于哪张表的.
有时不是真实的表名字,看到的是derivedx(x是个数字,可以理解为是第几步执行的结果)

mysql> explain select * from (select * from ( select * from t3 where id=3952602) a) b;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table      | type   | possible_keys     | key     | key_len | ref  | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  2 | DERIVED     | <derived3> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  3 | DERIVED     | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       |      |    1 |       |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
```

## type
**这列很重要,显示了连接使用了哪种类别,有无使用索引**
**从最好到最差的连接类型为const、eq__reg、ref、range、indexhe和ALL**

### type-system

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| system | 表仅有一行(=系统表)。这是 const 连接类型的一个特例 |

```sql
这是const联接类型的一个特例。表仅有一行满足条件.如下(t3表上的id是 primary key)

mysql> explain select * from (select * from t3 where id=3952602) a ;
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
| id | select_type | table      | type   | possible_keys     | key     | key_len | ref  | rows | Extra |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL              | NULL    | NULL    | NULL |    1 |       |
|  2 | DERIVED     | t3         | const  | PRIMARY,idx_t3_id | PRIMARY | 4       |      |    1 |       |
+----+-------------+------------+--------+-------------------+---------+---------+------+------+-------+
```

### type-const

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| const | const 用于用常数值比较 PRIMARY KEY 时。当查询的表仅有一行时,使用 System, **说明索引一次就找到了,最多只会有一条匹配的行** |

```sql
表最多有一个匹配行，它将在查询开始时被读取。因为仅有一行，在这行的列值可被优化器剩余部分认为是常数。const表很快，因为它们只读取一次！

const用于用常数值比较PRIMARY KEY或UNIQUE索引的所有部分时。在下面的查询中，tbl_name可以用于const表：
SELECT * from tbl_name WHERE primary_key=1；
SELECT * from tbl_name WHERE primary_key_part1=1和 primary_key_part2=2；

例如:
mysql> explain select * from t3 where id=3952602;
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
| id | select_type | table | type  | possible_keys     | key     | key_len | ref   | rows | Extra |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
|  1 | SIMPLE      | t3    | const | PRIMARY,idx_t3_id | PRIMARY | 4       | const |    1 |       |
+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
```

### type-eq_ref

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| eq_ref | const 用于用常数值比较 PRIMARY KEY 时。当查询的表仅有一行时,使用 System,**使用唯一索引查找(主键或唯一索引)** |

```sql
对于每个来自于前面的表的行组合，从该表中读取一行。这可能是最好的联接类型，除了const类型。它用在一个索引的所有部分被联接使用并且索引是UNIQUE或PRIMARY KEY。

eq_ref可以用于使用= 操作符比较的带索引的列。比较值可以为常量或一个使用在该表前面所读取的表的列的表达式。

在下面的例子中，MySQL可以使用eq_ref联接来处理ref_tables：

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
    AND ref_table.key_column_part2=1;

例如
mysql> create unique index  idx_t3_id on t3(id) ;
Query OK, 1000 rows affected (0.03 sec)
Records: 1000  Duplicates: 0  Warnings: 0

mysql> explain select * from t3,t4 where t3.id=t4.accountid;
+----+-------------+-------+--------+-------------------+-----------+---------+----------------------+------+-------+
| id | select_type | table | type   | possible_keys     | key       | key_len | ref                  | rows | Extra |
+----+-------------+-------+--------+-------------------+-----------+---------+----------------------+------+-------+
|  1 | SIMPLE      | t4    | ALL    | NULL              | NULL      | NULL    | NULL                 | 1000 |       |
|  1 | SIMPLE      | t3    | eq_ref | PRIMARY,idx_t3_id | idx_t3_id | 4       | dbatest.t4.accountid |    1 |       |
+----+-------------+-------+--------+-------------------+-----------+---------+----------------------+------+-------+
```

### type-ref

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| ref | 连接不能基于关键字选择单个行,可能查找 到多个符合条件的行。 叫做 ref 是因为索引要 跟某个参考值相比较。这个参考值或者是一 个常数,或者是来自一个表里的多表查询的 结果值,**多个匹配,非唯一索引访问(只有普通索引)** |

```sql
对于每个来自于前面的表的行组合，所有有匹配索引值的行将从这张表中读取。如果联接只使用键的最左边的前缀，或如果键不是UNIQUE或PRIMARY KEY（换句话说，如果联接不能基于关键字选择单个行的话），则使用ref。如果使用的键仅仅匹配少量行，该联接类型是不错的。

ref可以用于使用=或<=>操作符的带索引的列。

在下面的例子中，MySQL可以使用ref联接来处理ref_tables：

SELECT * FROM ref_table WHERE key_column=expr;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
    AND ref_table.key_column_part2=1;

例如:

mysql> drop index idx_t3_id on t3;
Query OK, 1000 rows affected (0.03 sec)
Records: 1000  Duplicates: 0  Warnings: 0

mysql> create index idx_t3_id on t3(id) ;
Query OK, 1000 rows affected (0.04 sec)
Records: 1000  Duplicates: 0  Warnings: 0

mysql> explain select * from t3,t4 where t3.id=t4.accountid;
+----+-------------+-------+------+-------------------+-----------+---------+----------------------+------+-------+
| id | select_type | table | type | possible_keys     | key       | key_len | ref                  | rows | Extra |
+----+-------------+-------+------+-------------------+-----------+---------+----------------------+------+-------+
|  1 | SIMPLE      | t4    | ALL  | NULL              | NULL      | NULL    | NULL                 | 1000 |       |
|  1 | SIMPLE      | t3    | ref  | PRIMARY,idx_t3_id | idx_t3_id | 4       | dbatest.t4.accountid |    1 |       |
+----+-------------+-------+------+-------------------+-----------+---------+----------------------+------+-------+
2 rows in set (0.00 sec)
```

### type-ref_or_null

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| ref_or_null | 如同 ref, 但是 MySQL 必须在初次查找的结果 里找出 null 条目,然后进行二次查找 |

```sql
该联接类型如同ref，但是添加了MySQL可以专门搜索包含NULL值的行。在解决子查询中经常使用该联接类型的优化。

在下面的例子中，MySQL可以使用ref_or_null联接来处理ref_tables：

SELECT * FROM ref_table
WHERE key_column=expr OR key_column IS NULL
```

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| index_merge | 说明索引合并优化被使用了 |

```sql
该联接类型表示使用了索引合并优化方法。在这种情况下，key列包含了使用的索引的清单，key_len包含了使用的索引的最长的关键元素。

例如:
mysql> explain select * from t4 where id=3952602 or accountid=31754306 ;
+----+-------------+-------+-------------+----------------------------+----------------------------+---------+------+------+------------------------------------------------------+
| id | select_type | table | type        | possible_keys              | key                        | key_len | ref  | rows | Extra                                                |
+----+-------------+-------+-------------+----------------------------+----------------------------+---------+------+------+------------------------------------------------------+
|  1 | SIMPLE      | t4    | index_merge | idx_t4_id,idx_t4_accountid | idx_t4_id,idx_t4_accountid | 4,4     | NULL |    2 | Using union(idx_t4_id,idx_t4_accountid); Using where |
+----+-------------+-------+-------------+----------------------------+----------------------------+---------+------+------+------------------------------------------------------+
1 row in set (0.00 sec)
```

### type-unique_subquery

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| unique_subquery | 在某些 IN 查询中使用此种类型,而不是常规的 ref:value IN (SELECT primary_key FROM single_table WHERE some_expr) |

```sql
该类型替换了下面形式的IN子查询的ref：

value IN (SELECT primary_key FROM single_table WHERE some_expr)
unique_subquery是一个索引查找函数，可以完全替换子查询，效率更高
```

### type-index_subquery

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| index_subquery | 在某些IN查询中使用此种类型,与unique_subquery类似,但是查询的是非唯一性索引: value IN (SELECT key_column FROM single_table WHERE some_expr) |

```sql
该联接类型类似于unique_subquery。可以替换IN子查询，但只适合下列形式的子查询中的非唯一索引：

value IN (SELECT key_column FROM single_table WHERE some_expr)
```

### type-range

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| range | 只检索给定范围的行,使用一个索引来选择 行。key 列显示使用了哪个索引。当使用=、 <>、>、>=、<、<=、IS NULL、<=>、BETWEEN 或者 IN 操作符,用常量比较关键字列时,可 以使用 range |

```sql
只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引。key_len包含所使用索引的最长关键元素。在该类型中ref列为NULL。

当使用=、<>、>、>=、<、<=、IS NULL、<=>、BETWEEN或者IN操作符，用常量比较关键字列时，可以使用range

mysql> explain select * from t3 where id=3952602 or id=3952603 ;
+----+-------------+-------+-------+-------------------+-----------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys     | key       | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+-------------------+-----------+---------+------+------+-------------+
|  1 | SIMPLE      | t3    | range | PRIMARY,idx_t3_id | idx_t3_id | 4       | NULL |    2 | Using where |
+----+-------------+-------+-------+-------------------+-----------+---------+------+------+-------------+
1 row in set (0.02 sec)

```

### type-index

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| index | 全表扫描,只是扫描表的时候按照索引次序 进行而不是行。主要优点就是避免了排序, 但是开销仍然非常大按索引次序扫描，先读索引，再读实际的行，结果还是全表扫描，主要优点是避免了排序。因为索引是排好的 |

```sql
该联接类型与ALL相同，除了只有索引树被扫描。这通常比ALL快，因为索引文件通常比数据文件小。

当查询只使用作为单索引一部分的列时，MySQL可以使用该联接类型
```

### type-all

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| all | 最坏的情况,从头到尾全表扫描 |


### type-null

| type 重要的项,显示连接使用的类型,按最 优到最差的类型排序 | 说明 |
| --- | --- |
| null | 优化过程中就已经得到结果，不在访问表或索引 |

## possible_keys

| 项 | 说明 |
| --- | --- |
|possible_keys | 指出 MySQL 能在该表中使用哪些索引有助于 查询。如果为空,说明没有可用的索引 |

```sql
possible_keys列指出MySQL能使用哪个索引在该表中找到行。注意，该列完全独立于EXPLAIN输出所示的表的次序。这意味着在possible_keys中的某些键实际上不能按生成的表次序使用。

如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查WHERE子句看是否它引用某些列或适合索引的列来提高你的查询性能。如果是这样，创造一个适当的索引并且再次用EXPLAIN检查查询
```

## key

| 项 | 说明 |
| --- | --- |
| key | MySQL 实际从 possible_key 选择使用的索引。 如果为 NULL,则没有使用索引。很少的情况 下,mysql 会选择优化不足的索引。这种情 况下,可以在 SELECT 语句中使用 USE INDEX (indexname)来强制使用一个索引或者用 IGNORE INDEX(indexname)来强制 mysql 忽略索引 |

## key_len

| 项 | 说明 |
| --- | --- |
| key_len | 使用的索引的长度。在不损失精确性的情况 下,长度越短越好 |

## ref

| 项 | 说明 |
| --- | --- |
| ref | 显示索引的哪一列被使用了 |

## rows

| 项 | 说明 |
| --- | --- |
| rows | mysql 认为必须检查的用来返回请求数据的行数 |


## Extra 
* 该列包含MySQL解决查询的详细信息
**extra 中出现以下 2 项意味着 mysql 根本不能使用索引,效率会受到重大影响。应尽可能对此进行优化**

| 项 | 说明 |
| --- | --- |
| Using filesort | 表示 MySQL 会对结果使用一个外部索引排序,而不是从表里按索引次序读到相关内容。可能在内存或者磁盘上进行排序。MySQL 中无法利用索引完成的排序操作称为“文件排序”, **对于没有索引的列进行order by 就会出现filesort** |
| Using temporary | 表示 MySQL 在对查询结果排序时使用临时表。常见于排序 order by 和分组查询 group by, **MySQL 使用临时表来实现 distinct 操作** |
| Using index | 此查询使用了覆盖索引(Covering Index)，即通过索引就能返回结果，无需访问表。若没显示"Using index"表示读取了表数据。|
| Using where | 表示 MySQL 服务器从存储引擎收到行后再进行“后过滤”（Post-filter）。所谓“后过滤”，就是先读取整行数据，再检查此行是否符合 where 句的条件，符合就留下，不符合便丢弃。因为检查是在读取行后才进行的，所以称为“后过滤” |
| Distinct | 一旦mysql找到了与行相联合匹配的行，就不再搜索了 |
| Not exists | mysql优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行，就不再搜索 |
| Range checked for each | 没有找到理想的索引，因此对于从前面表中来的每一个行组合，mysql检查使用哪个索引，并用它来从表中返回行。这是使用索引的最慢的连接之一 |


# 联合查询的优化

* 简单举例子，LEFT JOIN 条件用于确定如何从右表搜索行,左边一定都有,所以右边是我们的关键点,一定需要建立索引
* inner join 和 left join 差不多,都需要优化右表。而 right join 需要优化左表

