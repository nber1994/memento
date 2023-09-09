--- 
title: http基础知识 
date: 2019-01-08
categories: 
- http
---
# http
- http是无状态的协议
- 为了保持http的状态，引入了cookie
- 请求报文结构
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128161900440_707298140.png)
- 响应报文结构
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128161926144_1237852835.png)

## http方法
- get
    - 获取资源
- post
    - 传输实体
- put
    - 传输文件
- head
    - 获取报文首部
- delete
    - 删除文件
- options
    - 获取url支持的方法
- trace
    - 将到达web服务器之前的通信节点换回给客户端
    - max-forward首部字段填入数值，每经过一个服务器该值就减一，当减为0时停止传输，并就地返回200 ok
## 持久连接节省通信量
- keep-alive 
    - 一个页面上会有多个需要http请求的资源
    - 如果每个资源都会建立和断开tcp连接，增加网络开销
    - 因此增加了keep-alive的方法，只要任一端没有明确断开连接，就会一直保持连接
- pipline
    - keep alive方法使得管线化成为可能
    - 之前一个http请求完成之后，才会请求下一个http请求
    - 管线化则同时发生多个请求，比持久化连接更快
## 使用cookie记录状态
- 客户端会根据服务端返回的响应报文中的set-cookie的首部字段，来保存cookie
- 当再次发生请求给服务端时，会将cookie一并发送给服务端

![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128164200907_1918509356.png)

- 响应报文
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128164226475_2067270234.png)

- 请求报文

## http报文
- http协议中传输的数据叫做http报文
    - 请求报文
    - 响应报文
- http会分为报文首部和报文主体两部分
- 两者之间由/r/n分割
### 报文结构
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128164553814_472898025.png)
- 请求报文
    - 报文首部
        - 请求行
        - 请求首部字段
        - 通用首部字段
        - 实体首部字段
    - 报文主体
        - 响应行
        - 响应首部字段
        - 通用首部字段
        - 实体首部字段
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128164929628_160128628.png)

## http状态码
- 1XX 信息性状态码
    - 接受的请求正在处理
- 2XX 成功状态码
    - 请求正常处理完毕
        - 200 ok
        - 204 No Content 
            - 请求成功，但是没有资源返回，此时客户端不会更新
        - 206 Paticial Content
            - 表示响应部分请求成功
- 3XX 重定向码
    - 需要附加操作以完成请求
        - 301 Moved Permanently
            - 永久重定向之后，以后对该uri的访问应该都使用最新的uri
        - 302 Found
            - 临时重定向，请求URI对应的资源临时分配了新的资源，暂时使用新的URI请求
        - 303 See Other
            - 与302具有相同的功能，但是规定访问新的URI时使用get方法
        - 304 Not Modified
            - 服务端允许访问资源，但是不满足条件时，会返回304
- 4XX 客户端错误状态码
    - 服务器无法处理请求
        - 400 Bad Request
            - 表示请求报文语法错误
        - 401 unauthorized
            - 没有权限
        - 403 Forbidden
            - 没有资源访问权限
        - 404 Not Found
            - 没有找到资源
- 5XX 服务端出错状态码
    - 服务器处理请求出错
        - 500 Internal Server Error
            - 服务端内部出错
        - 503 Service Unavaliable
            - 服务端暂时处于超负载或正在停机维护
            - 最好同时返回RetryAfter字段
## 报文首部结构
- 类型
    - 请求报文首部字段
        - Accept
            - 标明客户端可以接受什么格式的响应
        - Accept Charset
            - 客户端可以接受的字符集
        - Accept-Encoding
            - 客户端支持的编码
        - Accept Language
            - 客户端希望的语言
        - Host
        - If-Match
            - 用来匹配请求资源是否和标记匹配（ETag）
            ![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128181228604_203395618.png)
        - If-Modified_Since
            - 在时间点后是否改变
    - 响应报文首部字段
    - 通用首部字段
        - Cache-Control 可以使用no-cache来不是有缓存
        - Connection
            - 控制不再转发给代理的首部字段
             ![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128180525077_1653907998.png)
            - 管理持久连接
                - http1.1默认都是keep-alive，当想断开时，将connection置为close
        - Date
            - 创建日期
    - 实体首部字段
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128173919638_484841105.png)
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128173937060_1171495017.png)
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128173952665_1186675023.png)
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20181128174004923_766481872.png)

## http code
200 ok 响应成功
204 no content 成功但是没有内容返回
301 moved permanently 永久重定向
302 Found 临时重定向
303 See Other  临时重定向 但是使用GET
400 bad request 请求语义错误
401 Unauthorized 需要身份验证
403 forbidden 服务器拒绝执行请求
404 not found 资源找不到
500 internal server error 服务内部错误
502 bad gateway 服务器作为网关拿到了错误的响应
503 service unavaliable 服务还未准备好接收请求
504 gateway timeout 服务器作为网关，未能及时得到响应
