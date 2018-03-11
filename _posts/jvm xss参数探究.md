---
layout:     post
title:      jvm xss参数含义探究
date:       2017-03-11
author:     hsd
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - jvm
---

> jvm xss参数含义探究
最近为了准备面试，重温了一下周志明的jvm虚拟机那本书。看到有关stackOverflowError的部分，有些疑惑。上面说jvm虚拟机规范中规定：
    1.如果线程请求的栈深度大于虚拟机允许的最大深度，将抛出stackoverflowerror异常。
    2.如果扩展栈失败，oom异常

# xss参数
hotspot的参数xss用于设定虚拟机方法栈和本地方法栈的大小，但这个大小不是很明确，jvm规范里栈的实现可以是fix-sized或者是dynamically expandable的，没查到hotspot是fixed还是dynamic的，
但查了下linux的内存管理，大概了解这个参数的具体含义了（不排除理解有误，持续了解中）

# linux内存overcommit技术
进程只有使用到对应的内存页时os才会进行分配，因此进程申请的内存总数（committed memory）是可以大于ram+swap的，这种技术叫overcommit，其中swap的内存有个公式，其中m是ram大小（GB）：mem(swap) = m > 2? m+2: m*2

linux寄希望于应用进程不会使用所有内存，但万一用了太多，发生oom内存不足了，linux就有个oom-killer，来杀掉它认为造成内存不足的进程。linux里有个overcommmit机制，允许，有内核参数 vm.overcommit_memory 可以配置linux的行为：

- 0: 启发式overcommmit，这是缺省值，它允许overcommit，但过于明目张胆的overcommit会被拒绝，比如malloc一次性申请的内存大小就超过了系统总内存。Heuristic的意思是“试探式的”，内核利用某种算法（对该算法的详细解释请看文末）猜测你的内存申请是否合理，它认为不合理就会拒绝
- 1: 总是允许overcommit，对内存申请来者不拒。不过发生oom的时候进程还是会被杀掉的啦
- 2: 禁止overcommit

对于参数2，系统得有个阙值来判断什么是overcommit

    grep -i commit /proc/meminfo
    CommitLimit:     5967744 kB
    Committed_AS:    5363236 kB
其中CommitLimit就是这个判断overcommit的阙值，commited_as是进程们已申请的内存。使用sar -r也可以查看内存使用情况

    02:10:01 PM kbmemfree kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
    02:20:01 PM   1754884   1147820     39.54       748    533080   2618460     52.37    525904    417408        12
    02:30:02 PM   1759852   1142852     39.37       748    533108   2568432     51.37    526160    417380        12
    Average:      1757368   1145336     39.46       748    533094   2593446     51.87    526032    417394        12
其中kbcommit是进程申请的内存总数，%commit是kbcommit/ram+swap，不过和上面的/proc/meminfo稍微有点差距不知道为什么

对于默认参数0，启发式overcommit，大概是单次申请的内存大小不能超过 【free memory + free swap + pagecache的大小 + SLAB中可回收的部分】，否则本次申请就会失败

# 总结
根据前面对linux overcommit的介绍，在linux下jvm应该是申请xss大小的栈，不过绝大多数情况下都用不到这么多，只有在jvm实际用到更多的时候linux才会给jvm分配，当分配不了时jvm抛出oom，当栈大小超出xss时就会抛出stackOverflowException。OOM还是极少见的，一般都是stackoverflow

# 参见
http://linuxperf.com/?p=102
https://www.etalabs.net/overcommit.html
