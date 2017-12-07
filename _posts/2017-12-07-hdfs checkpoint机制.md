---
layout:     post
title:      hdfs checkpoint机制
date:       2017-12-07
author:     hsd
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - 大数据
    - hadoop
---
>理解hdfs中检查点如何工作的对区分健康的集群和失败的集群很有意义,译文，见[原文](https://blog.cloudera.com/blog/2014/03/a-guide-to-checkpointing-in-hadoop/)
# 理解hdfs中检查点如何工作的对区分健康的集群和失败的集群很有意义
检查点是维护和持久化hdfs文件系统元数据的重要组成部分，对于高效的namenode恢复和重启是至关重要的，是集群健康状况的重要指示器。然呢，检查点也可以是给hadoop集群运维人员带来疑惑的来源。在这篇文章中，我会解释hdfs中使用检查点的目的，在不同集群配置下检查点是如何工作的，最后以一些运维问题和重要bug修复结尾。


# hdfs中文件系统元数据

首先，我们来看下namenode是怎么持久化元数据的

![](/img/hdfs_namenode.png)

从高层看，namenode的主要责任是存储hdfs的命名空间，比如说目录树，文件权限，文件到块id的映射。安全地将这些元数据（以及对它的改变）持久化到可靠存储对于保证可用性来说是至关重要的
文件系统元数据使用两种不同的形式进行存储：fsimage和edit log。fsimage表示文件系统元数据在某个时间点的快照，然而，虽然fsimage的文件格式能很高效地读取，却不适合making小的增量的更新比如说重命名一个文件。因此，namenode将修改操作记录到editlog中而不是一有改变就写fsimage。这样，如果namenode宕机了，可以首先将fsimage载入然后将editlog中记录的操作重放，以赶上系统最新的状态。editlog由一系列的文件组成，叫做edit log段（segments），一起表示自从fsimage创建以来的所有对namesystem的修改。

另外，这种模式对于传统文件系统来说是很普遍的，log-structured filesystem使用这种模式到了极限，只只使用log来存储数据，但更普遍的日志文件系统比如ext3、ext4和xfs支持在变更持久化到磁盘前先将变更记录到日志中

# 为什么检查点如此重要
A typical edit ranges from 10s to 100s of bytes，但随着时间的推移edit log会变得很笨重。庞大的editlog会产生一堆问题。极端情况下， 它会占满所有节点上所有可用的磁盘空间，但更微妙的是，庞大的editlog会极大地拖慢namenode的启动因为namenode需要重放所有这些更改。因此checkpointing油然而生

checkpointing是将fsimage和editlog组装成一个新的fsimage的过程，这样，namenode可以直接载入fsimage最后的内存状态，这是一个更高效的操作，降低了namenode的启动时间。

![](/img/hdfs_checkpoint.png)

然而，创建一个新的fsimage是一个很消耗io和cpu的操作，有时花费上好几分钟。在checkpoint过程中，namesystem需要限制来自其他用户的并发访问。因此，与其暂停活跃的namenode，hdfs将这个操作推给secondary namenode或者叫standy namenode取决于nameonde高可用是否配置了。这两种情况我们都会讨论

然而在任一中情况，checkpointing被以下任一中情况出发：1.自从上次checkpoint以来已经过了足够的时间（dfs.namenode.checkpoint.period）2.足够的editlog事务次数（dfs.namenode.checkpoint.txns）。checkpointing node周期性地检查上两种情况是否满足（dfs.namenode.checkpoint.check.period），如果是，启动checkpointing过程

# checkpiointing with a standby namenode
checkpointing和ha设置比起来要简单得多，因此，我们先讨论

当namenode高可用被配置时，active和standby namenode拥有edit的共享存储，一般来说，共享存储由三个或以上的日志节点组成（journal node），但这部分从checkpointing过程抽象出去了。

standby namenode通过周期性地重放active namenode写到共享edit文件夹中的新edit，维持着一个相对较新的命名空间。结果就是，checkpoingting就是简单地检查两个先决条件中满足了哪个，保存命名空间到一个新的fsimage（大致等同于执行hdfs dfsadmin –saveNamespace命令行），然后通过http传输新的fsimage到active namenode
	
![](/img/hdfs_checkpoint_ha.png)

这里，standby namenode简写为SbNN，Active NameNode简写为ANN：
1. Sbnn检查是否满足了两个先决条件中的一个：上次checkpoint依赖经过的时间或累计的edits的数量
2. SbNN将的namespace保存到一个新的fsimage中，中间名字使用fsimage.ckpt_，txid就是最新的edit日志事务的事务ID。然后SbNN给fsimage写一个MD5文件，将fsimage重新命名到fsimage_。当这发生时，大多数其他SbNN操作都被阻塞了，这意味着管理操作比如namenode故障恢复或访问SbNN web网页的一部分。日常的HDFS客户端操作比如listing、reading和writing files不受影响因为这些操作是由ANN提供服务的
3. SbNN发送一个HTTP get请求给ANN的GetImageServlet（getimage?putimage=1）。这个URL参数也带了新fsimage的事务ID和SbNN的主机名和HTTP端口
4. ANN Servlet使用Get请求中的信息来给SbNN的GetImageServlet发送GET请求。类似于standby，ANN首先将新的fsimage使用一个中间名称fsimage.ckpt_保存到文件中，然后将新fsimage重命名到fsimage_

# SecondaryNameNode上的Checkpointing
在一个非HA的部署中，checkpointing通过SecondaryNameNode而不是standby Namenode完成的。因为没有共享的edits文件夹或自动tailing edit日志，SecondaryNameNode不得不经过一些更多的步骤，即在继续相同的基本步骤之前进行namespace视图更新。

![](/img/hdfs_checkpoint_non_ha.png)

这里，NameNode简写做NN，SecondaryNameNode简写做2NN:
1. 2NN检查两个先决条件是否被满足：…
2. 在没有共享edit目录的情况下，最近的edit日志的事务ID需要通过向NN发起显式的RPC查询（NamenodeProtocol#getTransactionId）.
3. 2NN触发edit日志滚动(edit log roll…不太好翻译)，这将结束当前的edit日志片段，开启一个新的。在SNN压缩之前的片段（all the previous ones）的时候，NN可以继续写edit到新的片段中。这同时也会返回当前fsimage的事务ID和刚roll过的edit日志片段。显式触发edit日志roll在HA配置下不是必要的，因为standby NameNode会周期roll edit日志，正交于checkpointing
4. 如果必要的话，2NN使用新下载的fsimage重新加载它的命名空间
5. 2NN重放的新的edit log片段来追上现在的transaction ID，从这里开始，剩下的和HA例子中的一样的
6. 2NN将它的命名空间写到一个新的fsimage中
7. 2nn和nn通过http get请求（/getimage?putimage=1），使得nn的servlet发送get请求到2nn上下载新的fsimage
	
# 运维启示（operational implications)
关于checkpointing最大的运维考虑就是当它不能发生时。我们已经见到一些场景，里面namenode累积了成百GB的edit logs，然后没有人注意到知道最后磁盘被完全填满了使得nn崩溃，当这个发生时，除了重启NN等待它重放所有的edit之外没有太多其他可以做的。因为这个话题潜在的严重性，当当前的fsimage过期之后或检查点2nn、sbnn宕机后或nn的磁盘接近饱和后，Cloudera Manager会进行警告
checkpointing是一个非常耗费io和网络的操作，会影响到客户端性能，对于有着百万级的文件和几个GB的fsimage来说这尤为正确，因为将新的fsimage拷贝到namenode可以吃掉所有可用的带宽。在这种情况下，可以通过dfs.image.transfer.bandwidthPerSec来限制传输速率，如果你调整了这个参数，你可能也需要基于你预期的传输时间调整dfs.image.transfer.timeout

如果你使用更老版本的CDH(译注:应该是cloudera hadoop吧)，有一些和checkpointing相关的话题可能会让升级变值得
- HDFS-4304 (fixed in CDH 4.1.4, 4.2.1, and 4.3.0). Previously, it was possible to write an edit log operation so big that it couldn’t be read when replaying the edit log. This would cause checkpointing and NameNode startup to fail. This was a problem for files with lots of blocks, since closing a file involved writing all the block IDs to the edit log. The fix was to simply increase the size of the maximum allowable edit log operation.
- HDFS-4305 (fixed in CDH 4.3.0). Related to HDFS-4304 above, files with a large number of blocks are typically due to misconfiguration. For example, a user might accidentally set a block size of 128KB rather than 128MB, or might only use a single reducer for a large MapReduce job. This issue was fixed by having the NameNode enforce a minimum block size as well as a maximum number of blocks per file.
- HDFS-4816 (fixed in CDH 4.5.0). Previously, image transfer from the standby NN to the active NN held the standby NN’s write lock. Thus if the active NN failed during image transfer, the standby NN would not be able to failover until the transfer completed. Since transferring the fsimage doesn’t actually modify any namespace data, the transfer was simply moved outside the critical section.
- HDFS-4128 (fixed in CDH 4.3.0). If the 2NN hit an out-of-memory (OOM) exception during edit log replay, it could get stuck in an inconsistent state where it would try replaying the edit log from the incorrect offset during future checkpointing attempts. Since it’s likely that this OOM would keep happening even if we fixed log replay, the 2NN now simply aborts if it fails to replay logs a few times. The underlying fix though is to configure your 2NN with the same heap size as your NameNode.
- HDFS-4300 (fixed in CDH 4.3.0). If the 2NN or SbNN experienced an error while transferring an edits file, it would not retry downloading the complete file later. This process would stall checkpointing, since it’d be impossible to replay the partial edits file. This issue was fixed by first transferring the edits file to a temporary location and then renaming it to its final destination after transfer completes.
- HDFS-4569 (fixed in CDH4.2.1 and CDH4.3.0). Bumped the image transfer timeout from 1 minute to 10 minutes. The default timeout was causing issues for checkpointing with multi-GB fsimages, especially with throttling turned on.

# 结论
checkpointing是健康的hdfs运维一个至关重要的部分。在这篇博客中，你学到了文件系统元数据是如何在hdfs中持久化的，checkpointing在其中的重要性，checkpointing在ha和non-ha配置中分别是如何工作的，最后覆盖了一些挑选过的关于checkpointing重要的修复和改进

通过理解checkpointing的目的以及它是如何工作的，现在你知道在你的生产集群中如何调试这类的问题了
