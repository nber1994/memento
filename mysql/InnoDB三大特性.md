--- 
title: InnoDB三大特性
date: 2019-01-08
categories: 
- mysql   
---
## 三大特性
* 插入缓冲
* 两次写
* 自适应哈希索引

## 自适应哈希索引
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181102200713773_1016629773.png)
InnoDB会监控表上二级索引的查找，如果耳机索引多次被查询，则会对该热数据建立hash索引  
这些操作都是InnoDB自己的行为，用户控制不了
