---
ID: 49
post_title: MySQL 多实例
author: getitsimple
post_excerpt: ""
layout: post
permalink: >
  http://hua-fei.com.cn/index.php/2016/02/24/mysql-multi-instances/
published: true
post_date: 2016-02-24 10:29:00
---
在某些情形下，为了达到资源的高效利用可能需要安装MySQL多实例在同一节点上，这儿就介绍一下整个安装配置的过程。具体对多实例能带来的好处可以参考此文<a href="http://www.hellodb.net/2011/06/mysql_multi_instance.html" target="_blank">MySQL单机多实例方案</a>
<h1>方式一mysqd_safe</h1>
安装依赖库

<pre class="brush:bash">
yum install cmake
yum install gcc-c++
yum install ncurses-devel
</pre>

&nbsp;

创建MySQL用户和组

<pre class="brush:bash">
sudo groupadd mysql
sudo useradd -s /sbin/nologin -g mysql -M mysql
</pre>
&nbsp;

创建MySQL路径和数据日志目录

<pre class="brush:bash">
mkdir /usr/local/mysql
mkdir /usr/local/mysql/data
mkdir /usr/local/mysql/log
</pre>
&nbsp;

编译安装MySQL
<pre class="brush:bash">
cd mysql-5.5.20
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/mysql &amp;&amp; make &amp;&amp; make install
mkdir /usr/local/mysql/data/{3306,3307,3308}/data –p
tree /usr/local/mysql/data/
mkdir /usr/local/mysql/log/{3306,3307,3308}/log –p
</pre>
&nbsp;

修改配置文件

需要给每个实例都创建
<pre class="brush:bash">
vim /usr/local/mysql/data/3306/my.cnf
</pre>
&nbsp;
<pre class="brush:bash">
[client]
#password       = your_password
port           = 3306
socket         = /tmp/mysql3306.sock
&nbsp;

user                             = mysql
# Here follows entries for some specific programs
&nbsp;

# The MySQL server
[mysqld]
port           = 3306
socket         = /tmp/mysql3306.sock
basedir              = /usr/local/mysql
datadir              = /usr/local/mysql/data/3306/data

[mysqld_safe]
log-error=/var/log/mysqld3306.log
pid-file=/var/run/mysqld3306/mysqld.pid
</pre>

复制另外两个配置文件
<pre class="brush:bash">
sed -e 's/3306/3307/g' /usr/local/mysql/data/3306/my.cnf &gt; /usr/local/mysql/data/3307/my.cnf
sed -e 's/3306/3308/g' /usr/local/mysql/data/3306/my.cnf &gt; /usr/local/mysql/data/3308/my.cnf
tree -L 3 /usr/local/mysql/data
</pre>

修改数据文件目录的所有者为mysql
<pre class="brush:bash">
chown -R mysql:mysql /usr/local/mysql/data/3306/data
chown -R mysql:mysql /usr/local/mysql/data/3307/data
chown -R mysql:mysql /usr/local/mysql/data/3308/data
chown -R mysql:mysql /usr/local/mysql/data/
chown -R mysql:mysql /var/run/mysqld3306
chown -R mysql:mysql /var/run/mysqld3307
chown -R mysql:mysql /var/run/mysqld3308
</pre>


启动MySQL
<pre class="brush:bash">
/usr/local/mysql/bin/mysqld_safe --defaults-file=/usr/local/mysql/data/3306/my.cnf &amp;
/usr/local/mysql/bin/mysqld_safe --defaults-file=/usr/local/mysql/data/3307/my.cnf &amp;
/usr/local/mysql/bin/mysqld_safe --defaults-file=/usr/local/mysql/data/3308/my.cnf &amp;
</pre>

停止MySQL
<pre class="brush:bash">
[root@localhost ~]# /usr/local/mysql/bin/mysqladmin -uroot -p -h localhost -P 3306 --socket=/tmp/mysql3306.sock shutdown
Enter password:
[root@localhost ~]# /usr/local/mysql/bin/mysqladmin -uroot -p -h localhost -P 3307 --socket=/tmp/mysql3307.sock shutdown
Enter password:
[root@localhost ~]# /usr/local/mysql/bin/mysqladmin -uroot -p -h localhost -P 3308 --socket=/tmp/mysql3308.sock shutdown
</pre>

登录MySQL
<pre class="brush:bash">
/usr/local/mysql/bin/mysql -u root -p --socket=/tmp/mysql3306.sock
</pre>

关闭防火墙启动远程访问
<pre class="brush:bash">
service iptables stop iptables
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
</pre>
<a href="http://blog.chinaunix.net/xmlrpc.php?r=blog/article&uid=21710354&id=4652983">http://blog.chinaunix.net/xmlrpc.php?r=blog/article&amp;uid=21710354&amp;id=4652983</a>

<h1>方式二mysql_multi</h1>
创建安装目录和数据目录
<pre class="brush:bash">
mkdir /usr/local/mysql_multi
mkdir /usr/local/mysql_multi/data{3309,3310,3311}
</pre>

编译安装
<pre class="brush:bash">
cd mysql-5.5.20
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/mysql_multi &amp;&amp; make &amp;&amp; make install
</pre>

#把用到的工具添加到/usr/bin目录
<pre class="brush:bash">
ln -s /usr/local/mysql_multi/bin/mysqld_multi /usr/bin/mysqld_multi
</pre>

#初始化数据目录
<pre class="brush:bash">
scripts/mysql_install_db --datadir=/usr/local/mysql_multi/data3309 --user=mysql
scripts/mysql_install_db --datadir=/usr/local/mysql_multi/data3310 --user=mysql
scripts/mysql_install_db --datadir=/usr/local/mysql_multi/data3311 --user=mysql
chown -R mysql:mysql /usr/local/mysql_multi/data3309
chown -R mysql:mysql /usr/local/mysql_multi/data3310
chown -R mysql:mysql /usr/local/mysql_multi/data3311
./mysqld_multi –example
</pre>

修改对应配置
<pre class="brush:bash">
vi my_multi.conf
</pre>

<pre class="brush:bash">
[mysqld_multi]
mysqld     = /usr/local/mysql_multi/bin/mysqld_safe
mysqladmin = /usr/local/mysql_multi/bin/mysqladmin
user       = mysql
#password   = my_password

[mysqld3309]
socket     = /tmp/mysql3309.sock
port       = 3309
pid-file   = /usr/local/mysql_multi/data3309/mysql.pid
log-error=/var/log/mysqld3309.log
datadir   = /usr/local/mysql_multi/data3309
user       = mysql

[mysqld3310]
socket     = /tmp/mysql3310.sock
port       = 3310
pid-file   = /usr/local/mysql_multi/data3310/mysql.pid
log-error   =/var/log/mysqld3310.log
datadir   = /usr/local/mysql_multi/data3310
user       = mysql

[mysqld3311]
socket     = /tmp/mysql3311.sock
port       = 3311
pid-file   = /usr/local/mysql_multi/data3311/mysql.pid
log-error=/var/log/mysqld3311.log
datadir   = /usr/local/mysql_multi/data3311
user       = mysql
</pre>

将mysql_multi添加到环境变量中
<pre class="brush:bash">
vi /etc/profile
</pre>

<pre class="brush:bash">
export PATH=/usr/local/mysql_multi/bin:$PATH


#查看数据库状态
<pre class="brush:bash">
mysqld_multi --defaults-extra-file=/usr/local/mysql_multi/my_multi.cnf report
</pre>

#启动
<pre class="brush:bash">
mysqld_multi --defaults-extra-file=/usr/local/mysql_multi/my_multi.cnf start
</pre>