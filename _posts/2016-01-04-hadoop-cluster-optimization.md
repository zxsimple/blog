---
ID: 8
post_title: Hadoop集群性能调优
author: getitsimple
post_excerpt: ""
layout: post
permalink: >
  http://hua-fei.com.cn/index.php/2016/01/04/hadoop-cluster-optimization/
published: true
post_date: 2016-01-04 13:48:38
---
<strong>Hadoop</strong><strong>集群性能调优</strong>
<h1>mapred-site.xml</h1>
<strong>mapred.child.java.opts</strong>

这 个参数是配置每个map或reduce使用的内存数量。默认的是200M。对于这个参数，我个人认为，如果内存是8G，CPU有8个核，那么就设置成1G 就可以了。实际上，在map和reduce的过程中对内存的消耗并不大，但是如果配置的太小，则有可能出现”无可分配内存”的错误。所以，对于这个配置我 总结了一个简单的公式：map/reduce的并发数量(总和不大于CPU核数)×mapred.child.java.opts &lt; 该节点机器的总内存。

Sanders: 总内存 / (Map/Reduce 并发数) * 80%

-Xmx512m

<strong>dfs.block.size </strong>

一般来说，块的大小也决定了你map的数量。block size的大小是需要根据输入文件的大小以及计算是产生的map来综合考量的。一般来说，文件大，集群数量少，还是建议将block size设置大一些的好。

<strong>disable ipv6</strong><strong>配置</strong>

某些 Hadoop 版本在配置了 IPv6 的计算机上会监听错网络地址

修改bin/hadoop，添加下行：

HADOOP_OPTS="$HADOOP_OPTS -Djava.net.preferIPv4Stack=true"
<h1>Mapper</h1>
<strong>mapred.compress.map.output </strong>

几乎所有会产生大量map output的hadoop作业，都能够通过将中间文件用Lzo进行压缩来提升性能。尽管lzo会使得cpu的load有所增高，但是reduce在shuffle阶段的disk IO通常都能节省不少时间，对job的性能有好处。

当 采用map中间结果压缩的情况下，用户还可以选择压缩时采用哪种压缩格式进行压缩，现在hadoop支持的压缩格式 有：GzipCodec，LzoCodec，BZip2Codec，LzmaCodec等压缩格式。通常来说，想要达到比较平衡的cpu和磁盘压缩 比，LzoCodec比较适合。但也要取决于job的具体情况。

&lt;property&gt;
&lt;name&gt;mapred.compress.map.output&lt;/name&gt;
&lt;value&gt;true&lt;/value&gt;
&lt;/property&gt;

&lt;property&gt;
&lt;name&gt;mapred.map.output.compression.codec&lt;/name&gt;
&lt;value&gt;com.hadoop.compression.lzo.LzoCodec&lt;/value&gt;
&lt;/property&gt;

<strong>mapred.tasktracker.map.tasks.maximum</strong>

The maximum number of map tasks that will be run simultaneously by a task tracker.

一个tasktracker最多可以同时运行的map任务数量

默认值：2

mapred.tasktracker.map.tasks.maximum = cpu数量

Sanders: 应该设置成为等于或略小于CPU总核数

cpu数量 = 服务器CPU总核数 / 每个CPU的核数
服务器CPU总核数 = more /proc/cpuinfo | grep 'processor' | wc -l
每个CPU的核数 = more /proc/cpuinfo | grep 'cpu cores'

<strong>mapred.map.tasks</strong>

The default number of map tasks per job

一个Job会使用task tracker的map任务槽数量，这个值 ≤ mapred.tasktracker.map.tasks.maximum

默认值：2

优化值：

CPU数量 （我们目前的实践值）

(CPU数量 &gt; 2) ? (CPU数量 * 0.75) : 1 （mapr的官方建议）

<strong>io.sort.mb </strong>

每 一个map都会对应存在一个内存 buffer， map会将已经产生的部分结果先写入到该buffer中，这个buffer默认是100MB大小. 当map的产生数据非常大时，并且把io.sort.mb调 大，那么map在整个计算过程中spill的次数就势必会降低，map task对磁盘的操作就会变少，如果map tasks的瓶颈在磁盘上，这样调整就会大大提高map的计算性能。

Tune io.sort.mb value until you observe that the number of spilled records equals (or is as close to equals) the number of map output records.

优化值:

逐渐增大io.sort.mb, 直到map的输出个数接近于spill数量

<strong>io.sort.spill.percent </strong>

这 个值就是上述buffer的阈值，默认是0.8，既80%，当buffer中的数据达到这个阈值，后台线程会起来对buffer中已有的数据进行排序，然 后写入磁盘，此时map输出的数据继续往剩余的20% buffer写数据，如果buffer的剩余20%写满，排序还没结束，map task被block等待。

<strong>io.sort.factor</strong>

当一个map task执行完之后，本地磁盘上(mapred.local.dir)有若干个spill文件，map task最后做的一件事就是执行merge sort，把这些spill文件合成一个文件（partition），有时候我们会自定义partition函数，就是在这个时候被调用的。

当map的中间结果非常大，调大io.sort.factor，有利于减少merge次数，进而减少 map对磁盘的读写频率，有可能达到优化作业的目的。

默认是10

需要看spill的数量, 综合不同map的结果来调整
<h1>Reducer</h1>
<strong>mapred.tasktracker.reduce.tasks.maximum</strong>

The maximum number of reduce tasks that will be run simultaneously by a task tracker.

一个task tracker最多可以同时运行的reduce任务数量

默认值：2

优化值： CPU核数

如果使用fair的调度模式，设置成相同，应该是可以的，但是如果是FIFO模式，我个人认为在map或是reduce阶段，CPU的核数没有得到充分的利用，有些可惜，所以，FIFO模式下，还是尽量配置的map并发数量多于redcue并发数量

<strong>mapred.reduce.tasks</strong>

The default number of reduce tasks per job. Typically set to 99% of the cluster's reduce capacity, so that if a node fails the reduces can still be executed in a single wave.

一个Job会使用task tracker的reduce任务槽数量

默认值：1

优化值：
<ol>
	<li>95 * nodes * mapred.tasktracker.tasks.maximum</li>
</ol>
理由：启用95%的reduce任务槽运行task, recude task运行一轮就可以完成。剩余5%的任务槽永远失败任务，重新执行
<ol start="2">
	<li>75 * nodes * mapred.tasktracker.tasks.maximum</li>
</ol>
理由：因为reduce task数量超过reduce槽数，所以需要两轮才能完成所有reduce task。

对于三台机器每个16核来说，mapred.reduce.tasks的个数应该在0.95*3*16之间1.75*3*16

<strong>mapred.reduce.parallel.copies</strong>

Reduce copy数据的线程数量，默认值是5

Reduce task在做shuffle时，实际上就是从不同的已经完成的map上去下载属于自己这个reduce的部分数据，由于map通常有许多个，所以对一个reduce来说，下载也可以是并行的从多个map下载

Reduce 到每个完成的Map Task copy数据（通过RPC调用），默认同时启动5个线程到map节点取数据。这个配置还是很关键的，如果你的map输出数据很大，有时候会发现map早就 100%了，reduce一直在1% 2%。。。。。。缓慢的变化，那就是copy数据太慢了，5个线程copy 10G的数据，确实会很慢，这时就要调整这个参数了，但是调整的太大，又会事半功倍，容易造成集群拥堵，所以 Job tuning的同时，也是个权衡的过程，你要熟悉你的数据

<strong>mapred.job.shuffle.input.buffer.percent</strong>

Reduce 在shuffle阶段对下载来的map数据，并不是立刻就写入磁盘的，而是会先缓存在内存中，然后当使用内存达到一定量的时候才刷入磁盘。当指定了JVM 的堆内存最大值以后，上面这个配置项就是Reduce用来存放从Map节点取过来的数据所用的内存占堆内存的比例，默认是70%

<strong>mapred.job.shuffle.merge.percent </strong>

同Map的io.sort.spill.percent 一样，当shuffle时不是等到buffer满后再在flush到磁盘，而是达到一定阈值后写入磁盘，其默认值是0.66

从各个Map输出端通过网络下载的结果需要写入磁盘, 在高速网络中写磁盘速度有可能block下载速度, 所以这个指标是网络速度和磁盘速度还有数据量的平衡.

<strong>mapred.job.reduce.input.buffer.percent </strong>

当reduce task真正进入reduce函数的计算阶段的时候，在读取reduce需要的数据时需要多少的内存百分比来作为reduce读已经sort好的数据的 buffer百分比。默认情况下为0，也就是说，默认情况下，reduce是全部从磁盘开始读 处理数据。如果这个参数大于0，那么就会有一定量的数据被缓存在内存并输送给reduce，当reduce计算逻辑消耗内存很小时，可以分一部分内存用来 缓存数据

<strong>mapred.inmem.merge.threshold</strong>

从map节点取过来的文件个数，当达到这个个数之后，也进行merger sort，然后写到reduce节点的本地磁盘；mapred.job.shuffle.merge.percent优先判断.默认值1000，完全取决于map输出数据的大小

默认值1000