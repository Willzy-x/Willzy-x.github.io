---
layout:     post
title:      QUIC 协议
subtitle:   一种基于UDP的快速可靠的网络协议
date:       2020-08-27
author:     HYC
header-img: img/high_speed_train.jpg
catalog: true
tags:
    - QUIC
    - Network
    - UDP
---

## 1. 简介

Quick UDP Internet Connections（QUIC）是一种实验性的网络传输协议。由Google开发，它在两个端点建立链接，且支持多路复用（multiplexing）。

> 多路复用简称复用通常表示在**一个信道上**传输**多路信号或数据流**的过程和技术。而且根据使用的技术可以分为：
>
> - 时分复用（TDM）
> - 频分复用（FDM）
> - 空分复用（SDM）
> - 码分复用（CDM）



HTTP-Over-QUIC正式更名为HTTP/3，这标志着QUIC将在几年内成为支持HTTP的主要协议。



![quic](http://ychu.top/img/materials/quic.png)



## 2. 特性



### 2.0 操作系统问题

TCP和UDP协议都是在操作系统层面的，因此应用程序只能使用不能修改，这就导致了TCP等协议非常缓慢的迭代。然而QUIC是基于用户空间的，QUIC就相对于TCP和UDP更加灵活易用。



### 2.1 没有队头阻塞的多路复用

通过使用UDP协议解决了HTTP/2中TCP层的**队头阻塞**问题（队头阻塞主要是 TCP 协议的可靠性机制引入的。TCP 使用序列号来标识数据的顺序，数据必须按照顺序处理，如果前面的数据丢失，后面的数据就算到达了也不会通知应用层来处理）。



QUIC的多路复用和HTTP/2类似，在一条QUIC链接上可以并发发送多个HTTP请求（Stream）。但是相对HTTP/2来说，QUIC有一个很大的优势：QUIC一个链接上的多个stream之间没有依赖。加入某个stream丢失了一个UDP数据包，影响的只会是那个stream的处理，而不影响其他的stream。



虽然多路复用是一个很强大的特性，但是同时也恶化了TCP的一个问题就是”队头阻塞“。因为QUIC的stream之间是独立的，不会影响后来的stream，因此不存在队头阻塞问题。然而，并不是所有的QUIC数据都不会受到队头阻塞的影响，由于算法的限制，stream丢失一个头部数据的时候，可能遇到队头阻塞。



### 2.2 快速建立链接

QUIC可以0RTT**建立链接的速度更快**，而TCP需要3次握手，2RTT才能即建立链接，对于短链接的情况来说，延迟太大。0RTT建立*传输层*链接，0RTT建立*加密层*链接。



先复习一下RTT的定义是什么：

> The **round-trip delay** (**RTD**) or **round-trip time** (**RTT**) is the length of time it takes for a signal to be sent plus the length of time it takes for an acknowledgement of that signal to be received.

即信号发送给接收方的时间加上接收方传回的确认信号的时间。为什么说QUIC是0RTT建立链接呢？是因为建立在UDP的基础上，QUIC发送握手数据后，即链接建立，接收方就可以直接发送数据回去了而不需要确认（acknowledge），所以是0RTT。



![quic_compare](http://ychu.top/img/materials/quic_compare.png)



### 2.3 链接的复用

链接迁移功能使得**链接可以最大程度的复用**，不再有重建链接的消耗。



一条TCP链接是由一个4元组（双方的IP地址和端口）标识的，如果当4元组中任意一个元素发生变化的时候这条链接依然维持着，能够保证业务逻辑不中断。



那QUIC是怎么做的呢？因为QUIC不再以上面提到的4元组作为链接的标识，而是使用一个**64-bit的随机数**作为ID来标识。这样就算IP或者端口发生变化的时候只要ID不变，这个链接就已然维持着。因此，上层业务逻辑感知不到变化不会中断，也就不需要重连。



### 2.4 快速修复数据

FEC（Forward-Error-Correction）可以**更快地修复恢复丢失数据**。



FEC的原理就是，当QUIC发送数据时，可以带上一些额外的数据字节，以防某些数据包无法到达的情况发生。QUIC通过XOR的形式来实现FEC，这种方式非常简单迅速而且提供了N+1的冗余数据应对数据包无法全部到达的情况。



这个基于XOR的修正系统有一个“组”数据包的概念。这个“组”中的任意一个数据包丢失都可以通过“组”中的最后一个XOR数据包来恢复。而且由于FEC数据包在网络上也是有负载的，因此从拥塞控制的角度来说，QUIC会将FEC数据包和其他的数据包同等对待。



## 3. 实现



### 3.1 客户端

Google的Chrome于2012年开始开发QUIC协议并且在2013年8月20日释出。现在QUIC协议在当前Chrome版本中被默认开启。



### 3.2 服务器端

截至2017年，有三种活跃维护中的实现。谷歌的服务器及谷歌发布的[原型服务器](https://code.google.com/p/chromium/codesearch#chromium/src/net/tools/quic/quic_server.cc)使用Go语言编写的[quic-go](https://github.com/lucas-clemente/quic-go)及[Caddy](https://zh.wikipedia.org/wiki/Caddy)的试验性QUIC支持。



其他的一些实现可以在[这里](https://github.com/quicwg/base-drafts/wiki/Implementations)找到：



## Reference

1. [https://zh.wikipedia.org/wiki/%E5%BF%AB%E9%80%9FUDP%E7%BD%91%E7%BB%9C%E8%BF%9E%E6%8E%A5](https://zh.wikipedia.org/wiki/快速UDP网络连接)
2. https://zhuanlan.zhihu.com/p/32553477
3. https://docs.google.com/presentation/d/15e1bLKYeN56GL1oTJSF9OZiUsI-rcxisLo9dEyDkWQs/edit#slide=id.gaf8802b44_0_87
4. https://www.youtube.com/watch?v=RIFnXaiRs_o
5. https://docs.google.com/document/d/1Hg1SaLEl6T4rEU9j-isovCo8VEjjnuCPTcLNJewj7Nk/edit#


