--- 
title: InnoDB文件 
date: 2019-01-08
categories: 
- mysql 
---
# 文件
- ## 参数文件
- ## 日志文件
    1. 错误日志
    2. 二级制日志
        1. 恢复
        2. 复制
        3. 审计
    3. 慢查询日志
    4. 查询日志
- ## socket文件
- ## pid文件
- ## Mysql表结构定义文件
- ## InnoDB存储引擎文件
### 表空间文件
innodb存储按照表空间进行存放。初始化有一个10M的ibdata1文件，文件就是默认的表空间文件    
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181107213409117_494097334.png)

### 重做日志文件
每个innodb引擎至少与一个重做日志文件组，每个组内至少有两个重做日志文件    
并以循环写入的方式运行
