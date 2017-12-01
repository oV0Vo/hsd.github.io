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
# Join操作是如何进行的
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
&emsp;一图胜千言，假设我们对userData和events进行join操作（userData.join(events))，图中每一个小块对应于RDD的一个分区
![](img/spark_join.jpg)