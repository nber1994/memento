--- 
title: redis集群 
date: 2019-01-08
categories: 
- redis 
---
- redis集群通过分片来进行数据共享，并且提供复制和故障转移的功能
# 节点
- 一个集群是由多个节点组成
- CLUSTER MEET ip port 命令用于将节点加入自己的集群中
- 集群中每个节点会维护的数据结构
    - clusterNode 保存了节点的状态，比如创建时间，名字，配置纪元，节点ip port等
    - clusterLink
    - clusterState
## clusterNode
```c
struct clusterNode {
    //创建时间
    mstimer_t ctime;
    //节点的名字
    char name[REDIS_CLUSTER_NAMELEN];
    //节点标识
    //主从节点，上线下线
    int flags;
    //当前配置纪元，用于故障转移
    uint64_t configEpoch;
    //节点IP
    char ip[REDIS_IP_STR_LEN];
    //节点端口
    int port;
    //保存连接节点所需的信息
    clusterLink *link;
    //保存节点的槽指派信息
    unsigned char slots[16384/8];
    //槽指派的个数
    int numslots；
    //复制的节点
    struct clusterNode *slaveof;
}
```
## clusterLink
- clusterLink保存了连接节点所需的信息
```c
typedef struct clusterLink {
    //连接创建时间
    mstime_t time;
    //TCP套接字描述符
    int fd;
    //输出缓冲区,保存待发给其他节点的消息
    sds sndbuf;
    //输入缓冲区，保存着从其他节点接收到的消息
    sds revbuf;
    //与这个连接相关联的节点，没有就为null
    struct clusterNode *node;
} clusterLink;
```
- redisClinet和clusterLink结构很像，但是redsiClient对应的是客户端的链接信息，而clusterLink对应的是和其他节点的连接信息
## clusterState
- 该结构保存了在当前节点的视角下，整个集群的状态
```c
typedef struct clusterState {
    //指向当前节点的指针
    clusterNode *self;
    //集群当前的配置纪元，用于实现故障转移
    uint64_t currentEpoch;
    //集群当前状态：上线还是下线
    int state;
    //集群中至少处理一个槽的节点的数量
    int size;
    //集群单节点名单（包括自己）
    dict *nodes;
    //记录了集群中槽的指派情况
    clusterNode *slot[16384];
    //正在导入的槽的节点
    clusterNode importing_slots_from[16384];
} clusterState;
```
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121154717423_1668313282.png)
# 集群
## cluster meet实现
- 当cluster meet ip port指向B
    - 节点A会为B在自己的clusterState.nodes中创建一个节点数据
    - 之后，A会根据ip+port向B发送meet命令
    - B接收到meet命令之后，会将A添加到自己的clusterState.nodes
    - 之后，B向A发送pong命令，表面自己已经接收到了meet消息
    - A收到B的pong消息之后，会向B发送ping命令
    - B收到A的ping命令之后，B知道已经成功的接收到了自己的pong消息，握手完成
    - 之后A会向集群的其他节点通过Gossip协议将B的信息同步
## 槽指派
- 集群通过分片的方式来保存数据库中的键值对：集群的整个数据库被分为了16384个槽
- 集群最多可以处理0或者多个槽
- 数据库中的16384个槽如果都有节点在处理时，则我们说集群为上线状态，否则集群为下线状态
### 节点槽指派信息的记录
```c
struct clusterNode {
    unsigned char slots[16384/8];
    int numslots;
}
```
- slots是一个二进制位数组，长度为16384/8=2048字节，16384个bit位
- 如果索引i位的bit位1，则i槽指派给该节点
- 检查某个槽点和设置某个槽点的时间复杂度都为O(1)
- numslots为槽数组中为1的个数
- 节点还会把自己的槽指派结构发送到集群中其他的节点
- 节点在接收到其他节点的slots信息之后，会找到对应的clusterNode，并对其进行更新
### 集群槽指派信息的记录
```c
typedef struct clusterState {
    clusterNode *slots[16384];
} clusterState;
```
- 如果slots[i]为null表明槽i未指派任何节点
- slots[i]指向一个clusterNode，表明槽i指向该节点
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121114139727_2138546451.png)

## 在集群中执行命令
- 在集群中执行命令时，
    - 首先计算出键所在的槽数
    - 判断槽是否是本节点负责
        - 如果是则查找键值，并返回
        - 如果不是本节点负责，则返回moved错误，指引客户端转向到新的节点
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121114721112_116402118.png)

### 计算键属于哪个槽
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121114817495_12794809.png)
### 判断槽是否由当前节点处理
- 如果clusterState.slots[i]等于myself的话，则是本节点负责
- 如果不等于的话，则会去除clusterState.slots[i]指向节点的ip和port，返回moved错误指引客户端转向其他节点
### moved命令
- 客户端收到moved port ip错误之后，会转向新的节点，并重新执行命令
### 节点数据库的实现
- 节点保存键值对和过期键值对的方式和单机一抹一眼
- 但是节点只使用0号数据库
- 节点还会维护一个slots_to_keys跳跃表来保存槽和键之间的关系
```c
typedef struct clusterState {
    ziplist *slot_to_keys;
}
```
    - slot_to_keys是一个跳跃表，表节点的分值对应一个槽号，而成员变量则指向数据库键
    - 每当新增键时，都会讲这个键和键的槽号关联到slot_to_keys
    - 通过slot_to_keys结构，可以很方便的对某个或者某些槽的所有键进行批量操作
        - 例如cluster getkeyinslot slot count命令，返回最多count个属于slot的键1
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121120718879_293990163.png)

## 重新分片
- 集群中可以进行重新分片的操作，可以将任意数量的槽指派到另一个节点，所有关联的键值对会迁移到目标节点
- 重新分片可以在线进行，期间集群不需要下线并且可以正常响应用户的请求
- 重新分片是由redis管理工具redis-trib负责执行的，redis提供了所有的命令，redis-trib负责向目标和源节点发送命令实现重新分片
### 分片步骤
- redis-trib向目标节点发送cluster setslot <slot> importing <sourc_id>，让目标节点准备好从源节点导入键值对
    - 该命令会将slusterState中的importing_slots_from数组更新，importing_slots_from[i]的对应一个clusterNode
    ```c
    typedef struct clusterState {
        clusterNode *importing_slots_from[16384];
    } clusterState;
    ```
- redis-trib对源节点发送cluster setslot <slot> migrating <target_id>，让源节点准备好将键值对迁移到目标节点
    - 该命令会将clusterState中的migrating_slots_to记录下目标节点的信息
```c
    typedef struct clusterState {
        clusterNode *migrating_slots_to[16384];
    } clusterState;
```
- redis-trib向源节点发送cluster getkeysinslot slot count命令，获取最多count个属于slot的键值对的键名
- 对于获得的键名，redis-trib都向源节点发送一个migrate target_id target_port key_name 0 timeout命令，将键原子性的迁移至目标节点
### ASK错误
- 在重新分片期间，源节点的一个槽中的键值对可能存在一部分在本节点，而另一部分在其他节点的情况
- 当节点收到键请求，
    - 首先在自己的数据库中进行查找，如果存在则返回
    - 如果不存在，检查键值所在槽i对应的migrating_slots_to[i]是否为空，
        - 如果不为空，这个键可能已经迁移到了新的节点，则会返回客户端一个ask错误，指引客户端转向到其他节点再次执行命令
### asking命令
- 当客户端收到ask错误时，则会转向新的节点，首先发送asking命令，打开该节点的redis_asking标识
- 然后发送之前的命令，节点收到命令后，计算键值所处槽i
    - 如果是本节点负责槽，则正常执行
    - 如果不存在，则检查importing_slot_from[i]是否为空
        - 如果为空则表示没有迁移，返回moved
        - 如果不为空，则判断是否客户端带有asking标识
            - 存在，则破例执行该命令
            - 不存在则返回moved
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121143034033_1880281441.png)
- redis_asking标识是一个一次性的标识，当节点执行了一个带有redsi_asking标识客户端的命令之后，这个标识就会被移除
### ask和moved命令的区别
- moved错误代表槽的负责权已经由一个节点转移到另一个节点，之后客户端对于对于槽i的所有指令都会转移到新的节点
- ask是在槽迁移期间的一个临时措施，客户端之后关于槽i的请求还是会请求原来的节点
# 复制与故障转移
- redis集群中的节点分为主节点和从节点
    - 主节点处理槽
    - 从节点复制主节点，并在主节点下线后代替主节点
## 设置从节点
- 向一个节点发送cluster replicate node_id，会使该节点变为node_id节点的从节点
    - 从节点会更新自己clusterNode节点中的slaveof属性
```c
struct clusterNode {
    strutc clusterNode *slaveof;
} 
```
    - 从节点也会更新clusterState.self.flags属性，变为redis_node_slave
    - 从节点对主节点进行复制操作，该操作和单机复制操作一样
    - 从节点成为某个节点的从节点的消息会发送给集群的其他节点
## 故障检测
- 集群中的每个节点都会定期向集群中的其他节点发送ping消息，来检测对方是否在线
    - 如果规定时间内没有收到pong的回复，则会将该节点标记为疑似下线（pfail）
    - 并且会将该消息发送到集群的其他节点
    - 集群中的节点收到疑似下线的消息之后，会在自己的clusterState.nodes字典中找到疑似下线的节点，将下线报告添加到fail_reports链表里面
```c
struct clusterNode{
    //一个链表，记录了所有其他节点对该节点的下线报告
    list *fail_reports;
}
```
```c
struct clusterNodeFailReport {
    //报告目标节点下线的节点
    struct clusterNode *node;
    //最后一次被报告的时间
    mstime_t time;
}
```
- 当半数以上负责槽处理的节点认为某个节点疑似下线了，那么该节点会被标记为已下线(fail)
- 标记某个节点下线的节点会把该信息同步到集群的其他节点
    - 其他节点收到消息后会立即将下线节点的状态标记为下线
## 故障转移
- 当一个从节点发现主节点已经下线，从节点将开始对主节点进行故障转移
    - 复制节点中会有一个节点被选中
    - 该节点会执行slave no one，成为新的主节点
    - 从节点撤销所有对已下线主节点的槽指派，并将这些槽支配给自己
    - 新的主节点会广播一条pong命令，告诉集群新的主节点
    - 新的主节点开始处理槽，故障转移完成
### 选取新的主节点
- 新的主节点是通过选举产生的，具体方法为
    - 集群的配置纪元是一个自增计数器，初始为0
    - 当集群某个节点开始一次故障转移时，集群的配置纪元会加一
    - 对于每个配置纪元，每个节点只有一次投票机会。会投给第一个请求投票的节点
    - 当从节点发现主节点下线，会向集群广播cluster_type_failover_auth_request，要求节点想自己投票
    - 当某个节点具有投票权(负责处理槽)，并且还未投票，则会返回cluster_type_failover_auth_ack支持
    - 每个参与投票节点都会统计ack消息，当投票大于n/2+1，该从节点作为主节点
    - 如果一个配置纪元里没有选出主节点，则会进入新的配置纪元重新选举
## 消息
- 消息类型
    - meet
        - 要求meet节点加入集群
    - ping
        - 默认一秒钟会从节点列表中随机五个节点发送ping
        - 如果上次ping距离现在时间大于cluster-node-timeout的一般，也会发送ping
    - pong
        - 回复meet，ping
        - 故障转移之后，也会广播pong，来让其他节点更新对自己的状态
    - fall
        - 当一个节点认为另一个节点下线时，会广播一条fail消息，收到消息的节点会立即下线节点
        - 其实节点之间通过gossip协议，也会将一个节点下线的信息传递给其他节点，但是没有fail速度快
    - publish
        - 节点接收到publish命令时，节点会执行命令，同事会广播，收到的节点也会执行相同的publish操作
## publish消息的实现
- 客户端向某个节点发送publish channel message命令
    - 节点向channel发送message
    - 广播publish消息到其他节点
    - 收到publish消息的节点也会想channel发送message
- 向某个节点发送publish命令，会导致所有的节点都向channel发送publish消息
## 重点回顾
- 节点通过握手来讲其他节点添加到集群中
- 集群中的16384个槽都会被指派给其他的节点，每个节点都会记录哪些槽指派给了别人，哪些指派给了自己
- 节点收到命令时，检查键所在槽是否为该节点负责，否则会返回一个moved错误，指引客户端转向新的节点
- redis集群的重新分片工作是有redis-trib实现的，重新分片就是将槽中的所有键值对迁移到另外的节点
- 如果节点A正在迁移槽点i到节点B，当节点A没能查到键值时，会向客户端发送一个ack错误，指引客户端转向新的节点
- moved表示槽i的负责全转移到了另一个节点，ask只是两个节点在迁移过程中的历史措施
- 集群中从节点用于复制，主节点下线时，代替主节点
- 集群中的五种消息：meet，ping，pong，fail，publish
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181121165539219_1794835342.png)
