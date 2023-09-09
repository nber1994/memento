--- 
title: redis问题 
date: 2019-08-09
categories: 
- redis 
---
# redis-问题
BRPOP，BLPOP

https://github.com/antirez/redis/issues/2576
why redis-cluster use 16384 slots? crc16() can have 2^16 -1=65535 different remainders。
原因是：

正常的心跳包携带节点的完整配置，可以用幂等方式替换旧节点以更新旧配置。这意味着它们包含原始形式的节点的插槽配置，它使用带有16k插槽的2k空间，但使用65k插槽将使用高达8k的空间。
同时，由于其他设计权衡，Redis Cluster不太可能扩展到超过1000个主节点。
因此，16k处于正确的范围内，以确保每个主站有足够的插槽，最多1000个主站，但足够小的数字可以轻松地将插槽配置传播为原始位图。请注意，在小型集群中，位图难以压缩，因为当N很小时，位图将设置插槽/ N位，这是设置的大部分位。
