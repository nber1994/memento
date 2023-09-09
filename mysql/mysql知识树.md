--- 
title: mysql知识树 
date: 2019-01-08
categories: 
- mysql 
---
# mysql架构
* mysql层的结构
* 查询优化器
* join 连表查询
* binlog
* ddl/dml/sql
* 查询缓存
* 查询优化
* explain
* sql军规
* 外键完整性约束
# 连接池
* 连接on-thread-per-con
* 线程池
# 存储引擎
* 存储引擎的区别
## InnoDB：
* 三大特性
* 索引 ：
* 索引覆盖
* B+，hashmap，聚簇索引，前缀，全文索引
* 索引建立的规则
* 事务&隔离级别
* 当前读，快照度
* mvcc 锁
* 事务实现
* acid
* redo log undo log
# 工程 分表分库分片
# 内存管理
