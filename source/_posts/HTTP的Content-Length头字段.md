---
title: HTTP的Content-Length头字段
date: 2021-12-04 23:30:31
categories:
- 网络
tags: 
- HTTP
---





# HTTP的Content-Length头字段

***Content-Length***用于**描述HTTP消息实体的传输长度**（*the transfer-length of the message-body*）。

在`HTTP`协议中，消息实体长度和消息实体的传输长度是有区别的。

> 消息实体长度：消息实体本身的长度。
>
> 消息实体的传输长度：消息实体在传输时的长度。

比如说gzip压缩下，消息实体长度就是压缩前的长度，消息实体的传输长度是gzip压缩后的长度。

在具体的HTTP交互中，客户端是如何获取消息长度的呢，主要基于以下几个规则：
响应为`1xx`，`204`，`304`或者head请求，则直接忽视掉消息实体内容。

如果有***Transfer-Encoding***，则优先采用***Transfer-Encoding***里面的方法来找到对应的长度。比如说***Chunked***模式。

如果***head***中有***Content-Length***，那么这个***Content-Length***既表示实体长度，又表示传输长度。

如果实体长度和传输长度不相等（比如说设置了***Transfer-Encoding***），那么则不能设置***Content-Length***。如果设置了***Transfer-Encoding***，那么***Content-Length***将被忽视。

**有chunk就不能有content-length** 。

其实后面几条几乎可以忽视，简单总结后如下：
1、***Content-Length***如果存在并且有效的话，则必须和**消息内容的传输长度**完全一致。（经过测试，如果过短则会截断，过长则会导致超时。）
2、如果存在***Transfer-Encoding***（重点是***chunked***），则在***header***中不能有***Content-Length***，有也会被忽视。
3、如果采用短连接，则直接可以通过服务器关闭连接来确定消息的传输长度。（这个很容易懂）

结合`HTTP`协议其他的特点，比如说HTTP1.1之前的不支持keep alive。那么可以得出以下结论：
1、在`HTTP1.0`及之前版本中，***Content-Length***字段可有可无。
2、在`HTTP1.1`及之后版本。如果是***keep alive***，则***Content-Length***和***chunk***必然是二选一。若是非***keep alive***，则和`HTTP1.0`一样。***Content-Length***可有可无。

