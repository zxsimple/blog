> 网上对Tomcat的优化略多，但大部分只是从某方面的优化入手，没有一个全面的优化实践。由于对Tomcat的优化还在持续，本文也会将随时更新，同时择时添加一些性能方面的对比做参考。 此处我尝试把Tomcat优化方方面面的内容都整理到一起。

### 以Server方式启动

默认的Tomcat配置只适用开发场景，而默认Tomcat的启动方式居然不是以Server方式启动，在JVM参数中指定`-server`参数 对于server mode和client mode的对比，可以参考[So You Want High Performance](https://tomcat.apache.org/articles/performance.pdf)

而Tomcat官网却对如此重要的参数并未说明(或许是我没找到)。

> For performance reasons, it is recommended that the application is run in Server mode. Apache Tomcat does not run in Server mode by default. You can set the Server mode by using the JVM **-server** parameter. You can set the JVM parameter in the JAVA_OPTS variable in the environment variable in the setenv file.

### Tomcat Connector的I/O模型

`Tomcat`的`Connector`通常有三种I/O模型，BIO阻塞式I/O，NIO非阻塞式I/O和ARP，前面两个都知道，关于ARP(Apache Runtime Portable)它通过调用操作系统的Native库来处理I/O，因此在I/O密集型的场景下有更好的性能表现。关于三种I/O模式的介绍可以参考[修改Tomcat Connector运行模式，优化Tomcat运行性能](http://www.365mini.com/page/tomcat-connector-mode.htm)和[Tomcat Connector三种运行模式（BIO, NIO, APR）的比较和优化](http://www.fwqtg.net/tomcat-connector三种运行模式（bio-nio-apr）的比较和优化.html)

此处介绍一下ARP的启用过程。

**编译安装ARP库**

```bash
cd /home/hadoop/software
wget http://apache.fayea.com/apr/apr-1.5.2.tar.gz
wget http://apache.fayea.com/apr/apr-util-1.5.4.tar.gz
tar -zxvf apr-1.5.2.tar.gz
tar -zxvf apr-util-1.5.4.tar.gz
cd apr-1.5.2
./configure
make && make install
cd ..
cd apr-util-1.5.4
./configure --prefix=/usr/local/apr --with-apr=/usr/local/apr --with-java-home=/usr/java/jdk1.7.0_67-cloudera
make && make install
```

**安装Tomcat Native库**

```bash
cd /home/hadoop/apache-tomcat-8.0.24/bin
tar zxvf tomcat-native.tar.gz
cd tomcat-native-1.1.33-src/jni/native
./configure --with-apr=/usr/local/apr --with-java-home=/usr/java/jdk1.7.0_67-cloudera
make && make install
```

**修改Connector的Protocol**

```xml
<Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
```

### JVM参数调优

**-Xms**

初始堆大小默认(`MinHeapFreeRatio`参数可以调整)空余堆内存小于40%时，JVM就会增大堆直到`-Xmx`的最大限制.

**-Xmx**

最大堆大小默认(`MaxHeapFreeRatio`参数可以调整)空余堆内存大于70%时，JVM会减少堆直到 `-Xms`的最小限制

**–Xmn**

设置年轻代大小为512m。整个堆大小=年轻代大小 + 年老代大小 + 持久代大小。持久代一般固定大小为64m，所以增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。

**-Xss**

是指设定每个线程的堆栈大小。这个就要依据你的程序，看一个线程 大约需要占用多少内存，可能会有多少线程同时运行等。一般不易设置超过1M，要不然容易出现out ofmemory。

**-XX:+AggressiveOpts**

作用如其名（aggressive），启用这个参数，则每当JDK版本升级时，你的JVM都会使用最新加入的优化技术（如果有的话）

**-XX:+UseBiasedLocking**

启用一个优化了的线程锁，我们知道在我们的appserver，每个http请求就是一个线程，有的请求短有的请求长，就会有请求排队的现象，甚至还会出现线程阻塞，这个优化了的线程锁使得你的appserver内对线程处理自动进行最优调配。

**-XX:PermSize=128M-XX:MaxPermSize=256M**

JVM使用`-XX:PermSize`设置非堆内存初始值，默认是物理内存的1/64； 在数据量的很大的文件导出时，一定要把这两个值设置上，否则会出现内存溢出的错误。 由`-XX:MaxPermSize`设置最大非堆内存的大小，默认是物理内存的1/4。 那么，如果是物理内存4GB，那么64分之一就是64MB，这就是PermSize默认值，也就是永生代内存初始大小； 四分之一是1024MB，这就是MaxPermSize默认大小。

**-XX:+DisableExplicitGC**

在程序代码中不允许有显示的调用`System.gc()`。看到过有两个极品工程中每次在DAO操作结束时手动调用`System.gc()`一下，觉得这样做好像能够解决它们的out ofmemory问题一样，付出的代价就是系统响应时间严重降低，就和我在关于`Xms`，`Xmx`里的解释的原理一样，这样去调用GC导致系统的JVM大起大落，性能不到什么地方去哟！

**-XX:+UseParNewGC**

此处使用并行回收器，回收器的选取跟业务有很大的关系，建议高吞吐量场景下使用并行收集器回收垃圾，对于快速响应场景采用并发收集器收集垃圾。

**-XX:MaxTenuringThreshold**

设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概率。 这个值的设置是根据本地的jprofiler监控后得到的一个理想的值，不能一概而论原搬照抄。

**-XX:+CMSParallelRemarkEnabled**

在使用`UseParNewGC`的情况下, 尽量减少 mark 的时间

**-XX:+UseCMSCompactAtFullCollection**

在使用concurrent gc 的情况下, 防止 memoryfragmention, 对live object 进行整理, 使 memory 碎片减少。

**-XX:LargePageSizeInBytes**

指定 Java heap的分页页面大小

**-XX:+UseFastAccessorMethods**

get,set 方法转成本地代码

**-XX:+UseCMSInitiatingOccupancyOnly**

指示只有在 oldgeneration 在使用了初始化的比例后concurrent collector 启动收集

**-XX:CMSInitiatingOccupancyFraction=70**

`CMSInitiatingOccupancyFraction`，这个参数设置有很大技巧，基本上满足`(Xmx-Xmn)*(100- CMSInitiatingOccupancyFraction)/100>=Xmn`就不会出现promotion failed。在我的应用中Xmx是6000，Xmn是512，那么`Xmx-Xmn`是5488兆，也就是年老代有5488 兆，`CMSInitiatingOccupancyFraction=90`说明年老代到90%满的时候开始执行对年老代的并发垃圾回收（CMS），这时还 剩10%的空间是5488*10%=548兆，所以即使Xmn（也就是年轻代共512兆）里所有对象都搬到年老代里，548兆的空间也足够了，所以只要满 足上面的公式，就不会出现垃圾回收时的promotion failed； 因此这个参数的设置必须与Xmn关联在一起。

**-Djava.awt.headless=true**

这个参数一般我们都是放在最后使用的，这全参数的作用是这样的，有时我们会在我们的J2EE工程中使用一些图表工具如：jfreechart，用于在web网页输出GIF/JPG等流，在winodws环境下，一般我们的app server在输出图形时不会碰到什么问题，但是在linux/unix环境下经常会碰到一个exception导致你在winodws开发环境下图片显示的好好可是在linux/unix下却显示不出来，因此加上这个参数以免避这样的情况出现。

### Connector优化

**maxConnections**

最大的连接数，超过此连接数的请求会被接受但是不会被处理

**maxThreads**

最大处理线程

**minSpareThreads**

初始化时创建的线程数

**maxSpareThreads**

一旦创建的线程超过这个值，Tomcat就会关闭不再需要的socket线程。

**acceptCount**

指定当所有可以使用的处理请求的线程数都被使用时，可以放到处理队列中的请求数，超过这个数的请求将不予处理

**maxHttpHeaderSize**

http header的最大值

**enableLookups**

当web应用程序向要记录客户端的信息时，它也会记录客户端的IP地址或者通过域名服务器查找机器名 转换为IP地址。 DNS查询需要占用网络，并且包括可能从很多很远的服务器或者不起作用的服务器上去获取对应的IP的过程，这样会消耗一定的时间。

> 在有些情况下Tomcat负载并不高，而又有大量WAIT线程时，很有可能是因为acceptCount设置过小。
>
> 如果Tomcat中定义了多个Connector，可以定义Executor线程池，各个Connector共享线程池。