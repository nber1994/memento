--- 
title: redis应用场景 
date: 2019-08-09
categories: 
- redis 
---

```c
127.0.0.1:6379> select 1
OK
```
# 字符串对象

## 常用命令
```c
# 字符串对象

## 设置
127.0.0.1:6379[1]> set a a
OK

## 获取
127.0.0.1:6379[1]> get a
"a"

## 删除
127.0.0.1:6379[1]> del testString
(integer) 1

## 设置并返回原值
127.0.0.1:6379[1]> get b
"b"
127.0.0.1:6379[1]> getset b c
"b"

## 增加 减少，追加
127.0.0.1:6379[1]> set testInt 1
OK
127.0.0.1:6379[1]> incr testInt
(integer) 2
127.0.0.1:6379[1]> decr testInt
(integer) 1
127.0.0.1:6379[1]> append a a
(integer) 2
127.0.0.1:6379[1]> get a
"aa"
127.0.0.1:6379[1]> incrby int 2
(integer) 3
127.0.0.1:6379[1]> decrby int 2
(integer) 1
```
## 应用场景
* 可以使用incr作为计数器，且不用担心并发问题

# hash对象
## 常用命令
```c

# hash对象

## 设置，批量设置，获取，批量获取
127.0.0.1:6379[1]> hset myhash key value
(integer) 1
127.0.0.1:6379[1]> hset myhash name jty
(integer) 1
127.0.0.1:6379[1]> hset myhash age 18
(integer) 1
127.0.0.1:6379[1]> hset myhash sex male
(integer) 1
127.0.0.1:6379[1]> hmset myhash type 1 status 2 level A
OK
127.0.0.1:6379[1]> hget myhash sex
"male"
127.0.0.1:6379[1]> hmget myhash name age sex type
1) "jty"
2) "18"
3) "male"
4) "1"
127.0.0.1:6379[1]> hgetall myhash
 1) "key"
 2) "value"
 3) "name"
 4) "jty"
 5) "age"
 6) "18"
 7) "sex"
 8) "male"
 9) "type"
10) "1"
11) "status"
12) "2"
13) "level"
14) "A"

## 增加键值
127.0.0.1:6379[1]> hincrby myhash age 6
(integer) 24

## 判断是否存在
127.0.0.1:6379[1]> hexist myhash age
(error) ERR unknown command 'hexist'
127.0.0.1:6379[1]> hexists myhash age
(integer) 1

## hash表大小
127.0.0.1:6379[1]> hlen myhash
(integer) 7

## 获取所有所有键名
127.0.0.1:6379[1]> hkeys myhash
1) "key"
2) "name"
3) "age"
4) "sex"
5) "type"
6) "status"
7) "level"

## 获取所有键值
127.0.0.1:6379[1]> hvalues myhash
(error) ERR unknown command 'hvalues'
127.0.0.1:6379[1]> hvalues myhash
(error) ERR unknown command 'hvalues'
```
## 应用场景
* 存储变更的数据

# 列表对象
```c
# 列表对象

## 对头对尾添加节点
127.0.0.1:6379[1]> lpush mylist 1 2 3
(integer) 3
127.0.0.1:6379[1]> lpush mylist a b c
(integer) 6

## 范围获取列表节点
127.0.0.1:6379[1]> lrange mylist 0 5
1) "c"
2) "b"
3) "a"
4) "3"
5) "2"
6) "1"
127.0.0.1:6379[1]> lrange mylist 0 -1
1) "c"
2) "b"
3) "a"
4) "3"
5) "2"
6) "1"

## 弹出节点
127.0.0.1:6379[1]> lpop mylist
"c"
127.0.0.1:6379[1]> rpop mylist
"1"

## 获取列表长度
127.0.0.1:6379[1]> llen mylist
(integer) 4

## 删除节点
127.0.0.1:6379[1]> lrem mylist 2 1
(integer) 0

## 设置节点值
127.0.0.1:6379[1]> lset mylist 2 sb
OK
127.0.0.1:6379[1]> lrange mylist 0 -1
1) "x"
2) "b"
3) "sb"
4) "2"

## 插入节点
127.0.0.1:6379[1]> linsert mylist after 2 insert2
(integer) 5
127.0.0.1:6379[1]> lrange mylist 0 -1
1) "x"
2) "b"
3) "sb"
4) "2"
5) "insert2"
127.0.0.1:6379[1]> linsert mylist after a aftera
(integer) 8

## 弹出某个节点并添加到另一个链表
127.0.0.1:6379[1]> rpoplpush mylist mylist2
"insert2"

```
## 应用场景
* lrpoplpush 多个程序间消息的交互
    * 消费者lpop获取列表1的元素，为了确保消费成功，则需要把该元素rpush到另一个列表2里，等待消费成功确认后将该元素从列表2中删除掉
    * 当一定时间以后列表2中的元素还未被删除，则会将该消息重新push到列表1中
* 微博关注列表，粉丝列表
* 也可以用来抢购，如果有是个库存，则每来一个从list中pop一个元素
* 最新列表，获取最新登录的人的列表

# 集合

```c

# 集合对象

## 添加元素
127.0.0.1:6379[1]> sadd myset a b c
(integer) 3

## 删除元素
127.0.0.1:6379[1]> srem myset a
(integer) 1

## 获取所有元素
127.0.0.1:6379[1]> smembers myset
1) "c"
2) "b"

## 判断是否是成员
127.0.0.1:6379[1]> sismember myset a
(integer) 0
127.0.0.1:6379[1]> sismember myset b
(integer) 1

## 获取集合差集
127.0.0.1:6379[1]> sdiff myset myset1
1) "a"
127.0.0.1:6379[1]> sdiff myset1 myset
1) "d"

## 获取交集
127.0.0.1:6379[1]> sinter myset myset1
1) "c"
2) "b"

## 获取并集
127.0.0.1:6379[1]> sunion myset myset1
1) "b"
2) "c"
3) "a"
4) "d"

## 获取集合大小
127.0.0.1:6379[1]> scard myset
(integer) 3

## 随意获取一个集合
127.0.0.1:6379[1]> srandmember myset 1
1) "a"
127.0.0.1:6379[1]> srandmember myset 2
1) "b"
2) "c"

## 获取差异并存入另一个集合
127.0.0.1:6379[1]> sdiffstore myset myset1 myset2
(integer) 3

## 获取交集并存入另一个集合
127.0.0.1:6379[1]> sinterstore myset2 myset myset1
(integer) 3
127.0.0.1:6379[1]> smembers myset2
1) "b"
2) "c"
3) "d"

## 获取并集并存入另一个集合
127.0.0.1:6379[1]> sunionstore myset2 myset myset1
(integer) 4
127.0.0.1:6379[1]> smembers myset2
1) "a"
2) "b"
3) "c"
4) "d"
```

## 应用场景
* 类似于布隆过滤器的功能
    * 存储访问者的IP地址
    * 查询用户名是否被注册
    * 限制用户一天只能投一次票
* 利用集合的并，交集功能
    * 获取同时购买两种商品的人
    * 共同喜好，共同关注

# 有序集合

```c

# 有序集合

## 添加元素
127.0.0.1:6379[1]> zadd mysort 100 jty 98 eb 99 sa 60 ab
(integer) 4

## 获取元素个数
127.0.0.1:6379[1]> zcard mysort
(integer) 4

## 获取某个元素分值
127.0.0.1:6379[1]> zscore mysort jty
"100"

## 删除某个元素
127.0.0.1:6379[1]> zrem mysort sa
(integer) 1

## 获取某个元素的分值
127.0.0.1:6379[1]> zscore mysort sa
(nil)

## 根据范围获取
127.0.0.1:6379[1]> zrange mysort 0 -1
1) "ab"
2) "eb"
3) "jty"
127.0.0.1:6379[1]> zrange mysort 0 -1 withscores
1) "ab"
2) "60"
3) "eb"
4) "98"
5) "jty"
6) "100"

## 根据排名删除
127.0.0.1:6379[1]> zremrangebyrank mysort 0 1
(integer) 2

## 根据分数删除
127.0.0.1:6379[1]> zremrangebyscore mysort 0 30
(integer) 2

## 根据分值获取元素
127.0.0.1:6379[1]> zrangebyscore mysort 20 100
1) "a"
2) "b"
3) "c"
4) "d"

## 增加某个元素的值
127.0.0.1:6379[1]> zincrby mysort 100 a
"120"

## 获取某个范围内的元素个数
127.0.0.1:6379[1]> zcount mysort 0 100
(integer) 3

## 获取某个元素的排名
127.0.0.1:6379[1]> zrank mysort a
(integer) 3

# 通用

## 获取对象的所有键名
127.0.0.1:6379[1]> keys a
1) "a"
127.0.0.1:6379[1]> keys myso?
(empty list or set)
127.0.0.1:6379[1]> keys mys?
(empty list or set)

## 获取超时时间
127.0.0.1:6379[1]> ttl mysort
(integer) -1
```
## 常用场景
* 排行榜
* 延时队列
    * 将当前时间戳作为score，同时通过某些算法（例如uuid4），计算出消息对应的key，作为name
    * 算出当前时间段通过zrangebyscore进行获取
    * 用途
        * 定时清理，超过一定空闲时间进行释放
        * 下订单30分钟后未付款则取消订单
        * 下单后60s发送短信
