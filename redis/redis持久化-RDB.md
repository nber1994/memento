--- 
title: redis持久化-RDB 
date: 2019-01-08
categories: 
- redis 
---
> redis持久化-RDB
- 二进制文件
- 一个服务器的所有非空数据库的所有的键值对成为数据库状态
- redis是一个内存型数据库，为了解决持久化问题，引入了RDB持久化和AOF持久化
## RDB文件的创建与载入
- 创建
    - RDB文件可以由save和bgsave命令生成
    - save会阻塞服务器进程，知道save命令结束，否则不会接受任何请求
    - gbsave命令则不同，他会派生一个子进程来进行任务，并不会阻碍redis的正常请求
- 载入
    - RDB文件的载入是在服务器启动时，检测到存在RDB文件的话就会自动载入RDB文件
- 值得一提的是，由于AOF文件更新的频率比RDB更快，所以redis会优先选择AOF文件来还原数据库状态
- 当没有开启AOF持久化功能时，才会采用RDB文件来恢复
### save和bgsave的互斥性
- 在bgsave期间，bgsave和save服务器会拒绝，这是因为为了防止竞争产生
- bgsave和bgrewariteaof也不能同时执行，虽然没有什么冲突，但是为了性能考虑
### 载入状态下服务器的状态
- 阻塞
## 对自动保存的设置
- redis允许用户对bgsave的频率进行设置
- 其中设置频率会设置两个指标
    - dirty计数器表示了上次save之后修改的键的数目
    - lastsave 上次bgsave的时间
- serverCron会定期的检查是否满足save的条件，从而进行保存
## RDB文件的结构
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118184724932_1056938818.png)
- REDIS是最开头的部分，保存着REDIS来标识是否是RDB文件
- db_version标识RDB版本
- databases包含着0或多个数据库的键值对数据
- EOF标识着RDB文件结束
- check_sum是根据其他部分计算出来的，用来校验RDB文件是否有损坏

### database部分
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118185640663_2133469789.png)

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118185656485_692289764.png)
- 每个databases包含着三个部分
    - selectdb 标志着是一个database段的开始
    - db_number标识出是哪个数据库
    - key_valud_pairs 存储着键值对
#### key_value_pairs部分
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118185945179_247664102.png)
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118190104694_667254428.png)
- type标识是哪种对象和实现方式
- EXPIRETIME 标志位，标识接下来读入的是一个到期时间
- ms 到期的时间戳
## 重点回顾
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181118190312976_385566762.png)
