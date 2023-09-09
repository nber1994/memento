--- 
title: InnoDB概述 
date: 2019-08-09
categories: 
- mysql 
---
InnoDB完整支持ACID事务

# 体系架构
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181107164912667_493096811.png)


InnoDB内部存在多个内存块，可以看做是内存池  
内存池负责如下工作:  

- 维护内部数据结构
- 缓存磁盘上的数据，方便的快速读取。同时对磁盘文件的修改进行缓存
- redo log的缓冲

后台线程：

- 负责刷新内存池中数据，保证缓冲池中的是最近的数据
- 将已经修改的数据文件刷新到磁盘中
- 保证在发送异常时回复

# 后台线程
## master thread
负责将缓冲池中的数据异步刷新到磁盘，脏页刷新，合并插入缓存等
## IO thread
InnoDB使用了大量的AIO处理IO请求，负责IO请求的回调
## purge thread
回收事务提交后的已经使用的undo页

# 内存
## 缓冲池
InnoDB是基于磁盘的数据库，通常使用缓冲池技术来提高数据库的整体性能  
1. 读取数据
缓冲池简单来说就是一块内存区域，当读取某一页时，会将数据从磁盘读取到缓冲池中，提高性能
2. 写入数据
对数据的修改，会首先修改缓冲池中的页，然后再已一定的频率将修改刷回磁盘，这里采用的是checkpoint机制刷到磁盘，而不是每次发生页更新时触发
**相关参数** innodb_buffer_pool_size
3. 缓冲池缓冲页类型：索引页，数据页，undo页，插入缓冲，自适应哈希索引，innodb存储的锁信息等  

 ![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181107171027175_338708704.png)

## LRU list和Free list和flush list
通常来说缓冲池是通过LRU算法来实现的，使用最频繁的页在列表的最前面，最少使用的页在对尾，当没有更多的内存时，首先释放列尾的页  
**在缓冲池中，每页的大小为16K**
### lru算法的改进
引入midpoint，新进入的页会放在midpoint的位置  
改进的原因是因为当有大的结果的sql出现时，如果放入对首的话，会把热点数据给覆盖掉，而该sql不一定是会频繁出现的  
同时还引入了一个innodb_old_blocks_time，规定了多久之后会将old数据放入new列表的对首

### free list
该list记录得是所有的未分配的页面，当innodb刚启动是，缓冲池所有的页都存在于free list中
### flush list
当LRU列表中的页被修改后，该页面被称为脏页，缓冲池中的数据与磁盘中的数据不一致。此时会通过checkpoint机制来讲脏页刷回磁盘，  
所以flush list中的都是脏页，同时lru列表中也会存在脏页

## 重做日志缓冲
InnoDB中除了缓冲池外，还有redo log缓冲，InnoDB会先将redo log存到这个缓冲区，然后按照一定概率将其刷新到日志文件中  
redo log会在如下情况下刷入磁盘：

- master thread每秒会刷新到文件
- 事务提交时会将日志刷新到文件（不一定commit group）
- 重做日志缓冲剩余空间小于1/2时

# checkpoint技术
当页发送变化时将页的版本刷新到磁盘，开销是十分大的。同时如果从缓存页落地磁盘时发送宕机时，是无法恢复的。
所以采用了wirte ahead技术，在事务提交时，先写重做日志，再去修改页面，这样一来，宕机之后可以通过redo log    
来恢复数据    
> checkpoint技术解决的问题

> 1. 缩短数据库的恢复时间

当发生宕机时，数据库并不用重做所有的日志，只需要在checkpoint之后的数据进行重做即可

> 2. 缓冲池不够时，将脏页刷回磁盘

当缓冲池不够时，lru算法会溢出最少使用的页，如果为脏页则会强制执行checkpoint

> 3. 重做日志不可用时，刷新脏页

redo log并不是无限大的，是循环写入的，在checkpoint之前的数据是可以覆盖写的

**对于InnoDB来说，是通过LSN(log sequence number)来标记版本的**  
**checkpoint是用过LSN来实现**
> LSN是自增的日志序列编号，当数据库启动时，会比较page的LSN和redo log的LSN，如果不相同则会同步
每一页都有LSN，redo log也有LSN，checkpoint也有LSN
## checkpoint种类
而Checkpoint所做的事情无外乎是将缓冲池中的脏页刷回到磁盘，不同之处在于每次刷新多少页到磁盘，每次从哪里取脏页，以及什么时间触发Checkpoint。        

- sharp checkpoint
在数据库关闭时将所有的脏页刷新回磁盘
- fuzzy checkpoint
刷新一部分到磁盘

# InnoDB关键特性
InnoDB特性包括：
1. 插入缓冲
2. 两次写
3. 自适应哈希索引
4. 异步IO
5. 刷新临近页
## 插入缓冲 insert buffer
听起来insert buffer像是缓冲池的一部分，实则不然。insert buffer是物理页的一部分    
## 为什么要引入insert buffer
对于innodb来说，主键是自增的，所以每次新增一条数据，并不需要随机存取，速度较快，但是一张表不止一个索引    
对于辅助索引来说，当不是唯一索引的时候，新增操作会引起离散的访问节页。这会降低插入的性能
## insert buffer如何工作
当插入一条数据时，先判断非聚簇索引页是否在缓冲池中，如在的话，则直接在缓冲池中修改或插入。    
如果不在的话，则会先放在insert buffer中，然后再已一定的频率将insert buffer和辅助索引的merge。    
这时可以将多个insert操作合并为一个，提高性能    
## insert buffer条件
- 索引是辅助索引
- 索引不唯一
## 宕机危害
服务器宕机时，没有合并的insert buffer会丢失
## 升级版本change buffer
innodb1.0之后，引入了change buffer，可以对insert，update，delete进行缓存
## insert buffer的内部实现
**insert buffer 为一颗B+树**

## double wirte两次写
inser buffer带来的是性能上的提升，但是double write就是数据页的可靠性
## 为什么引入double write
因为innodb每页大小为16k，但是文件系统来说，每一块大小为4k，这样来说，在这一页正在写入时发生了宕机，    
则此时会发生部分写失效，此时redo log是不能恢复的，因为redo log记录的都是改变，也无法进行恢复，因此，    
需要首先从副本还原该页，然后进行redo
## double write实现
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181107210717275_1111400420.png)

double buffer分为两部分，一部分是内存中，大小为2M，另一部分在共享表的物理磁盘上，大小也为2M    
**在缓冲区的脏页被刷新到磁盘时**， 首先会memcpy将数据复制到内存中的缓冲区中，然后分为两次，每次1M的    
将数据刷入磁盘缓存区中，随后调用sync同步。由于是顺序写，则开销不会很大。之后再将磁盘缓冲区中的数据离散的    
写入磁盘相应位置
## 恢复
当将页写入磁盘时发送崩溃时，在共享表空间中的double write找到副本，将其复制到表空间文件之后再应用重做日志    

## 自适应哈希索引AHI
当观察到有很多等值查询，则innodb会自动建立自适应哈希索引，和B+树共存，可以直接通过哈希索引来访问

## 异步IO
每次扫描完一个页的同时，可以同时扫描很多个页，然后等待所有完成即可
## 刷新临近页
将脏页刷入磁盘时，会检测所在区extent的所有脏页，一起刷入磁盘
