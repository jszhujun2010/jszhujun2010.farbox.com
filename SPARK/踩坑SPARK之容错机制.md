---
date: 2018-04-15 12:38
status: public
title: 踩坑SPARK之容错机制
---

[TOC]

# SPARK的容错

这块机制其实还不是太明白，很多都是看的[这位兄弟](http://liyichao.github.io/posts/spark-%E5%AE%B9%E9%94%99%E6%9C%BA%E5%88%B6.html)的博客，这里说说今天遇到的问题以及踩到的坑。

## 问题

最近在调一个spark程序，因为数据量太大，存在一些性能障碍。之前join的问题已经解决（过两天把这个方案补上）。一直以为这样就解决问题了，但通过新数据的测试，发现耗时仍然可能特别严重。一个问题是几个任务的GC时间过长，导致整体运行时间特别长（这个问题没有得到复现，如果再次遇到的话，目前只能通过一些已有GC方案解决）；另一个问题是，即使没有大的GC耗时，计算时间依旧很感人（一个任务大约要4~5h，是不大能接受的）。

为了加速任务，在队列资源不是特别紧张的前提下，我决定加一些机器。具体做法就是加num-executors, executor-memory以及executor-cores，然后把default-parallelism增大。一开始观察，速度的确有加快的样子，估计可以提速一倍（也是应该的，毕竟资源也加了一倍左右）。但是当任务跑到1/4的时候，突然出现了一个意外：一个executor突然挂了！

## SPARK的任务重启

之前没有仔细研究过，executor挂了spark会怎么处理。因为毕竟已经跑了一些结果了，总不能从头开始再重跑一遍吧。

首先我们来看一下driver日志报的问题：

```text
FetchFailed(BlockManagerId(301, some_port, 7391, None), shuffleId=4, mapId=69, reduceId=1579, message=org.apache.spark.shuffle.FetchFailedException: Failed to connect to some_port
```

原来是某个exetutor想要fetch数据（应该是shuffle read），但那个有数据的executor挂了，导致fetch失败。为啥我知道executor挂了？我是通过spark-ui看到的。

我们想一下，那个executor挂了的话，有什么后果？那个executor上存着上个stage算好的数据，然后这个stage的任务会依赖那些数据，所以这个会影响到很多这个stage的任务。

下面分三个角度看这个stage的任务：这个stage已经算好的任务，应该是不需要重新计算的；这个stage未启动的任务暂时不受到影响；这个stage已经启动但未完成的任务是什么影响呢？这个我们稍后再说。

先看spark在知道executor挂了之后做了些什么事情？假设我们当前的stage是9号stage，默认叫做9.0；现在因为那个executor挂了，这个stage不能顺利继续下去了。所以，spark就重启一个新的stage，叫做9.1。由于已经算好的就不要算了，所以任务数量就是之前的总量减去已经计算完成的数量。对于9.0已经启动但未完成的任务，9.1仍然会重启，但似乎二者之前没有进行沟通。

下面来看，那个executor已经算好的数据现在丢失了，spark要怎么做？由于spark的rdd之间的“血缘”关系，可以根据那个executor上rdd的生成方法，再重新算一遍就好了。这个只涉及到那个executor上的数据，所以开销会很小，但有可能会重新计算多个stage（我遇到的是分钟级别的重新计算）。

讲道理，有了这个容错重启任务的机制，分钟级别的重算不会带来很大的额外时间开销的。但通过spark ui观察，在那个9.0的时候，并行的任务有近1000个，现在9.1的并行任务只剩下300~400个，速度变得很慢，加了资源就跟没加一样，这怎么能忍？

回头看一下executor的log，积累了一段时间发现，很多executor一直在报这个：

```text
java.io.IOException: Connection from some_port closed
18/04/15 08:15:09 INFO RetryingBlockFetcher: Retrying fetch (1/30) for 20 outstanding blocks after 10000 ms
18/04/15 08:15:09 ERROR OneForOneBlockFetcher: Failed while starting block fetches
java.io.IOException: Connection from some_port closed
18/04/15 08:15:09 INFO RetryingBlockFetcher: Retrying fetch (1/30) for 20 outstanding blocks after 10000 ms
18/04/15 08:15:09 ERROR OneForOneBlockFetcher: Failed while starting block fetches
java.io.IOException: Connection from some_port closed
18/04/15 08:15:09 INFO RetryingBlockFetcher: Retrying fetch (1/30) for 20 outstanding blocks after 10000 ms
18/04/15 08:15:19 INFO TransportClientFactory: Found inactive connection to some_port, creating a new one.
18/04/15 08:17:26 INFO TransportClientFactory: Found inactive connection to some_port, creating a new one.
18/04/15 08:17:26 ERROR RetryingBlockFetcher: Exception while beginning fetch of 20 outstanding blocks (after 1 retries)
```

闲着无聊的我，就这样看了两个小时，这个错才消停。仔细观察，好像这个在尝试30次fetch数据。不过fetch的数据源就是那个已经挂了的executor，既然已经挂了，还一直在那儿尝试，不是有毛病嘛。

另外一个问题就是，为毛要重试30次这么多？感觉这应该是个配置，那就在spark ui的environment的tab里面搜一搜30。哈哈，果然有个30个在那儿：

```text
spark.shuffle.io.maxRetries: 30
```

看着名字，应该就是它了！spark一直在尝试fetch那个挂了的executor上的数据，一直要尝试30次！然后还有一个对应的参数：`spark.shuffle.io.retryWait=10s`，这个表示两次retry之间的间隔。知道这个问题后，查了一下文档，发现官方默认的retry次数是3次，不知道哪个运维把默认参数改成了30！还应该挨千刀的是，retryWait也从默认的5s改成了10s。跑得慢的原因很明显了，就是这两个参数，导致很多executor在进行无谓的挣扎，想要从一个挂了的executor上取数，也就是两个小时，一半以上的executor的资源都浪费了。

但是稍等，讲道理30次乘以10s，最多也就浪费300s，也就是5min，怎么会浪费2h呢？这里我猜想：

```text
18/04/15 08:15:19 INFO TransportClientFactory: Found inactive connection to some_port, creating a new one.
```

这里应该是一个提示，当我发现那个executor没法连接上的时候，就想着重新建立一个连接。但毕竟那个节点已经挂了，必然一直没有回应，那就需要等待连接超时。连接超时时间很长，比如是5min，那算下来这个时间也就差不多要两个小时了。

## SPARK 2.1.0的坑

那么问题又来了，spark怎么这么傻？明明那个exector挂了，还是要做尝试。难道driver不能告知每个executor：那个挂了，不要去那里取数了，已经起了的任务就结束吧。通过查询多方资料才知道，原来早先设计的人好像没有考虑到这一点。下面是一些jira和github的issue，都是在吐槽这个问题：

```text
https://issues.apache.org/jira/browse/SPARK-20178
https://issues.apache.org/jira/browse/SPARK-20230
https://github.com/apache/spark/pull/17088
```

看样子是17年5月才close掉这个问题，所以可能要到spark 2.3.0里面才会修复这个问题。也算是踩了一次2.1.0版本spark的坑，不过公司里spark是不能随意升级的，以后还是需要手动处理这个问题，比如自己设置retry次数。不过一个可能的坑是，这个参数的描述是：

```text
This retry logic helps stabilize large shuffles in the face of long GC pauses or transient network connectivity issues.
```

所以GC如果是个问题的话，可能还要往上调。

另外一个收获就是知道了(Netty only)在文档里的意思，这个网络通信库看来我们用得很随意呀。