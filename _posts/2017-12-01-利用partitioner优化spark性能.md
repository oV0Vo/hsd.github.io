---
layout:     post
title:      利用partitioner优化spark性能
date:       2017-12-01
author:     hsd
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - 大数据
    - spark
---
>spark中数据传输是一个很昂贵的操作，优化spark性能往往集中于如何减少数据传输，利用好partitioner可以极大地减少不必要的网络传输，我们以join操作为例来看看如何进行优化

# 总结
&emsp;并不是特别高深的话题，总结一点就是spark会利用rdd的partition信息对join等操作进行优化，其他能从partitioner受益的操作有cogroup(), groupWith(), join(), leftOuterJoin(), rightOuterJoin(), groupByKey(), reduceByKey(), combineByKey(), lookup()
# 以Join操作为例
&emsp;假设我们有userData RDD，相对较大，每一会userData就join一下一个新的events RDD，比如
<pre>
// Initialization code; we load the user info from a Hadoop SequenceFile on HDFS.
// This distributes elements of userData by the HDFS block where they are found,
// and doesn't provide Spark with any way of knowing in which partition a
// particular UserID is located.
val sc = new SparkContext(...)
val userData = sc.sequenceFile[UserID, UserInfo]("hdfs://...").persist()
// Function called periodically to process a logfile of events in the past 5 minutes;
// we assume that this is a SequenceFile containing (UserID, LinkInfo) pairs.
def processNewLogs(logFileName: String) {
val events = sc.sequenceFile[UserID, LinkInfo](logFileName)
val joined = userData.join(events)// RDD of (UserID, (UserInfo, LinkInfo)) pairs
val offTopicVisits = joined.filter {
	case (userId, (userInfo, linkInfo)) => // Expand the tuple into its components
	!userInfo.topics.contains(linkInfo.topic)
}.count()
println("Number of visits to non-subscribed topics: " + offTopicVisits)
}
</pre>
&emsp;下图解释了join操作会产生的网络传输，每一次join操作都会产生一次如下的传输，图中每一个小块对应于RDD的一个分区
![](/img/spark_join.png)
&emsp;如果我们对userData一开始就进行partitionBy操作，将userData的数据进行分区（spark一开始并不知道原始数据文件是否有经过良好分区），即定义一个partitioner，然后rdd.partitionBy(partitioner)
<pre>
val sc = new SparkContext(...)
val userData = sc.sequenceFile[UserID, UserInfo]("hdfs://...")
                 .partitionBy(new HashPartitioner(100))   // Create 100 partitions
                 .persist()
</pre>
&emsp;那么join操作产生的网络传输将会变成这样，由于我们已经对userData进行分区了，那么join操作不会移动userData而是将eventsData shuffle到相应的userData的分区（使用userData的partitioner）
![](/img/spark_join_optimized.png)
&emsp;如果两个rdd的partitioner相等（对象的equals方法），那么join操作压根就不会移动数据

# 其他会从partitioner中受益的操作
&emsp;cogroup(), groupWith(), join(), leftOuterJoin(), rightOuterJoin(), groupByKey(), reduceByKey(), combineByKey(), lookup()
&emsp;注意如果对于一个key-value的rdd，我们对其进行转换但不会改变key只改变value的话，应该使用mapValue操作而不是map操作，因为map操作会清空partitioner（spark并不会检测我们的map代码是否会改变key）