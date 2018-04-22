---
ID: 58
post_title: HBase跨集群数据迁移
author: getitsimple
post_excerpt: ""
layout: post
permalink: >
  http://hua-fei.com.cn/index.php/2016/02/24/hbase-migration-between-clusters/
published: true
post_date: 2016-02-24 11:24:41
---
当HBase集群和Hive集群耦合在一起时，Hive的批量作业运行可能会导致HBase的访问不稳定。正常情况下一个HBase的Get操作可以在10ms内返回，而在集群繁忙时段这个可能会上升到几十秒，前端业务调用将会因此timeout。为了让HBase更稳定地支撑业务，我们采用的是读写分离的方案。即一个集群通过Hive产生数据，而数据结果是以HFile方式存储(原来采用<a href="https://cwiki.apache.org/confluence/display/Hive/HBaseIntegration">Hive-HBase集成表</a>的方式)，将此HFile加载到写集群的HBase表中，然后在写集群中生成HBase表的快照Snapshot，通过HBase的ExportSnapshot工具实现HBase数据在集群之间的迁移。

此处重点吐槽一下Hive-HBase集成表方案，此方案唯一的好处就是在Hive计算和HBase读取的场景下相对比较方便，而它却引入了很多问题，诸如字段变化后不易扩展，Hive写操作验证影响HBase的读响应。

方案的过程是：
1. 将Hive表以HFile方式存储，并映射到写集群的HBase表上
2. 在写集群上创建HBase表的快照Snapshot
3. 通过org.apache.hadoop.hbase.snapshot.ExportSnapshot工具将Snapshot导入到读集群 (重点注意‘<strong>export HADOOP_USER_NAME=hbase</strong>’ 权限问题很麻烦)
4. 在读集群上通过快照还原HBase表

这个方案通过HBase的Snapshot技术实现数据的迁移，基本上可以在秒级内完成，而整个过程HBase的抖动也是非常轻微的。

需要注意的是除上面提到的HDFS 用户权限外，拷贝数据时的网络带宽占用和Snapshot处理过程中的Timeout也是需要考虑的。带宽可以由ExportSnapshot的bandwidth和mapper参数控制，timeout需要修改hbase-site.xml来控制。
<pre class="brush:xml">
<property>
        <name>hbase.snapshot.master.timeoutMillis</name>
        <value>600000</value>
</property>
<property>
        <name>hbase.snapshot.master.timeout.millis</name>
        <value>600000</value>
</property>
<property>
        <name>hbase.snapshot.region.timeout</name>
        <value>600000</value>
</property>
</pre>

整个过程的Shell如下：

<pre class="brush:bash">
#!/bin/bash

hive_table_name=dm.group_member_uid
hbase_table_name=group_member_uid
tmp_hbase_table_name=group_member_uid_tmp
key_column=user_group_id
column_family=user
today=`date '+%Y%m%d'`
yesterday=`date '+%Y%m%d' -d '-1day'`
hdfsIp1=192.168.30.61
hdfsIp2=192.168.30.60
hmaster='192.168.30.41'

THIS="$0"
THIS_DIR=`dirname "$THIS"`
cd ${THIS_DIR}


# dev environment
# export HADOOP_CLASSPATH=/home/hadoop/users/jiangnan/project/hbasebulkload:/usr/lib/hbase/*:/usr/lib/hbase/lib/*:/etc/hbase/conf

# prod environment
export HADOOP_CLASSPATH=/home/hadoop/jiangnan/project/hbasebulkload:/usr/lib/hbase/*:/usr/lib/hbase/lib/*

echo "step 1. bulk load table from hive to hbase"

[ -e checkpoint ]
if [[ $? != 0 ]]; then
	java -server -Dregion.split.size=2 -cp /etc/hive/conf:/etc/hbase/conf:/etc/hadoop/conf:hive2hbase-1.0-SNAPSHOT-jar-with-dependencies.jar \
	    com.juanpi.bi.hive2hbase.HBaseBulkLoad -hivetablename ${hive_table_name} -keycolumn ${key_column} \
	    -hbasetablename ${tmp_hbase_table_name} -familyname ${column_family}
else
        chkpt=`cat checkpoint`
        if [[ $chkpt != $today ]]; then
		java -server -Dregion.split.size=2 -cp /etc/hive/conf:/etc/hbase/conf:/etc/hadoop/conf:hive2hbase-1.0-SNAPSHOT-jar-with-dependencies.jar \
		    com.juanpi.bi.hive2hbase.HBaseBulkLoad -hivetablename ${hive_table_name} -keycolumn ${key_column} \
		    -hbasetablename ${tmp_hbase_table_name} -familyname ${column_family}
        fi
fi

if [[ $? != 0 ]];
then
        echo "step 1 failed"
        exit -1
fi
echo $today > checkpoint

echo "display all snapshots ...find ${tmp_hbase_table_name}-snapshot-${today} "
echo "list_snapshots" | hbase shell | grep ${tmp_hbase_table_name}-snapshot-${today}
if [[ $? == 0 ]];
then
   echo "delete_snapshot ${tmp_hbase_table_name}-snapshot-${today}"
   echo "delete_snapshot '${tmp_hbase_table_name}-snapshot-${today}'" | hbase shell -n
fi

echo "step 2. create snaphot for table ${tmp_hbase_table_name} ... ..."

#
# create snapshot, try 5 time until it success!
#
snapshotFlag=1
for((i=1; i<6; i++))
do 
   echo "snapshot '${tmp_hbase_table_name}', '${tmp_hbase_table_name}-snapshot-${today}'" | hbase shell -n
   if [[ $? == 0 ]];
   then
     snapshotFlag=0
     break
   else
     sleep 60s
   fi
done

if [[ ${snapshotFlag} != 0 ]];
then
   echo "step2 create snapshot failed"
   exit -1
fi

#  test phase, old hbase cluster  
#  snapshot to old hbase cluster
#  
echo "drop old hbase original table ${hbase_table_name}"
echo "disable '${hbase_table_name}'" | hbase shell -n
echo "drop '${hbase_table_name}'" | hbase shell -n

echo "step test phase clone temp snapshot to old table"
echo "clone_snapshot '${tmp_hbase_table_name}-snapshot-${today}', '${hbase_table_name}'" | hbase shell -n
if [[ $? != 0 ]];
then
  echo "step test phase failed"
  exit -1
fi

#
#  judge the remote snapshot exist
#  export to remote hbase cluster 
#
echo "step 3. export snapshot to remote HBase cluster "
ssh ${hmaster} "echo \"list_snapshots\" | hbase shell | grep ${tmp_hbase_table_name}-snapshot-${today}"
if [[ $? == 0 ]];
then
   ssh ${hmaster} "echo \"delete_snapshot '${tmp_hbase_table_name}-snapshot-${today}'\" | hbase shell -n"
fi

export HADOOP_USER_NAME=hbase
hbase --config /etc/hbase/conf org.apache.hadoop.hbase.snapshot.ExportSnapshot -snapshot ${tmp_hbase_table_name}-snapshot-${today} -copy-to hdfs://${hdfsIp1}:8020/hbase -chuser hbase -chgroup hbase -mappers 16

if [[ $? != 0 ]];
then
    hbase --config /etc/hbase/conf org.apache.hadoop.hbase.snapshot.ExportSnapshot -snapshot ${tmp_hbase_table_name}-snapshot-${today} -copy-to hdfs://${hdfsIp2}:8020/hbase -chuser hbase -chgroup hbase -mappers 16 
   if [[ $? != 0 ]];
   then
     echo "step 3 failed"
     exit -1
   fi
fi

echo "step 4. create remote cluster  snapshot "
ssh ${hmaster} "echo \"list_snapshots\" | hbase shell | grep ${hbase_table_name}-snapshot-${today}"
if [[ $? == 0 ]];
then
   ssh ${hmaster} "echo \"delete_snapshot '${hbase_table_name}-snapshot-${today}'\" | hbase shell -n"
fi

ssh ${hmaster} "echo \"snapshot '${hbase_table_name}', '${hbase_table_name}-snapshot-${today}'\" | hbase shell -n"
if [[ $? != 0 ]];
then
        echo "4 remote cluster  snapshot  failed"
        exit -1
fi
echo "step 5.drop new cluster hbase table"
ssh ${hmaster} "echo \"step 5. drop original table ${hbase_table_name}\""
ssh ${hmaster} "echo \"disable '${hbase_table_name}'\" | hbase shell"
ssh ${hmaster} "echo \"drop '${hbase_table_name}'\" | hbase shell"

echo "step 6. clone temp snapshot to new table"
ssh ${hmaster} "echo \"clone_snapshot '${tmp_hbase_table_name}-snapshot-${today}', '${hbase_table_name}'\" | hbase shell -n"
if [[ $? != 0 ]];
then
        echo "step 6 failed"
        exit -1
fi

#
# delete yesterday snapshot
#
echo "step 7. delete yesterday  snapshot"
ssh ${hmaster} "echo \"list_snapshots\" | hbase shell | grep ${tmp_hbase_table_name}-snapshot-${yesterday}"
if [[ $? == 0 ]];
then
   ssh ${hmaster} "echo \"delete_snapshot '${tmp_hbase_table_name}-snapshot-${yesterday}'\" | hbase shell -n"
else
   echo "step 7 new cluster warning yesterday tmp_snapshot not found"
fi
ssh ${hmaster} "echo \"list_snapshots\" | hbase shell | grep ${hbase_table_name}-snapshot-${yesterday}"
if [[ $? == 0 ]];
then
   ssh ${hmaster} "echo \"delete_snapshot '${hbase_table_name}-snapshot-${yesterday}'\" | hbase shell -n"
else
   echo "step 7 new cluster warning yesterday snapshot not found"
fi


echo "list_snapshots" | hbase shell | grep ${tmp_hbase_table_name}-snapshot-${yesterday}
if [[ $? == 0 ]];
then
   echo "delete_snapshot '${tmp_hbase_table_name}-snapshot-${yesterday}'" | hbase shell -n
else
   echo "step 7 old cluster warning yesterday tmp_snapshot not found"
fi

exit 0
</pre>