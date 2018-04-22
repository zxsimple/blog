---
ID: 107
post_title: >
  基于分布式网络日志收集系统(基础篇)
author: getitsimple
post_excerpt: ""
layout: post
permalink: >
  http://hua-fei.com.cn/index.php/2014/02/25/distributed-log-system/
published: true
post_date: 2014-02-25 09:10:07
---
本文旨在介绍如何搭建一个基于分布式的网络日志采集系统，本文将从介绍理论入手，讲解所需技术的理论知识，然后实践此过程。

本文中涉及到的技术产品有Apache HTTP Server，Tomcat和RabbitMQ，会介绍Apache模块，Apache的代理和负载均衡技术；RabbitMQ相关消息处理技术。基于此技术基础将会实践如何利用这些技术搭建分布式网络日志收集系统。
<h3>Apache</h3>
<ul>
	<li>反向代理</li>
	<li>负载均衡</li>
	<li>Hook机制和Apache Module</li>
</ul>
<h3>RabittMQ</h3>
<ul>
	<li>AMPQ协议</li>
	<li>RabbitMQ机制</li>
</ul>
<h1>Apache Hook机制和模块</h1>
Apache的运行有两个主要阶段：启动阶段和运行阶段。
<h3>启动阶段</h3>
Apache Server在整个启动阶段处于一个单进程单线程的环境中。 这个阶段包括配置文件解析(如http.conf文件)、模块加载和系统资源初始化（例如日志文件、共享内存段、 数据库连接等）等工作。
<h3>运行阶段</h3>
Apache主要工作是处理用户的服务请求，Apache对HTTP的请求可以分为连接、处理和断开连接三个大的阶段。同时也可以分为11个小的阶段，依次为： Post-Read-Request，URI Translation，Header Parsing，Access Control，Authentication，Authorization， MIME Type Checking，FixUp，Response，Logging，CleanUp。
<h3>Apache的Hook机制</h3>
Apache的Hook机制是指：Apache在运行阶段允许模块(包括内部模块和外部模块)将自定义的函数注入到请求处理循环中。换句话说，模块可以在 Apache的任何一个处理阶段中挂接(Hook)上自己的处理函数，从而参与Apache的请求处理过程。
<img class="alignnone size-full wp-image-108" src="/wp-content/uploads/2016/02/distributed_web_log_apache_module.png" alt="distributed_web_log_apache_module" width="476" height="300" />
<h3>自定义Apache模块</h3>
<ol>
	<li>生成helloworld模块模板</li>

在此模板中有两个重要的方法helloworld_register_hooks和helloworld_handler
<pre class="brush:bash">sudo /usr/share/apache/bin/apxs -g -n helloworld
</pre>
在Apache加载阶段将通过hook机制执行helloworld_register_hooks函数然后将helloworld_handler注册到到钩子上去。
当Apache接收到请求后将会调用helloworld_handler来处理此请求。

	<li>编译helloworld模块</li>

<pre class="brush:bash">sudo /usr/share/apache/bin/apxs -c mod_helloworld.c
</pre>
	<li>配置模块</li>


然后到/usr/local/apache/conf/httpd.conf 中加入
<pre class="brush:bash">LoadModule helloworld_module modules/mod_helloworld.so
setHandler helloworld
</pre>
</ol>
这时通过访问http://loacalhost/helloworld就会触发helloworld_handler的逻辑。
<h1>Apache代理</h1>
<h3>正向代理</h3>
客户端无法直接访问外网络，而这时有一台服务器它可以访问外部网络，那么其它客户端可以通过此服务器间接地访问外部网络。这种情况下这台服务器称之为代理服务器，客户端知道目标网络的地址，但不能直接访问，通过代理服务其访问目标网络。

例如，我们在公司访问外部网络时是通过代理服务其访问的，在访问允许资源的时候我们是不知道代理服务器的存在，而当我们访问blog等受限资源时代理服务器就会阻止我们的请求。
<h3>反向代理</h3>
客户端不能访问目标网络或者不知道目标网络的地址,目标网络资源所在的网络内一台机器充当代理,客户端直接访问代理，代理通过内置的策略将请求路由给目标网络，在目标资源不明确的情况下，客户端通过直接访问代理来获得相应资源。

例如，我们在访问taobao时，我们只记得www.taobao.com这一个地址，当上百万的用户在访问同一个地址时不可能由同一个后台服务器来处理这些请求，那么反向代理服务器就会将这些请求路由到后台不同的服务器来处理。

这两种代理的区别是前者客户端知道目标资源，而不知道代理的存在；后者客户端只知道代理服务器的存在，而不知道此资源的最终提供者。
<img class="alignnone size-full wp-image-113" src="/wp-content/uploads/2016/02/distributed_web_log_reverse_proxy.png" alt="distributed_web_log_reverse_proxy" width="396" height="305" />
而对于设置apache的反向代理是非常简单的，只要修改ProxyRequest Off就可以
<pre class="brush:bash">sudo vim /etc/apache2/mods-enabled/proxy.conf
sudo vim conf/http.conf
ProxyRequests Off
</pre>
<h1>Apache负载均衡</h1>
将Apache作为LoadBalance前置机分别有三种不同的部署方式，分别是
<ul>
	<li>轮询均衡策略的配置</li>
</ul>
<pre class="brush:bash">ProxyPass / balancer://proxy/
BalancerMember http://172.20.29.64:8080/
BalancerMember http://172.20.29.64:8082/
</pre>
“ProxyPass”是配置代理服务器的命令，“/”代表发送请求的URL前缀，如：http://172.20.29.64/或者http://172.20.29.64/ies，这些URL都将符合上述过滤条件。“balancer://proxy/”表示要配置负载均衡，proxy代表负载均衡名。BalancerMember 及其后面的URL表示被代理的服务器及被代理服务器请求URL。

当客户端请求到来时在此配置策略下，请求将交替发送给两个BalancerMember。
<ul>
	<li>按权重分配均衡策略的配置</li>
</ul>
<pre class="brush:bash">ProxyPass / balancer://proxy/
BalancerMember http://172.20.29.64:8080/ loadfactor=3
BalancerMember http://172.20.29.64:8082/ loadfactor=1
</pre>
loadfactor表示负载均衡器发送给后台被代理服务器的请求数量权值，其值为1到100之间的任意数，默认为1。对于此配置策略，将会有连续3个请求被第一个成员处理，然后有1个请求被第二个成员处理。这种策略经常用在成员服务器负载和配置差异较大的情况下。
<ul>
	<li>权重请求响应负载均衡策略的配置</li>
</ul>
<pre class="brush:bash">ProxyPass / balancer://proxy/ lbmethod=bytraffic
BalancerMember http://172.20.29.64:8080/ loadfactor=3
BalancerMember http://172.20.29.64:8082/ loadfactor=1
</pre>
lbmethod=bytraffic表示后台被代理服务器处理请求的流量，此策略不通于策略二的区别是前者按请求数量来路由请求，后者按请求和响应的字节量来路由请求。相比之下策略三的负载将会更精确。

一般负责均衡算法有byrequest， bytraffic和bybusyness三种方式，具体采用何种方式需要看真实的业务场景。

<h1>RabbitMQ</h1>
高级消息队列协议（AMQP）是一个异步消息传递所使用的应用层协议规范。作为线路层协议，而不是API（例如JMS），AMQP客户端能够无视消息的来源任意发送和接受信息。现在，已经有相当一部分不同平台的服务器和客户端可以投入使用。

RabbitMQ是流行的开源消息队列系统，用erlang语言开发。

<h3>在AMQ模型里面有如下几个概念</h3>
<ol>
	<li>Broker：简单来说就是消息队列服务器实体。</li>

	<li>Exchange：消息交换机，它指定消息按什么规则，路由到哪个队列。</li>

	<li>Queue：消息队列载体，每个消息都会被投入到一个或多个队列。</li>

	<li>Binding：绑定，它的作用就是把exchange和queue按照路由规则绑定起来。</li>

	<li>Routing Key：路由关键字，exchange根据这个关键字进行消息投递。</li>

	<li>vhost：虚拟主机，一个broker里可以开设多个vhost，用作不同用户的权限分离。</li>

	<li>producer：消息生产者，就是投递消息的程序。</li>

	<li>consumer：消息消费者，就是接受消息的程序。</li>

	<li>channel：消息通道，在客户端的每个连接里，可建立多个channel，每个channel代表一个会话任务。</li>
</ol>
<img src="/wp-content/uploads/2016/02/distributed_web_log_rabbitmq.png" alt="distributed_web_log_rabbitmq" width="443" height="192" class="alignnone size-full wp-image-115" />

<h3>RabbitMQ消息队列的使用过程大概如下</h3>
<ol>

	<li>客户端连接到消息队列服务器，打开一个channel。</li>
	<li>客户端声明一个exchange，并设置相关属性。</li>


	<li>客户端声明一个queue，并设置相关属性。</li>


	<li>客户端使用routing key，在exchange和queue之间建立好绑定关系。</li>
	<li>客户端投递消息到exchange。</li>

	<li>exchange接收到消息后，就根据消息的key和已经设置的binding，进行消息路由，将消息投递到一个或多个队列里。</li>

</ol>

<img src="/wp-content/uploads/2016/02/distributed_web_log_mq.jpg" alt="distributed_web_log_mq" width="571" height="295" class="alignnone size-full wp-image-116" />
<h1>框架</h1>

<h3>Cloudera Flume</h3>

1. <a href="http://flume.apache.org/" target="_blank">http://flume.apache.org/</a>
2. <a href="http://www.cnblogs.com/oubo/archive/2012/05/25/2517751.html" target="_blank">http://www.cnblogs.com/oubo/archive/2012/05/25/2517751.html</a>

<h3>Facebook Scribe</h3>

1. <a href="http://www.cnblogs.com/brucewoo/archive/2011/12/13/2285482.html" target="_blank">http://www.cnblogs.com/brucewoo/archive/2011/12/13/2285482.html</a>
2. <a href="http://dongxicheng.org/search-engine/scribe-intro/" target="_blank">http://dongxicheng.org/search-engine/scribe-intro/</a>

<h3>Kafka</h3>

1. <a href="https://kafka.apache.org/" target="_blank">https://kafka.apache.org/</a>
2. <a href="http://research.microsoft.com/en-us/um/people/srikanth/netdb11/netdb11papers/netdb11-final12.pdf" target="_blank">http://research.microsoft.com/en-us/um/people/srikanth/netdb11/netdb11papers/netdb11-final12.pdf</a>
3. <a href="http://atcc.mysqlops.com/pdf/vipshop.pdf" target="_blank">http://atcc.mysqlops.com/pdf/vipshop.pdf</a>

<h1>参考资料</h1>

1. <a href="http://zjwyhll.blog.163.com/blog/static/7514978120126254222294" target="_blank">http://zjwyhll.blog.163.com/blog/static/7514978120126254222294</a>
2. <a href="http://httpd.apache.org/docs/2.4/developer/modguide.html" target="_blank">http://httpd.apache.org/docs/2.4/developer/modguide.html</a>
3. <a href="http://www.infoq.com/cn/articles/AMQP-RabbitMQ" target="_blank">http://www.infoq.com/cn/articles/AMQP-RabbitMQ</a>
4. <a href="http://www.nsbeta.info/archives/200" target="_blank">http://www.nsbeta.info/archives/200</a>