Serving系统相对于训练系统相对于训练系统面临低延时、高可用、高吞吐量方面的挑战。本文讨论`CTR`在线服务的设计架构。

### 非功能性需求

**低延时**

通常在`CTR`排序场景下，E2E的请求需要在50 ~ 100ms内完成响应，这其中包括业务处理，因此留给排序服务处理时间只有10 ~ 50ms；延时直接影响用户体验，除平均响应时间以外还得考虑`99th percentile`延时和服务的平滑程度。

**高并发**

在单个服务达到上万`QPS`的架构中，需要尽量提升单节点的处理能力，从服务调用全链路优化处理能力，减小请求网络传输量，降低计算的复杂度，充分利用多核并行计算等都是需要考虑的方面。

**高可用**

集群和网络拓扑是不可靠的；单节点故障，可用区网络故障等因素都可能引起不可以用。因此服务需要做到单个服务的自动恢复，通过节点和可用区的反亲和策略避免`鸡蛋放在一个篮子里`，另外良好的服务监控体系也有助于提升服务可用性。

**水平弹性**

在单节点处理请求的优化工作完成后就可以通过`scale out`增加节点实例来进一步提升服务处理能力，在`Cloud Native`时代交`Kubernetes`和相关网关组件来实现就可以了。

**可观测性**

[Observability](https://en.wikipedia.org/wiki/Observability)即服务内部状态对外可见的能力，也可以认为是监控体系中的一部分。在线上，经常需要观察请求的输入/输出参数值，以及分析请求调用链路中各个环节的时延情况，这对我们调试线上服务的处理逻辑和优化服务有很大帮忙，借助于`可观测性`的组件就可以实现快速线上debug和有目标的进行服务优化。[Jaeger](https://www.jaegertracing.io/)是个优秀的开源项目帮助实现服务的`可观测性`。

### 数据流和数据格式

在模型训练和在线预测中涉及到业务数据和特征数据的转化，特征数据进行模型预测，以及业务服务于Serving之间的集成；合理设计数据的存储格式，存储介质，选择合适的数据（序列化）传输协议，合理设计缓存对实现一个良好的Serving系统至关重要。



| 数据             | 阶段             | 存储介质(传输协议)                                  | 格式                                                         |
| ---------------- | ---------------- | --------------------------------------------------- | ------------------------------------------------------------ |
| 原始特征         | 特征处理输入     | HDFS/Hive                                           | Text/Parquet                                                 |
| 编码特征         | 特征处理输出     | HDFS                                                | LibSVM/LibFFM/[CSR](https://en.wikipedia.org/wiki/Sparse_matrix#Compressed_sparse_row_(CSR,_CRS_or_Yale_format)) |
| 特征Embedding    | 模型训练         | KV Store/[mmap](https://en.wikipedia.org/wiki/Mmap) | Embedding向量/向量索引                                       |
| 用户ID和候选集ID | 应用发起排序请求 | HTTP/Restful                                        | Text                                                         |
| 候选集           | 召回候选集输入   | [milvus](https://github.com/milvus-io/milvus)       | Item ID/Embedding向量                                        |
| 排序结果         | 精排             | Protobuffer                                         | Tensor                                                       |



### 特征处理

采用[分布式机器学习平台架构设计](https://zhuanlan.zhihu.com/p/349168557)中Pipeline的方式进行特征处理，定义一个[Pipeline Model](https://spark.apache.org/docs/latest/ml-pipeline.html#example-pipeline)，将对连续特征，单值离散和多值离散的处理以`stage`组织在Pipeline中：

```scala
val pipeline = new Pipeline().setStages(Array(tokenizer, hashingTF, lr))
```

在执行Pipeline `transform`之前会执行对stage中每个特征处理算子的`transformSchema`，这个过程会确定原始特征值和`transform`之后特征值的关系。以`StringIndexer`为例，它将会产生每个离散值和`Index`的映射。

```scala
protected def validateAndTransformSchema(
    schema: StructType,
    skipNonExistsCol: Boolean = false): StructType = {
    val (inputColNames, outputColNames) = getInOutCols()

    require(outputColNames.distinct.length == outputColNames.length,
            s"Output columns should not be duplicate.")

    val outputFields = inputColNames.zip(outputColNames).flatMap {
        case (inputColName, outputColName) =>
        schema.fieldNames.contains(inputColName) match {
            case true => Some(validateAndTransformField(schema, inputColName, outputColName))
            case false if skipNonExistsCol => None
            case _ => throw new SparkException(s"Input column $inputColName does not exist.")
        }
    }
    StructType(schema.fields ++ outputFields)
}
```

通过`save(path: String)`可以将Pipeline的处理特征的`FeatureMap`持久化，持久化的`FeatureMap`有助于我们：

- **处理增量特征**

  不可避免线上和增量数据中会不断有一些新的特征值产生，只需要`pipelineModel.load(persisted_featuremap)`后`fit`新的数据集就可以在原有基础上处理新增特征

- **处理实时特征**

  通过`Streaming`程序加载模型后通过`transform`处理实时特征

- **保证特征一致性**

  结合离线批量特征处理，增量特征更新以及实时特征生成，就可以确保线上线下特征的一致性，避免特征编码不一致引起的问题

我们将`FeatureMap`生成`<Feature Index - Feature Value>`以及`<Feature Value - Feature Index>`的**索引和反向索引**

### 特征Cache设计

对于典型的`Embedding+MLP`结构的`CTR`模型，在10亿特征维度输入`Embedding Size`为16的情况下模型的大小可以到6G，增加特征交叉和`Embedding Size`后模型大小很容易超过单个Device的物理内存。这对延时要求非常高的推荐在线服务具有很大的挑战。业界常用的做法有：

- **分布式Serving**

  模型`Embedding`通过sharding分布到多个`Parameter Server`节点上，训练和推理共享`Parameter Server`，在做预测时从多个`Parameter Server`上拉取参数进行计算。

  这个方案的问题在于训练和推理过程中会产生大量的`pull/push`产生的网络I/O，异构的网络环境和I/O的波动很容易引起服务的抖动。

- **GPU Serving**

  典型的就是Nvidia的`TensorRT`和`Triton`

  在动辄几十上百个模型在线上万TPS的并发环境中成本是需要考虑的

对于每个已知的`Feature`，它所训练得到的`Embedding`值是固定的，不同的User和Item只是由诸多特征值组合后得到的单个样例。我们可以考虑不从User/Item这个视角而是从特征值这个维度来看特征的表示。这样当样本特征发生变化可以组织任意的样本，结合[Overview: Extracting and serving feature embeddings for machine learning (google.com)](https://cloud.google.com/architecture/overview-extracting-and-serving-feature-embeddings-for-machine-learning)的思路，训练好`Embedding`后可以将`Embedding`存入Cache，在推理阶段直接从Cache中读取`Embedding`然后简化线上模型进行MLP计算推理。这样的好处在于大大降低线上模型的复杂程度、大小，加速线上模型的推理速度。

#### Feature-Embedding Cache

通过单独训练Embedding或者节后MLP后训练Embedding可以得到每个Feature ID对应的Embedding，以KV方式形成基础**Feature-Embedding Cache**。

| Key  | Desc         | Value                                                        |
| ---- | ------------ | ------------------------------------------------------------ |
| 1    | city=Beijing | -0.0005946392, 0.04863406, -0.054826, 0.016970966, 0.08773648, 0.10290126, 0.0886305, 0.02086803 |
| 1000 | gender=F     | -0.1218224, 0.00457253, -0.1300298, -0.14330669, -0.07828188, 0.050446015, 0.0888312, 0.05595955 |
| 5000 | color=红     | 0.0178568, 0.02670812, -0.0197524, 0.044513, 0.0527788, 0.0864418, -0.00748137, 0.00717828 |
| 7000 | age=30       | -0.03200363, -0.001716487, -0.03285564, -0.0374706, -0.053059, 0.089603, 0.07045869, 0.05091397 |

#### User/Item-Embedding Cache

通过执行离线计算可以计算每个User和Item对应的各个Feature值Embedding，对于一个100个特征的User，需要`100*8*4≈3KB`的存储空间。通常User规模可以达到10亿级别，而Item的规模量级会小很多，可以通过平衡Cache命中率和Cache容量将热点用户放入到**User-Embedding Cache**中。对于新用户和不活跃用户可以在线上查询**Feature-Embedding Cache**进行特征拼接。从而保证99th percentile延时要求。

### Cache存储

Embedding Cache的数据特点、容量以及在线读取并发的要求我们在选取Cache存储时需要考虑容量、速度和查询吞吐量满足此场景下的特点。

| Key数量 | 容量大小 | 查询延时 | QPS  |
| ------- | -------- | -------- | ---- |
| 亿级    | 百G      | <1ms     | 100K |

这个要求对Redis KV存储来说已经很吃力了，再加上更新缓存也会给它带来很大压力，我们考虑采用[Mmap](https://en.wikipedia.org/wiki/Mmap)来实现Cache。

#### Mmap

> [内存映射文件](https://www.write-bug.com/article/1596.html)
>
> 内存映射文件提供了一组独立的函数，使应用程序能够通过内存指针像访问内存一样访问磁盘上的文件。通过内存映射文件函数可以将磁盘上的文件全部或者部分映射到进程的虚拟地址空间的某个位置。一旦完成映射，对磁盘文件的访问就可以像访问内存文件一样简单。这样，向文件中写入数据的操作就是直接对内存进行赋值，而从文件的某个特定位置读取数据也就是直接从内存中取数据
>
> 当内存映射文件提供了对文件某个特定位置的直接读写时，真正对磁盘文件的读写操作是由系统底层处理的。而且在写操作时，数据也并非在每次操作时即时写入到磁盘，而是通过缓冲处理来提高系统整体性能。
>
> 内存映射文件的一个好处之一是，程序代码以标准的内存地址形式来访问文件数据，按照页面大小周期从磁盘写入数据的操作发生在后台，由操作系统底层来实现，这个过程对应用程序是完全透明的。虽然，内存映射文件最终还是要将文件从磁盘读入内存，实质上并没有省略什么操作，整体性能可能并没有获得什么提高，但是程序的结构会从中受益，缓冲区边界等问题将不复存在。而且对文件内容更新后的写操作也有操作系统自动完成，有操作系统判断内存中的页面是否为脏页面并仅将脏页面写入磁盘，比程序自己将全部数据写入文件的效率要高很多。

由于Java天生的GC特性导致`stop the world`，在线上是不可接受的。自然地考虑采用`Offheap`的`MappedByteBuffer`基于**内存映射文件**实现大容量高速缓存。

MMap具有以下特点：

1. 可以多进程进行文件的读写
2. 可以跨文件系统和操作系统进行文件移植
3. 快速随机访问
4. 支持每秒百万的`get`操作
5. 多进程并不影响读取速度

大家熟知的[Kafak](https://blog.csdn.net/zc19921215/article/details/104150371), [RocektMQ]([RocketMQ(七)：高性能探秘之MappedFile](https://www.cnblogs.com/yougewe/p/14164651.html))等高速I/O系统也是基于MMap来实现的。

我们可以用`MappedByteBuffer`和`RandomAccessFile`来实现，也可以用[Chronicle-Map](https://github.com/OpenHFT/Chronicle-Map)来快速实现高速Cache。

通过我的[测试](https://github.com/zxsimple/offheap-store-perf)，采用[Chronicle-Map](https://github.com/OpenHFT/Chronicle-Map)可以实现对5000万数据1114320 QPS/s的随机查询速度。

> 在实际测试过程中甚至SAS相比SSD的读取速度更快，这也说明了MMap不会采用传统的I/O进行数据读写

![Chronicle Map Read QPS by Storage](https://github.com/zxsimple/offheap-store-perf/raw/main/images/chroniclemap-read-ops-by-storage.png)

**Cache生成过程**

1. **Cache生成**：加载User/Item数据，从**Feature-Embedding Cache**中读取各个特征的`Embedding`值，然后逐个`concat`后作为Value，uid/item_id作为key然后[put](https://github.com/zxsimple/offheap-store-perf/blob/main/src/main/java/com/zxsimple/perf/ChronicleMapCache.java)到Cache中，在处理完所有数据后将会生成映射文件；

2. **Cache文件分发**：在以`Kubernete`s实现的弹性推理集群中首先通过`NAS`(当然也可以是其他网络存储)挂载#1中生成的映射文件，在每个Serving实例启动前执行`cp`将NAS中的文件复制到每个实例的本地，在此每个实例启动的时候可以挂载宿主机中共享的`Volume`作为本地Cache映射文件目录，因为通过[测试](https://github.com/zxsimple/offheap-store-perf)看到多进程读同一个映射文件不会影响Cache读取速度。

3. **加载Cache**：从Cache映射文件恢复`ChronicleMap`，然后像使用Map一样实现按key查询

   ```java
   map = ChronicleMapBuilder.of(Long.class, float[].class)
           .name("map")
           .entries(size)
           .averageValueSize(4 * EMBEDDING_SIZE)
           .recoverPersistedTo(new File(basePath), true);
   ```

这样我们就可以在Native代码中以内存Map一样进行Cache查询，完全不用担心网络I/O。

### Ranking Server

Ranking Server由一下几部分组成：

| 组件            | 输入                       | 输出                     | 存储/协议 |
| --------------- | -------------------------- | ------------------------ | --------- |
| Ranking Server  | User以及Candidate Items    | 每个Item的排序Score      | HTTP      |
| User/Item Cache | uid/item_id                | User/Item的Embedding表示 | MMap      |
| TF Serving      | User/Item Embedding Tensor | Score Tensor             | gRPC      |

由于Ranking Server接收每个User的Candidate列表，返回Item排序Score，只需要对应的id，不需要完整的feature；同时为了业务侧集成更方便直接采用Restful来实现。

- Ranking Server接收User/Item的id，直接查询User/Item-Embedding Cache获取User/Item的`Embedding`向量；

- 对于Cache中不存在的User，先获取用户的各个特征，经过查询**特征处理**阶段产生的`<Feature Value - Feature Index>`的**反向索引**可以得到用户的每个特征对应的Feature Index，在查询User-Embedding Cache获取非活跃用户或新用户的Embedding表示；

  > 这个过程相较于直接查询User-Embedding Cache会多出用户画像查询和特征索引查询，是低效的，因此我们一定要保证第一步中User-Embedding Cache查询的命中率

- 然后通过TensorFlow Serving gRPC接口调用模型实现排序模型的预测完成排序

- Ranking Server经过过滤后返回业务TopN的排序Item列表

Ranking Server和TF Serving的调用拓扑有一个比较tricking但确实提升性能的设计：将Ranking Server调用TF Serving的**分离**结构改为**耦合**结构，即将Ranking Server和TF Serving做在同一个镜像中，在启动Ranking Server时同时启动两个进程，通过`localhost`本地回环地址调用TF Serving的gRPC接口 ，这样就不用走网卡通信了。

> 又一个干掉网络I/O的设计!:smirk:
### 参考

[图像检索：向量索引](https://yongyuan.name/blog/vector-ann-search.html)

[Pensieve: An embedding feature platform](https://engineering.linkedin.com/blog/2020/pensieve)

[Overview: Extracting and serving feature embeddings for machine learning](https://cloud.google.com/architecture/overview-extracting-and-serving-feature-embeddings-for-machine-learning)
