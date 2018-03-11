---
layout:     post
title:      tcp socket buffer和window size
date:       2018-01-06
author:     hsd
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - tcp
    - 网络编程
---

> tcp buffer size和window size
之前稍微有点搞混淆了，上网查了些资料弄得算是比较清楚了

## buffer
是内核中为socket维护的发送缓存和接收缓存。根据socket是否是blocking的，read、write有不同的表现：

1. read总是在接收缓冲区有数据时立即返回，而不是等到给定的read buffer填满时返回。只有当receive buffer为空时，blocking模式才会等待，而nonblock模式下会立即返回-1（errno = EAGAIN或EWOULDBLOCK）

2. blocking的write只有在缓冲区足以放下整个buffer时才返回（与blocking read并不相同）。nonblock write则是返回能够放下的字节数，之后调用则返回-1（errno = EAGAIN或EWOULDBLOCK）

对于blocking的write有个特例：当write正阻塞等待时对面关闭了socket，则write则会立即将剩余缓冲区填满并返回所写的字节数，再次调用则write失败（connection reset by peer）

java socket通过setSendBufferSize和setReceiveBufferSize设置

## window size
指的就是tcp协议中flow control使用的window字段，用于控制有多少可以在网络中运输的packet（即未被ack的packet），在bandwidth-delay product（带宽乘以延迟）比较高的情况下网络带宽利用率可能会比较低（延迟高，带宽也高，所有的packet都是突发的，然后发送端陷入等待）

linux上可以通过sysctl设置/proc/sys/net/ipv4/tcp_rmem参数，这个参数有三个值，分别是最小、默认、最大window size

    root@ghyt:~# cat /proc/sys/net/ipv4/tcp_rmem
    4096    8192    16384
    
也可以检查是否开启了windows scale选项，linux从2.6.8开始默认是开启这个选项的，可以通过/proc/sys/net/ipv4/tcp_window_scaling参数查看是否开启

## 参见
http://www.cnblogs.com/promise6522/archive/2012/03/03/2377935.html

https://serverfault.com/questions/775837/how-to-set-the-maximum-tcp-receive-window-size-in-linux

https://en.wikipedia.org/wiki/TCP_window_scale_option

不得不提tcp和unix socket编程领域的两本圣经级的书
《TCP/IP Illustrated, vol 1》 by Richard Stevens

《Unix Network Programming， vol 1》(3rd Edition) by Richard Stevens