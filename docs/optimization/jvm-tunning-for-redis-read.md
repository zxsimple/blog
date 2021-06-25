### 问题描述

多线程并发读取Redis数据并写入到本地文件，开始写入文件速度很快，到一定阶段后(7G左右)写入速度急剧变慢，CPU 100%。最终报`java.lang.OutOfMemoryError: GC overhead limit exceeded` GC失败，程序退出。

### 问题分析

通过`jstat -gcutil [pid] [interval] [times]`打印GC的情况，发现在程序hang住时Eden和Old区使用为100%，FGC(Full GC)频繁发生。

![](_images\jstat.png)

查看程序代码发现在调用jdeis读取Redis时会生成`String`对象，由于是高并发读取Redis，因此Eden区生成大量的生命周期极短的对象，YGC来不及回收，而FGC频发。这时处理GC花费的时间超过 **98%**, 并且GC回收的内存少于 **2%**，就会产生上述错误。

```java
public String get(String key)
{
    checkIsInMultiOrPipeline();
    this.client.sendCommand(Protocol.Command.GET, new String[] { key });
    return this.client.getBulkReply();
}

public String getBulkReply()
{
    byte[] result = getBinaryBulkReply();
    if (null != result) {
      return SafeEncoder.encode(result);
    }
    return null;
}

public static String encode(byte[] data)
{
    try
    {
      return new String(data, "UTF-8");
    }
    catch (UnsupportedEncodingException e)
    {
      throw new JedisException(e);
    }
}
```

### 解决过程

**修改读取Redis数据方式**

修改读取Redis数据方式，避免产生String对象。这个办法在JVM参数调整前，开始没有效果，JVM调整后未做尝试。

```java
Charset cs = Charset.forName("UTF-8");
cs.decode(ByteBuffer.wrap(op.get(key.getBytes("UTF-8")))) // 传入byte[] key，获取byte[]值，然后解码为String
```
**修改JVM参数**

首先增加`-XX:UseGCOverheadLimit`取消GC忙碌限制，这只是一个掩耳盗铃的做法，解决不了问题。

增大CGTimeRatio，设置`-XX:GCTimeRatio=9`，给GC更多的时间来做垃圾回收。

> **吞吐量：**吞吐量为垃圾回收时间与非垃圾回收时间的比值，通过-XX:GCTimeRatio=<N>来设定，公式为1/（1+N）。例如，-XX:GCTimeRatio=19时，表示5%的时间用于垃圾回收。默认情况为99，即1%的时间用于垃圾回收。 

使用并行垃圾回收器`-XX:+UseParallelGC`，同时增加并行回收的线程数`-XX:ParallelGCThreads=16`。

增加JVM内存大小`-Xmx64g`，同时增加新生代空间大小`-Xmn32g`。

再启动程序发现YGC的数量基本在FGC的两倍以上，Eden和Old代虽然占用率很高但是GC能够正常工作。程序也可以正常执行。

**最终JVM参数**

```bash
-Xms32g -Xmx64g -Xmn32g -XX:PermSize=512m -XX:MaxPermSize=1024m -XX:+UseParallelGC -XX:ParallelGCThreads=16 -XX:-UseGCOverheadLimit -XX:GCTimeRation=9
```
