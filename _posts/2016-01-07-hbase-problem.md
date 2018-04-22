---
ID: 16
post_title: HBase问题汇总
author: getitsimple
post_excerpt: ""
layout: post
permalink: >
  http://hua-fei.com.cn/index.php/2016/01/07/hbase-problem/
published: true
post_date: 2016-01-07 18:10:05
---
<h2>1.    Spring管理HBase链接，停止Tomcat时报错</h2>
Tag：Spring, HBase

问题：
<pre class="brush:java">
org.apache.hadoop.hbase.client.HConnectionManager - Connection not found in the list, 
can't delete it (connection key=HConnectionKey{properties={hbase.rpc.timeout=60000, hbase.zookeeper.property.clientPort=2181, hbase.client.pause=100, 
zookeeper.znode.parent=/hbase, hbase.client.retries.number=35, 
hbase.zookeeper.quorum=192.168.30.28, 192.168.30.19, 192.168.30.11,192.168.30.27,192.168.30.29}, username='hadoop'}). May be the key was modified?
java.lang.Exception
	at org.apache.hadoop.hbase.client.HConnectionManager.deleteConnection(HConnectionManager.java:466)
	at org.apache.hadoop.hbase.client.HConnectionManager.deleteConnection(HConnectionManager.java:402)
	at org.springframework.data.hadoop.hbase.HbaseConfigurationFactoryBean.destroy(HbaseConfigurationFactoryBean.java:80)
	at org.springframework.beans.factory.support.DisposableBeanAdapter.destroy(DisposableBeanAdapter.java:238)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.destroyBean(DefaultSingletonBeanRegistry.java:510)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.destroySingleton(DefaultSingletonBeanRegistry.java:486)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.destroySingleton(DefaultListableBeanFactory.java:740)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.destroySingletons(DefaultSingletonBeanRegistry.java:455)
	at org.springframework.context.support.AbstractApplicationContext.destroyBeans(AbstractApplicationContext.java:1090)
	at org.springframework.context.support.AbstractApplicationContext.doClose(AbstractApplicationContext.java:1064)
	at org.springframework.context.support.AbstractApplicationContext.close(AbstractApplicationContext.java:1010)
	at org.springframework.web.context.ContextLoader.closeWebApplicationContext(ContextLoader.java:558)
	at org.springframework.web.context.ContextLoaderListener.contextDestroyed(ContextLoaderListener.java:143)
	at org.apache.catalina.core.StandardContext.listenerStop(StandardContext.java:5064)
	at org.apache.catalina.core.StandardContext.stopInternal(StandardContext.java:5726)
	at org.apache.catalina.util.LifecycleBase.stop(LifecycleBase.java:232)
	at org.apache.catalina.core.ContainerBase$StopChild.call(ContainerBase.java:1590)
	at org.apache.catalina.core.ContainerBase$StopChild.call(ContainerBase.java:1579)
	at java.util.concurrent.FutureTask.run(FutureTask.java:262)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)

</pre>

解决
<table style="height: 47px;" width="734">
<tbody>
<tr>
<td width="553">&lt;<strong>hdp</strong><strong>:hbase-configuration </strong><strong>configuration-ref</strong><strong>="hadoopConfiguration" </strong><strong>delete-connection</strong><strong>="false" </strong>/&gt;</td>
</tr>
</tbody>
</table>
<h2>2.    HBase BulkLoad时不能读取partition list文件</h2>
<img class="alignnone size-full wp-image-17" src="http://ec2-54-169-247-55.ap-southeast-1.compute.amazonaws.com/blog/wp-content/uploads/2016/01/hbase-bulk-load.png" alt="hbase-bulk-load" width="1403" height="380" />

因为ImportTsv时间需要在Cluster执行，但是实际是在Local执行，

This program needs to be run in Distributed Mode

<a href="http://hbase.apache.org/book.html#trouble.mapreduce">http://hbase.apache.org/book.html#trouble.mapreduce</a>

&nbsp;

To solve this problem, you should run your MR job with your HADOOP_CLASSPATH set to include the HBase dependencies. The "hbase classpath" utility can be used to do this easily. For example (substitute VERSION with your HBase version):

<h2>HBase做快照时由于timeout失败</h2>
在执行snapshot时莫名失败，没有很明显的错误
<pre class="brush:java">
ERROR Java::OrgApacheHadoopHbaseIpc::RemoteWithExtrasException: org.apache.hadoop.hbase.snapshot.HBaseSnapshotException: Snapshot { ss=group_member_uid_tmp-snapshot-20160108 table=group_member_uid_tmp type=FLUSH } had an error. Procedure group_member_uid_tmp-snapshot-20160108 { waiting=[dn1-5-16-j751132-ha-da.juanpi.com,60020,1450775385933] done=[] }
at org.apache.hadoop.hbase.master.snapshot.SnapshotManager.isSnapshotDone(SnapshotManager.java:350)
</pre>
去HMaster的日志和RegionServer的日志看了一遍看到snapshot被abort并在多个RegionServer上失败

<pre class="brush:java">
Received abort on procedure with no local subprocedure dim_user_ip_tmp-snapshot-20151229, ignoring it.
org.apache.hadoop.hbase.errorhandling.ForeignException$ProxyThrowable via dn12-30-28-2ND7Z42.juanpi.com,60020,1450775422501:org.apache.hadoop.hbase.errorhandling.ForeignException$ProxyThrowable: java.io.IOException: org.apache.zookeeper.KeeperException$NoNodeException: KeeperErrorCode = NoNode for /hbase/online-snapshot/acquired/dim_user_ip_tmp-snapshot-20151229/dn12-30-28-2ND7Z42.juanpi.com,60020,1450775422501
</pre>

再在Zookeeper上一看，确实发现是执行Snapshot超时，HBase自动关闭了此操作
<pre class="brush:java">
EndOfStreamException: Unable to read additional data from client sessionid 0x650680df9d34ef1, likely client has closed socket
	at org.apache.zookeeper.server.NIOServerCnxn.doIO(NIOServerCnxn.java:220)
	at org.apache.zookeeper.server.NIOServerCnxnFactory.run(NIOServerCnxnFactory.java:208)
	at java.lang.Thread.run(Thread.java:745)
</pre>

默认HBase生成Snapshot的时间都是60s，如果网络有延时，这个时间就有可能会不够，需要调整。
<pre class="brush:xml">
<property>
    <name>hbase.snapshot.master.timeoutMillis</name>
    <!-- Change from default of 60s to 600s to allow for slow flushing of tables -->
    <value>600000</value>
    <description>
      This is the time HBase master waits for the snapshot operation to complete.
      Do not confuse this hbase.snapshot.master.timeout.millis, which although
      sounding similar, serves a very different purpose.
      Note: This property has a completely different meaning before hbase version
      0.94.11 and should not enabled on a cluster using snapshots and running
      a version before 0.94.11.
    </description>
  </property>
  <property>
    <name>hbase.snapshot.master.timeout.millis</name>
    <!-- Change from default of 60s to 600s to allow for slow flushing of tables -->
    <value>600000</value>
    <description>
      This is the timeout the master indicates the client to wait when it takes
      the snapshot. The client actually waits longer than this due to exponential
      backoff. See HBaseAdmin.snapshot for the exact algorithm.
    </description>
  </property>
  <property>
    <name>hbase.snapshot.region.timeout</name>
    <!-- Change from default of 60s to 600s to allow for slow flushing of tables -->
    <value>600000</value>
    <description>
      This is the time the regionserver waits to complete all of its activities
      for a snapshot operation.
    </description>
 </property>
</pre>