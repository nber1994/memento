--- 
title: ping命令
date: 2019-08-09
categories: 
- network
---
> ping A
* 基于ICMP，网络层，速度快
* 拼接IP包，加入A为目的地址，以及ICMP报文组成IP包发送至网络中寻址
* 到达目的IP机器子网后，会根据ARP协议，发送到目的IP机器，目的IP机器收到包后解析，然后封装响应包
