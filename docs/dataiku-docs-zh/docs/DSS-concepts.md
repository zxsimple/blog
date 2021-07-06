# DSS概念 #

>注意：一步步介绍DSS，建议参考[教程](https://www.dataiku.com/learn/portals/tutorials.html)

## 数据集
数据集是DSS中的核心对象，数据集是具有相同schema的一系列数据记录，与SQL世界中的表很类似。

DSS支持各种类型的数据集，例如：
- SQL表或者自定义SQL查询
- MongoDB中的collection
- 服务器上的数据文件目录
- Hadoop集群上的数据文件目录

## Recipes
Recipes是数据应用的构建单元，当进行诸如聚合、链接的时候，会通过在DSS中创建recipe来完成。

Recipes有输入数据集和输出数据集，以及从输入数据集到输出数据集的处理过程。

Data Science Studio支持各种类型的recipes：
- 执行在DSS中定义的可视化数据准备
- 执行SQL查询
- 执行Python脚本(可以使用或者不适用Pandas库)
- 执行Pig脚本
- 执行Hive查询
- 执行从输入数据集到输出数据集的数据同步

## 构建数据集
Recipes和数据集结合创建一张图，在图中定义数据集之间的关系已经如何构建数据集。这个图称为Flow。当输入数据集发生变化的时候依赖管理引擎将保证输出数据集是最新的。

## 被管控外部数据集
Data Science Studio从外部读取的数据叫"external"(外部)数据集。换句话说，当通过DSS recipes创建的数据集是"managed"(管控的)数据集。这意味着DSS拥有数据集之间的管理。例如，如果管控的数据集是SQL数据集，DSS将能够创建删除表，修改表结构。。。。。。

DSS通过"managed connections"(管控的链接)创建管控数据集，而管控链接表示数据存储，可通过以下存储创建管控数据集：
- DSS Server上的文件系统
- Hadoop HDFS
- SQL数据库
- Amazon S3
- ......

## 分区
>注意：分区在DSS社区版中不可用

分区指通过数据集的维度将数据拆分成多个，其中每个分区包含数据集的一个子数据集。
例如，一个表示客户的数据集可以通过国家进行分区。

有两种类型的分区维度：
- "离散"分区维度，这种分区一般有少量值。例如：国家、业务部门
- "时间"分区维度，数据集将通过固定的时间间隔进行分区(年、月、日或者小时)。时间在处理日志文件分区时通常有用。

一个数据集可以有多个维度进行分区。例如，访问日志可以被日期和产生日志所在的服务器进行分区。

在可能的情况下，DSS会隐式的分区对数据进行分区。例如，如果RDBMS引擎自身支持分区，在之上创建的SQL数据集将会支持本地分区，DSS会把数据集分区映射到SQL分区上。

在DSS中分区支持多种分区目标。

## 增量
分区时计算和增量单元。当数据集分区后，不需要构建整个数据集，只需按分区构建。

分区会被完全重计算。当构建数据集的X分区时，分区之前的数据会被删除，生成该数据集的recipes输出将会覆盖分区。重新计算分区时幂等的：多次计算不会产生重复记录。

尤其在处理时间序列数据的时候非常重要。如果你的输入日志数据是按天分区，输出数据集也是按天分区，当你每天构建分区X时，将不会产生重复数据。

##分区级依赖
分区数据集允许用户在分区级进行依赖管理。不同于用recipe基于输入数据集创建输出数据集，用户可指定在哪个输入数据集分区上创建输出数据集分区。

我们来看个例子：
- “日志”数据集按天分区，每天上游系统会增加当天分区
- “丰富化日志”数据集也是按天分区。每天，将会在相同分区的日志数据集分区上计算丰富化日志数据集。
- “滑动报表”数据集也是每天分区。每天计算之前7天的报表数据。
![](https://doc.dataiku.com/dss/latest/_images/partition-dependencies-1.jpg)

为实现此，我们在“日志”和“丰富化日志”之间申明"equals"依赖，在“丰富化日志”和“滑动报表”之间申明“sliding days”依赖。
![](https://doc.dataiku.com/dss/latest/_images/partition-dependencies-2.jpg)

当DSS计算分区X的“滑动报表”是，它将会计算：
- “丰富化日志”的分区X, X-1, X-2, ... X-6
- “日志”的分区X, X-1, X-2, ... X-6

DSS会检查哪些分区时最新的，同时计算缺失的分区。DSS会自动并行化计算“丰富化日志”每个缺失日期分区再计算“滑动报表”。

## 性能
通常当数据集分区后，它会在数据集上提高查询性能。尤其当SQL数据集本身所在的RDBMS支持分区的时候。