---
typora-copy-images-to: ..\md-Hbase-Doc
---

# 简明HBase入门教程-开篇

http://www.nosqlnotes.com/technotes/hbase/hbase-overview-concepts/

这是HBase入门系列的第1篇文章，介绍HBase的数据模型、适用场景、集群关键角色、建表流程以及所涉及的HBase基础概念，本文内容基于HBase 2.0 beta2版本。本文既适用于HBase新手，也适用于已有一定经验的HBase开发人员。

> **一些常见的HBase新手问题**
>
> 1. 什么样的数据适合用HBase来存储？
> 2. 既然HBase也是一个数据库，能否用它将现有系统中昂贵的Oracle替换掉？
> 3. 存放于HBase中的数据记录，为何不直接存放于HDFS之上？
> 4. 能否直接使用HBase来存储文件数据？
> 5. Region(HBase中的数据分片)迁移后，数据是否也会被迁移？
> 6. 为何基于Spark/Hive分析HBase数据时性能较差？

## 开篇

用惯了Oracle/MySQL的同学们，心目中的数据表，应该是长成这样的：![1.RDBMS-Table](C:\Work\Source\docs\md-doc\md-Hbase-Doc\1.RDBMS-Table-1531210515322.png)

这种表结构规整，每一行都有固定的列构成，因此，非常适合结构化数据的存储。但在NoSQL领域，数据表的模样却往往换成了另外一种”画风”：![2.NoSQLTable](C:\Work\Source\docs\md-doc\md-Hbase-Doc\2.NoSQLTable-1531210512567.png)

行由看似”杂乱无章”的列组成，行与行之间也无须遵循一致的定义，而这种定义恰好符合半结构化数据或非结构化数据的特点。本文所要讲述的HBase，就属于该派系的一个典型代表。这些”杂乱无章”的列所构成的多行数据，被称之为一个”稀疏矩阵”，而上图中的每一个”黑块块”，在HBase中称之为一个KeyValue。

Apache HBase官方给出了这样的定义：

> *[Apache](http://www.apache.org/) HBase™ is the [Hadoop](http://hadoop.apache.org/) database, a **distributed**, **scalable**, **big data store**.即：Apache HBase是基于Hadoop构建的一个**分布式**的、**可伸缩**的**海量数据存储系统**。*

HBase常被用来存放一些结构简单，但数据量非常大的数据(通常在TB/PB级别)，如历史订单记录，日志数据，监控Metris数据等等，HBase提供了简单的基于Key值的快速查询能力。

HBase在国内市场已经取得了非常广泛的应用，在搜索引擎中，也可以看出来，HBase在国内呈现出了逐年上升的势态：![0.HBaseTrends](C:\Work\Source\docs\md-doc\md-Hbase-Doc\0.HBaseTrends.png)从Apache HBase所关联的github项目的commits统计信息来看，也可以看出来该项目非常活跃：![0.GithubCommits](C:\Work\Source\docs\md-doc\md-Hbase-Doc\0.GithubCommits.png)(需要说明的一点：HBase中的每一次commit，都已经过社区Commiter成员严格的Review，在commit之前，一个Patch可能已经被修改了几十个版本)

令人欣喜的是，国内的开发者也积极参与到了HBase社区贡献中，而且被社区接纳了多名PMC以及Committer成员。

本文将以**一条数据在HBase中的“旅程”**为线索，介绍HBase的核心概念与流程，几乎每一部分都可以展开成一篇独立的长文，但本文旨在让读者能够快速的了解HBase的架构轮廓，所以很多特性/流程被被一言带过，但这些特性在社区中往往经历了漫长的开发过程。至于讲什么以及讲到什么程度，本文都做了艰难的取舍，在讲解的过程中，将会穿插解答本文开始所提出的针对初学者的一些常见问题。

本文适用于HBase新手，而对于具备一定经验的HBase开发人员，相信本文也可以提供一些有价值的参考。本文内容基于HBase 2.0 beta 2版本，对比于1.0甚至是更早期的版本，2.0出现了大量变化，下面这些问题的答案与部分关键的变化相关（新手可以直接跳过这些问题）：

> 1. HBase meta Region在哪里提供服务？
> 2. HBase是否可以保证单行操作的原子性？
> 3. Region中写WAL与写MemStore的顺序是怎样的？
> 4. 你是否遇到过Region长时间处于RIT的状态？ 
> 5. 你认为旧版本中Assignment Manager的主要问题是什么？
> 6. 在面对Full GC问题时，你尝试做过哪些优化？
> 7. 你是否深究过HBase Compaction带来的“写放大”有多严重？
> 8. HBase的RPC框架存在什么问题？导致查询时延毛刺的原因有哪些？



本系列文章的**整体行文思路**如下：

> 1. 介绍HBase数据模型
> 2. 基于数据模型介绍HBase的适用场景快速
> 3. 介绍集群关键角色以及集群部署建议
> 4. 示例数据介绍
> 5. 写数据流程
> 6. 读数据流程
> 7. 数据更新
> 8. 负载均衡机制
> 9. HBase如何存储小文件数据



这些内容将会被拆成几篇文章。至于集群服务故障的处理机制，集群工具，周边生态，性能调优以及最佳实践等进阶内容，暂不放在本系列文章范畴内。

## 约定

本文范围内针对一些关键特性/流程，使用了加粗以及加下划线的方式做了强调，如”**ProcedureV2**“。这些特性往往在本文中仅仅被粗浅提及，后续计划以独立的文章来介绍这些特性/流程。

**术语缩写**：对于一些进程/角色名称，在本文范围内可能通过缩写形式来表述：![RoleNames](C:\Work\Source\docs\md-doc\md-Hbase-Doc\RoleNames.jpg)

## 数据模型

RowKey用来表示唯一一行记录的**主键**，HBase的数据是按照RowKey的**字典顺序**进行全局排序的，所有的查询都只能依赖于这一个排序维度。

> *通过下面一个例子来说明一下”**字典排序**“的原理：*
>
> *RowKey {“abc”, “a”, “bdf”, “cdf”, “defg”}按字典排序后的结果为{“a”, “abc”, “bdf”, “cdf”, “defg”}*
>
> *也就是说，当两个RowKey进行排序时，先对比两个RowKey的第一个字节，如果相同，则对比第二个字节，依此类推…如果在对比到第M个字节时，已经超出了其中一个RowKey的字节长度，那么，短的RowKey要被排在另外一个RowKey的前面*

## 稀疏矩阵

参考了Bigtable，HBase中一个表的数据是按照稀疏矩阵的方式组织的，”开篇”部分给出了一张关于HBase数据表的*抽象图*，我们再结合下表来加深大家关于”稀疏矩阵”的印象：![SparseMap](http://www.nosqlnotes.com/wp-content/uploads/2018/03/SparseMap.jpg)

看的出来：**每一行中，列的组成都是灵活的，行与行之间并不需要遵循相同的列定义**， 也就是HBase数据表”**schema-less**“的特点。

## Region

区别于Cassandra/DynamoDB的”Hash分区”设计，HBase中采用了”Range分区”，将Key的完整区间切割成一个个的”Key Range” ，每一个”Key Range”称之为一个Region。也可以这么理解：将HBase中拥有数亿行的一个大表，**横向切割**成一个个”**子表**“，这一个个”**子表**“就是**Region**：![3.Regions](http://www.nosqlnotes.com/wp-content/uploads/2018/03/3.Regions.png)Region是HBase中负载均衡的基本单元，当一个Region增长到一定大小以后，会自动分裂成两个。

## Column Family

如果将Region看成是一个表的**横向切割**，那么，一个Region中的数据列的**纵向切割**，称之为一个**Column Family**。每一个列，都必须归属于一个Column Family，这个归属关系是在写数据时指定的，而不是建表时预先定义。![4.RegionAndColumnFamilies](http://www.nosqlnotes.com/wp-content/uploads/2018/03/4.RegionAndColumnFamilies.png)

## KeyValue

KeyValue的设计不是源自Bigtable，而是要追溯至论文”The log-structured merge-tree(LSM-Tree)”。每一行中的每一列数据，都被包装成独立的拥有**特定结构**的KeyValue，KeyValue中包含了丰富的自我描述信息:

![5.KeyValue](C:\Work\Source\docs\md-doc\md-Hbase-Doc\5.KeyValue.png)

看的出来，KeyValue是支撑”稀疏矩阵”设计的一个关键点：一些Key相同的任意数量的独立KeyValue就可以构成一行数据。但这种设计带来的一个显而易见的缺点：**每一个KeyValue所携带的自我描述信息，会带来显著的数据膨胀**。

## 适用场景

在介绍完了HBase的数据模型以后，我们可以回答本文一开始的前两个问题：

> 1. 什么样的数据适合用HBase来存储？
> 2. 既然HBase也是一个数据库，能否用它将现有系统中昂贵的Oracle替换掉？



HBase的数据模型比较简单，数据按照RowKey排序存放，适合HBase存储的数据，可以简单总结如下：

- 以**实体**为中心的数据

  实体可以包括但不限于如下几种：

  - 自然人／账户／手机号／车辆相关数据
  - 用户画像数据（含标签类数据）
  - 图数据（关系类数据）

描述这些实体的，可以有基础属性信息、实体关系(图数据)、所发生的事件(如交易记录、车辆轨迹点)等等。

- 以**事件**为中心的数据
  - 监控数据
  - 时序数据
  - 实时位置类数据
  - 消息/日志类数据

上面所描述的这些数据，有的是结构化数据，有的是半结构化或非结构化数据。HBase的“稀疏矩阵”设计，使其应对非结构化数据存储时能够得心应手，但在我们的实际用户场景中，结构化数据存储依然占据了比较重的比例。由于HBase仅提供了基于RowKey的单维度索引能力，在应对一些具体的场景时，依然还需要基于HBase之上构建一些专业的能力，如：

- **OpenTSDB** 时序数据存储，提供基于Metrics+时间+标签的一些组合维度查询与聚合能力

- **GeoMesa** 时空数据存储，提供基于时间+空间范围的索引能力

- **JanusGraph** 图数据存储，提供基于属性、关系的图索引能力



HBase擅长于存储结构简单的海量数据但索引能力有限，而Oracle等传统关系型数据库(RDBMS)能够提供丰富的查询能力，但却疲于应对TB级别的海量数据存储，HBase对传统的RDBMS并不是取代关系，而是一种补充。

HBase与HDFS

我们都知道HBase的数据是存储于HDFS里面的，相信大家也都有这么的认知：

> HBase是一个**分布式数据库**，HDFS是一个**分布式文件系统**

理解了这一点，我们先来粗略回答本文已开始提出的其中两个问题：

> 1. **HBase中的数据为何不直接存放于HDFS之上？**
>
>    HBase中存储的海量数据记录，通常在几百Bytes到KB级别，如果将这些数据直接存储于HDFS之上，会导致大量的小文件产生，为HDFS的元数据管理节点(NameNode)带来沉重的压力。
>
> 2. **文件能否直接存储于HBase里面？**
>
>    如果是几MB的文件，其实也可以直接存储于HBase里面，我们暂且将这类文件称之为小文件，HBase提供了一个名为MOB的特性来应对这类小文件的存储。但如果是更大的文件，强烈不建议用HBase来存储，关于这里更多的原因，希望你在详细读完本文所有内容之后能够自己解答。



## 集群角色



> ***关于集群环境，你可以使用国内外大数据厂商的平台，如Cloudera，Hontonworks以及国内的华为，都发行了自己的企业版大数据平台，另外，华为云、阿里云中也均推出了全托管式的HBase服务。***

我们假设集群环境已经Ready了，先来看一下集群中的**关键角色**：![ClusterRoles](C:\Work\Source\docs\md-doc\md-Hbase-Doc\ClusterRoles.jpg)

相信大部分人对这些角色都已经有了一定程度的了解，我们快速的介绍一下各个角色在集群中的**主要**职责(**注意**：这里不是列出所有的职责)：

**ZooKeeper**

在一个拥有多个节点的分布式系统中，假设，只能有一个节点是主节点，如何快速的选举出一个主节点而且让所有的节点都认可这个主节点？这就是HBase集群中存在的一个最基础命题。利用ZooKeeper就可以非常简单的实现这类”仲裁”需求，ZooKeeper还提供了基础的事件通知机制，所有的数据都以 ZNode的形式存在，它也称得上是一个”微型数据库”。

**NameNode**

HDFS作为一个分布式文件系统，自然需要文件目录树的**元数据**信息，另外，在HDFS中每一个文件都是按照Block存储的，文件与Block的关联也通过**元数据**信息来描述。NameNode提供了这些**元数据信息的存储**。

- **DataNode**

HDFS的数据存放节点。

- **RegionServer**

HBase的**数据服务节点**。

- **Master

HBase的管理节点，通常在一个集群中设置一个主Master，一个备Master，主备角色的”仲裁”由ZooKeeper实现。 Master**主要职责**：

1. 负责管理所有的RegionServer

2. 建表/修改表/删除表等DDL操作请求的服务端执行主体

3. 管理所有的数据分片(Region)到RegionServer的分配如果一个RegionServer宕机或进程故障，由Master负责将它原来所负责的Regions转移到其它的RegionServer上继续提供服务

4. Master自身也可以作为一个RegionServer提供服务，该能力是可配置的

   

## 集群部署建议

如果基于物理机/虚拟机部署，通常建议：

- RegionServer与DataNode联合部署，RegionServer与DataNode按1:1比例设置。

这种部署的优势在于，RegionServer中的数据文件可以存储一个副本于本机的DataNode节点中，从而在读取时可以利用HDFS中的”**短路径读取(Short Circuit)**“来绕过网络请求，降低读取时延。![Deployment](C:\Work\Source\docs\md-doc\md-Hbase-Doc\Deployment.jpg)



- 管理节点独立于数据节点部署

如果是基于物理机部署，每一台物理机节点上可以设置几个RegionServers/DataNodes来提升资源使用率。

也可以选择基于容器来部署，如在HBaseCon Asia 2017大会知乎的演讲主题中，就提到了知乎基于Kubernetes部署HBase服务的实践。

对于公有云HBase服务而言，为了降低总体拥有成本(*TCO*)，通常选择”**计算与存储物理分离**“的方式，从架构上来说，可能导致平均时延略有下降，但可以借助于共享存储底层的IO优化来做一些”弥补”。HBase集群中的RegionServers可以按逻辑划分为多个Groups，一个表可以与一个指定的Group绑定，可以将RegionServer Group理解成将一个大的集群划分成了多个逻辑子集群，借此可以实现多租户间的隔离，这就是HBase中的**RegionServer Group**特性。

## 示例数据

给出一份我们日常都可以接触到的数据样例，先简单给出示例数据的字段定义：![Data-Sample-Definition](C:\Work\Source\docs\md-doc\md-Hbase-Doc\Data-Sample-Definition.jpg)

如上定义与实际的通话记录字段定义相去甚远，本文力求简洁，仅给出了最简单的示例。如下是”虚构”的样例数据：

![Data-Sample](C:\Work\Source\docs\md-doc\md-Hbase-Doc\Data-Sample.jpg)

在本文大部分内容中所涉及的一条数据，是上面加粗的最后一行”**Mobile1**“为”**13400006666**“这行记录。

## 写数据之前：建立连接

### Login

在启用了安全特性的前提下，Login阶段是为了完成**用户认证**(确定用户的合法身份)，这是后续一切**安全访问控制**的基础。当前Hadoop/HBase仅支持基于Kerberos的用户认证，ZooKeeper除了Kerberos认证，还能支持简单的用户名/密码认证，但都基于静态的配置，无法动态新增用户。如果要支持其它第三方认证，需要对现有的安全框架做出比较大的改动。

### 创建ConnectionConnection

可以理解为一个HBase集群连接的抽象，建议使用ConnectionFactory提供的工具方法来创建。因为HBase当前提供了两种连接模式：同步连接，异步连接，这两种连接模式下所创建的Connection也是不同的。我们给出ConnectionFactory中关于获取这两种连接的典型方法定义：

```
CompletableFuture<AsyncConnection> createAsyncConnection(Configuration conf,  User user);  

Connection createConnection(Configuration conf, ExecutorService pool, User user)      throws IOException;
```

Connection中主要维护着两类共享的资源：

- 线程池
- Socket连接

这些资源都是在真正使用的时候才会被创建，因此，此时的连接还只是一个”虚拟连接”。

## 写数据之前：创建数据表

### DDL操作的抽象接口 – Admin

Admin定义了常规的DDL接口，列举几个典型的接口：

```
void createNamespace(final NamespaceDescriptor descriptor) throws IOException; 
void createTable(final HTableDescriptor desc, byte[][] splitKeys) throws IOException;
TableName[] listTableNames() throws IOException;
```



### 预设合理的数据分片 – Region

分片数量会给读写吞吐量带来直接的影响，因此，建表时通常建议由用户主动指定划分**Region分割点**，来设定Region的数量。HBase中数据是按照RowKey的字典顺序排列的，为了能够划分出合理的Region分割点，需要依据如下几点信息：

- Key的组成结构

- Key的数据分布预估

  如果不能基于Key的组成结构来预估数据分布的话，可能会导致数据在Region间的分布不均匀

- 读写并发度需求

  依据读写并发度需求，设置合理的Region数量

### 为表定义合理的Schema

既然HBase号称”schema-less”的数据存储系统，那何来的是schema? 的确，在数据库范式的支持上，HBase非常弱，这里的Schema，主要指如下一些信息的设置：

- NameSpace设置

- Column Family的数量

- 每一个Column Family中所关联的一些**关键配置**：

  - Compression

  ​	HBase当前可以支持Snappy，GZ，LZO，LZ4，Bzip2以及ZSTD压缩算法

  - DataBlock Encoding

  ​	HBase针对自身的特殊数据模型所做的一种压缩编码

  - BloomFilter

  ​	可用来协助快速判断一条记录是否存在

  - TTL

  ​	指定数据的过期时间

  - StoragePolicy

  ​	指定Column Family的存储策略，可选配置有：“ALL_SSD”，”ONE_SSD”，”HOT”，”WARM”，”COLD”，”LAZY_PERSIST”

  HBase中并不需要预先设置Column定义信息，这就是HBase schema-less设计的核心。

## Client发送建表请求到Master

建表的请求是通过RPC的方式由Client发送到Master:

- RPC接口基于Protocol Buffer定义

- 建表相关的描述参数，也由Protocol Buffer进行定义及序列化

Client端侧调用了Master服务的什么接口，参数是什么，这些信息都被通过RPC通信传输到Master侧，Master再依据这些接口\参数描述信息决定要执行的操作。2.0版本中，HBase目前已经支持基于Netty的**异步RPC框架**。

> ***关于HBase RPC框架***
>
> *早期的HBase RPC框架，完全借鉴了Hadoop中的实现，那时，Netty项目尚不盛行。*

Master侧接收到Client侧的建表请求以后，一些主要操作包括：

1. 生成每一个Region的描述信息对象HRegionInfo，这些描述信息包括：Region ID, Region名称，Key范围，表名称等信息

2. 生成每一个Region在HDFS中的文件目录

3. 将HRegionInfo信息写入到记录元数据的hbase:meta表中。

   

> ***说明***
>
> *meta表位于名为”hbase”的namespace中，因此，它的全称为”hbase:meta”。*
>
> *但在本系列文章范畴内，常将其缩写为”meta”。*

整个过程中，新表的状态也是记录在hbase:meta表中的，而不用再存储在ZooKeeper中。如果建表执行了一半，Master进程挂掉了，如何处理？这里是由HBase自身提供的一个名为**Procedure(V2)**的框架来保障操作的事务性的，备Master接管服务以后，将会继续完成整个建表操作。

一个被创建成功的表，还可以被执行如下操作：

- **Disable** 将所有的Region下线，该表暂停读写服务
- **Enable** 将一个Disable过的表重新Enable，也就是上线所有的Region来正常提供读写服务
- **Alter** 更改表或列族的描述信息

## Master分配Regions到各个RegionServers

新创建的所有的Regions，通过**AssignmentManager**将这些Region按照轮询(Round-Robin)的方式分配到每一个RegionServer中，具体的分配计划是由**LoadBalancer**来提供的。

AssignmentManager负责所有Regions的分配/迁移操作，Master中有一个定时运行的线程，来检查集群中的Regions在各个RegionServer之间的负载是否是均衡的，如果不均衡，则通过LoadBalancer生成相应的Region迁移计划，HBase中支持多种负载均衡算法，有最简单的仅考虑各RegionServer上的Regions数目的负载均衡算法，有基于迁移代价的负载均衡算法，也有数据本地化率优先的负载均衡算法，因为这一部分已经提供了插件化机制，用户也可以自定义负载均衡算法。

## 总结

到目前为止，文章介绍了如下关键内容：

1. HBase项目概述，呈现了HBase社区的活跃度以及搜索引擎热度等信息
2. HBase数据模型部分，讲到了RowKey，稀疏矩阵，Region，Column Family，KeyValue等概念
3. 基于HBase的数据模型，介绍了HBase的适合场景（以实体/事件为中心的简单结构的数据）介绍了HBase与HDFS的关系
4. 介绍了集群的关键角色：ZooKeeper, Master, RegionServer，NameNode, DataNode
5. 集群部署建议
6. 给出了一些示例数据
7. 写数据之前的准备工作：建立集群连接，建表（建表时应该定义合理的Schema以及设置合理的Region数量），建表由Master处理，新创建的Regions由Region AssignmentManager负责分配到各个RegionServer。下一篇文章将正式开始介绍写数据的流程。





# 简明HBase入门教程-Write全流程

http://www.nosqlnotes.com/technotes/hbase/hbase-overview-writeflow/

如果将上篇内容理解为一个冗长的”铺垫”，那么，从本文开始，剧情才开始正式展开。本文基于提供的样例数据，介绍了写数据的接口，RowKey定义，数据在客户端的组装，数据路由，打包分发，以及RegionServer侧将数据写入到Region中的全部流程。

**本文整体思路：**

1. 前文内容回顾
2. 示例数据
3. HBase可选接口介绍
4. 表服务接口介绍
5. 介绍几种写数据的模式
6. 如何构建Put对象(包含RowKey定义以及列定义)
7. 数据路由
8. Client侧的分组打包
9. Client发RPC请求到RegionServer
10. 安全访问控制
11. RegionServer侧处理：Region分发
12. Region内部处理：写WAL
13. Region内部处理：写MemStore



为了保证”故事”的完整性，导致本文篇幅过长，非常抱歉，读者可以按需跳过不感兴趣的内容。

 

## 前文回顾

 

上篇文章主要介绍了如下内容：

- HBase项目概况(搜索引擎热度/社区开发活跃度)
- HBase数据模型(RowKey，稀疏矩阵，Region，Column Family，KeyValue)
- 基于HBase的数据模型，介绍了HBase的适合场景（以**实体/事件**为中心的简单结构的数据）
- 介绍了HBase与HDFS的关系，集群关键角色以及部署建议
- 写数据前的准备工作：建立连接，建表



## 示例数据

（上篇文章已经提及，这里再复制一次的原因，一是为了让下文内容更容易理解，二是个别字段名称做了调整） 

给出一份我们日常都可以接触到的数据样例，先简单给出示例数据的字段定义： 

![Data-Sample-Definition](C:\Work\Source\docs\md-doc\md-Hbase-Doc\Data-Sample-Definition-1531216616650.jpg) 

本文力求简洁，仅给出了最简单的几个字段定义。如下是”虚构”的样例数据： 

![Data-Sample](http://www.nosqlnotes.com/wp-content/uploads/2018/03/Data-Sample.jpg) 



在本文大部分内容中所涉及的一条数据，是上面加粗的最后一行”**Mobile1**“为”**13400006666**“这行记录。 

在下面的流程图中，我们使用下面这样一个红色小图标来表示该数据所在的位置： 

![data](C:\Work\Source\docs\md-doc\md-Hbase-Doc\data.png)

## 写数据

### 可选接口

HBase中提供了如下几种主要的接口： 

- **Java Client API** 

HBase的基础API，应用最为广泛。 

- **HBase Shell** 

基于Shell的命令行操作接口，基于Java Client API实现。 

- **Restful API** 

Rest Server侧基于Java Client API实现。 

- **Thrift API** 

Thrift Server侧基于Java Client API实现。 

- **MapReduce Based Batch Manipulation API** 

基于MapReduce的批量数据读写API。 



除了上述主要的API，HBase还提供了**基于Spark的批量操作接口**以及**C++ Client**接口，但这两个特性都被规划在了3.0版本中，当前尚在开发中。 

无论是HBase Shell/Restful API还是Thrift API，都是基于Java Client API实现的。因此，接下来关于流程的介绍，都是基于Java Client API的调用流程展开的。 



### 关于表服务接口的抽象

同步连接与异步连接，分别提供了不同的表服务接口抽象： 

- Table 同步连接中的表服务接口定义 
- AsyncTable 异步连接中的表服务接口定义 

异步连接AsyncConnection获取AsyncTable实例的接口默认实现： 

```
default AsyncTable<AdvancedScanResultConsumer> getTable(TableName tableName) {
    return getTableBuilder(tableName).build();
}
```

同步连接ClusterConnection的实现类ConnectionImplementation中获取Table实例的接口实现： 

```
@Override
public Table getTable(TableName tableName) throws IOException {
    return getTable(tableName, getBatchPool());
}
```

### 写数据的几种方式

**Single Put** 

单条记录单条记录的随机put操作。Single Put所对应的接口定义如下： 

在AsyncTable接口中的定义： 

```
CompletableFuture<Void> put(Put put);
```

在Table接口中的定义： 

```
void put(Put put) throws IOException;
```

**Batch Put** 

汇聚了几十条甚至是几百上千条记录之后的**小批次**随机put操作。 

Batch Put只是本文对该类型操作的称法，实际的接口名称如下所示： 