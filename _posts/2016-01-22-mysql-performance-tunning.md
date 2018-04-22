---
ID: 37
post_title: MySQL 性能优化
author: getitsimple
post_excerpt: ""
layout: post
permalink: >
  http://hua-fei.com.cn/index.php/2016/01/22/mysql-performance-tunning/
published: true
post_date: 2016-01-22 12:37:53
---
<h1>thread_cache_size</h1>
今天发现MySQL Threads_created不断增长，都已经达到16K+了，MySQL也不能处理请求了！！！

<img class="alignnone wp-image-38" src="/wp-content/uploads/2016/01/mysql_thread_grow.png" alt="mysql_thread_grow" width="761" height="257" />

原来是如果thread_cache_size过小，Client在连接时如果没用缓冲的thread，造成资源的耗尽。

有人建议：

<em>Threads_created / Connections : If this is over 0.01, then increase thread_cache_size. At the very least, thread_cache_size should be greater than Max_used_connections</em>
<table>
<tbody>
<tr>
<td width="553">SHOW GLOBAL STATUS LIKE 'Threads%';

SHOW GLOBAL STATUS LIKE 'connections';

SHOW GLOBAL STATUS LIKE 'Max_used_connections';

SHOW VARIABLES like 'thread_cache_size%';

SHOW VARIABLES like 'max_connection%';</td>
</tr>
</tbody>
</table>
Threads_created: 1347

Connections: 74878

Max_used_connections：397

两者相比达到0.018远大于0.1，而我的thread_cache_size只有50

另外又有人提到一下规则

根据物理内存设置规则如下：
1G  ---&gt; 8
2G  ---&gt; 16
3G  ---&gt; 32

因此将thread_cache_size设置到400，这时Threads_created已经不再增长了