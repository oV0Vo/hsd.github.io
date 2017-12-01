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
<pre data-type="programlisting" data-code-language="scala"><code class="c1">// Initialization code; we load the user info from a Hadoop SequenceFile on HDFS.</code>
<code class="c1">// This distributes elements of userData by the HDFS block where they are found,</code>
<code class="c1">// and doesn't provide Spark with any way of knowing in which partition a</code>
<code class="c1">// particular UserID is located.</code>
<code class="k">val</code> <code class="n">sc</code> <code class="k">=</code> <code class="k">new</code> <code class="nc">SparkContext</code><code class="o">(...)</code>
<code class="k">val</code> <code class="n">userData</code> <code class="k">=</code> <code class="n">sc</code><code class="o">.</code><code class="n">sequenceFile</code><code class="o">[</code><code class="kt">UserID</code>, <code class="kt">UserInfo</code><code class="o">](</code><code class="s">"hdfs://..."</code><code class="o">).</code><code class="n">persist</code><code class="o">()</code>

<code class="c1">// Function called periodically to process a logfile of events in the past 5 minutes;</code>
<code class="c1">// we assume that this is a SequenceFile containing (UserID, LinkInfo) pairs.</code>
<code class="k">def</code> <code class="n">processNewLogs</code><code class="o">(</code><code class="n">logFileName</code><code class="k">:</code> <code class="kt">String</code><code class="o">)</code> <code class="o">{</code>
<code class="k">val</code> <code class="n">events</code> <code class="k">=</code> <code class="n">sc</code><code class="o">.</code><code class="n">sequenceFile</code><code class="o">[</code><code class="kt">UserID</code>, <code class="kt">LinkInfo</code><code class="o">](</code><code class="n">logFileName</code><code class="o">)</code>
<code class="k">val</code> <code class="n">joined</code> <code class="k">=</code> <code class="n">userData</code><code class="o">.</code><code class="n">join</code><code class="o">(</code><code class="n">events</code><code class="o">)</code><code class="c1">// RDD of (UserID, (UserInfo, LinkInfo)) pairs</code>
<code class="k">val</code> <code class="n">offTopicVisits</code> <code class="k">=</code> <code class="n">joined</code><code class="o">.</code><code class="n">filter</code> <code class="o">{</code>
<code class="k">case</code> <code class="o">(</code><code class="n">userId</code><code class="o">,</code> <code class="o">(</code><code class="n">userInfo</code><code class="o">,</code> <code class="n">linkInfo</code><code class="o">))</code> <code class="k">=&gt;</code> <code class="c1">// Expand the tuple into its components</code>
<code class="o">!</code><code class="n">userInfo</code><code class="o">.</code><code class="n">topics</code><code class="o">.</code><code class="n">contains</code><code class="o">(</code><code class="n">linkInfo</code><code class="o">.</code><code class="n">topic</code><code class="o">)</code>
<code class="o">}.</code><code class="n">count</code><code class="o">()</code>
<code class="n">println</code><code class="o">(</code><code class="s">"Number of visits to non-subscribed topics: "</code> <code class="o">+</code> <code class="n">offTopicVisits</code><code class="o">)</code>
<code class="o">}</code></pre>
&emsp;一图胜千言，假设我们对userData和events进行join操作（userData.join(events))，图中每一个小块对应于RDD的一个分区
![](img/spark_join.jpg)