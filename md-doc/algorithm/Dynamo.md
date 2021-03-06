[toc]


#Dynamo：Amazon的高可用性的键-值存储系统

Giuseppe DeCandia, Deniz Hastorun, Madan Jampani, Gunavardhan Kakulapati, Avinash Lakshman, Alex Pilchin, Swaminathan Sivasubramanian, Peter Vosshall and Werner Vogels

Amazon.com

译文参考 http://www.cnblogs.com/foxmailed/archive/2012/01/11/2318650.html ，稍微做了语句优化

##摘要
> ~~~cpp
Reliability at massive scale is one of the biggest challenges we face at Amazon.com, one of the largest e-commerce operations in the world; even the slightest outage has significant financial consequences and impacts customer trust. The Amazon.com platform, which provides services for many web sites worldwide, is implemented on top of an infrastructure of tens of thousands of servers and network components located in many datacenters around the world. At this scale, small and large components fail continuously and the way persistent state is managed in the face of these failures drives the reliability and scalability of the software systems.
~~~

~~~cpp
巨大规模系统的可靠性是我们在Amazon.com，这个世界上最大的电子商务公司之一，面对最大的挑战之一，即使最轻微的系统中断都会导致明显的经济后果,并且影响到客户对我们的信任。Amazon.com平台，它为全球许多网站服务，建立在成千上万的服务器和网络基础设施之上，而这些服务器和基础设置位于世界各地的许多数据中心。在这种规模下，各种大大小小的部件故障持续不断发生。在面对这些故障时，如何管理持久化的状态，决定了软件系统的可靠性和可扩展性。
~~~
 
> ~~~cpp
This paper presents the design and implementation of Dynamo, a highly available key-value storage system that some of Amazon’s core services use to provide an “always-on” experience. To achieve this level of availability, Dynamo sacrifices consistency under certain failure scenarios. It makes extensive use of object versioning and application-assisted conflict resolution in a manner that provides a novel interface for developers to use.
~~~

~~~cpp
本文介绍Dynamo的设计和实现，一个高度可用的key-value存储系统，一些Amazon的核心服务使用它用以提供一个“永远在线”的用户体验。为了达到这个级别的可用性，Dynamo在某些故障的场景中将牺牲一致性。它大量使用"对象版本"和"应用程序协助解决冲突"来为开发人员提供一个优雅可用的接口。
~~~


## 1.简介
> ~~~cpp
1. INTRODUCTION
Amazon runs a world-wide e-commerce platform that serves tens of millions customers at peak times using tens of thousands of servers located in many data centers around the world. There are strict operational requirements on Amazon’s platform in terms of performance, reliability and efficiency, and to support continuous growth the platform needs to be highly scalable. Reliability is one of the most important requirements because even the slightest outage has significant financial consequences and impacts customer trust. In addition, to support continuous growth, the platform needs to be highly scalable.
~~~

~~~cpp
Amazon运行着一个全球性的电子商务服务平台，在繁忙时段使用位于世界各地的许多数据中心的数千台服务器为几千万的客户服务。Amazon平台有严格的性能，可靠性和效率方面操作要求，并支持持续增长，因此平台需要高度可扩展性。可靠性是最重要的要求之一，因为即使最轻微的系统中断都有显著的经济后果和影响客户的信赖。此外，为了支持持续增长，平台需要高度可扩展性。
~~~
~~~
One of the lessons our organization has learned from operating Amazon’s platform is that the reliability and scalability of a system is dependent on how its application state is managed. Amazon uses a highly decentralized, loosely coupled, service oriented architecture consisting of hundreds of services. In this environment there is a particular need for storage technologies that are always available. For example, customers should be able to view and add items to their shopping cart even if disks are failing, network routes are flapping, or data centers are being destroyed by tornados. Therefore, the service responsible for managing shopping carts requires that it can always write to and read from its data store, and that its data needs to be available across multiple data centers.
~~~
~~~cpp
我们公司在运营Amazon平台时所获得的教训之一是，一个系统的可靠性和可扩展性依赖于它的应用状态如何管理。Amazon采用一种高度去中心化(decentralized)，松散耦合，由数百个服务组成的面向服务的架构。在这种环境中，特别需要一种"始终可用的"存储技术。例如，即使发生磁盘故障，网络线路抖动，或数据中心被龙卷风摧毁时，客户应该能够查看购物车和添加物品到自己的购物车。因此，负责管理购物车的服务，它可以随时写入和读取数据，并且数据需要跨越多个数据中心。
~~~
~~~
Dealing with failures in an infrastructure comprised of millions of components is our standard mode of operation; there are always a small but significant number of server and network components that are failing at any given time. As such Amazon’s software systems need to be constructed in a manner that treats failure handling as the normal case without impacting availability or performance.
~~~
```cpp
在一个由数百万个组件组成的基础设施中进行故障处理是我们的标准运作模式;在任何给定的时间段，总有一些小的但相当数量的服务器和网络组件故障，因此Amazon的软件系统需要将错误处理当作正常情况下来建造，而不影响可用性或性能。
```
~~~
To meet the reliability and scaling needs, Amazon has developed a number of storage technologies, of which the Amazon Simple Storage Service (also available outside of Amazon and known as Amazon S3), is probably the best known. This paper presents the design and implementation of Dynamo, another highly available and scalable distributed data store built for Amazon’s platform. Dynamo is used to manage the state of services that have very high reliability requirements and need tight control over the tradeoffs between availability, consistency, cost-effectiveness and performance. Amazon’s platform has a very diverse set of applications with different storage requirements. A select set of applications requires a storage technology that is flexible enough to let application designers configure their data store appropriately based on these tradeoffs to achieve high availability and guaranteed performance in the most cost effective manner.
~~~
~~~cpp
为了满足可靠性和可伸缩性的需要，Amazon开发了许多存储技术，其中Amazon S3服务大概是最为人熟知的。本文介绍了Dynamo的设计和实现，Dynamo是另外一个为Amazon的平台上构建的高度可用和可扩展的分布式数据存储系统。Dynamo被用来管理服务状态并且要求非常高的可靠性，而且需要严格控制可用性，一致性，成本效益和性能的之间的权衡。Amazon平台的不同应用对存储要求的差异化非常大。一部分应用需要存储技术具有足够的灵活性，让程序设计人员配置适当的数据存储来达到一种平衡，以最大化收益实现高可用性和性能保证。
~~~
~~~
There are many services on Amazon’s platform that only need primary-key access to a data store. For many services, such as those that provide best seller lists, shopping carts, customer preferences, session management, sales rank, and product catalog, the common pattern of using a relational database would lead to inefficiencies and limit scale and availability. Dynamo provides a simple primary-key only interface to meet the requirements of these applications.
~~~
~~~cpp
Amazon服务平台中的许多服务只需要通过主键访问数据存储。对于许多服务，如提供最畅销书排行榜，购物车，客户偏好，会话管理，销售等级，产品目录，使用关系数据库的模式会导致效率低下，可扩展性和可用性也很有限。Dynamo提供了一个简单的仅仅通过主键访问的接口，以满足这些应用的要求。
~~~
 
~~~
Dynamo uses a synthesis of well known techniques to achieve scalability and availability: Data is partitioned and replicated using consistent hashing [10], and consistency is facilitated by object versioning [12]. The consistency among replicas during updates is maintained by a quorum-like technique and a decentralized replica synchronization protocol. Dynamo employs a gossip based distributed failure detection and membership protocol. Dynamo is a completely decentralized system with minimal need for manual administration. Storage nodes can be added and removed from Dynamo without requiring any manual partitioning or redistribution.
~~~
~~~cpp
Dynamo使用了一些众所周知的技术来实现可伸缩性和可用性：使用一致性哈希进行数据分区(data partitioned)和复制(replicated)[10]，使用对象版本(object versioning)提供一致性[12]。在更新时，副本之间通过多数派仲裁(quorum-like)技术和去中心化的副本同步协议来保证一致性。Dynamo采用基于gossip的协议来进行检测分布式故障及成员(membership)变动(也即token环上的节点在响应节点加入(join)/离开(leaving)/移除(removing)/消亡(dead)等所采取的动作以维持DHT/Partitioning的正确语义)。Dynamo是一个完全去中心化的系统，只需要很少的人工管理。存储节点可以添加和删除，而不需要任何手动分区或手动重新分配数据。
~~~
~~~
In the past year, Dynamo has been the underlying storage technology for a number of the core services in Amazon’s e- commerce platform. It was able to scale to extreme peak loads efficiently without any downtime during the busy holiday shopping season. For example, the service that maintains shopping cart (Shopping Cart Service) served tens of millions requests that resulted in well over 3 million checkouts in a single day and the service that manages session state handled hundreds of thousands of concurrently active sessions.
~~~
~~~cpp
在过去的一年，Dynamo已经成为Amazon电子商务平台的核心服务的底层存储技术。它能够有效地扩展到极端高峰负载，在繁忙的假日购物季节没有任何的停机时间。例如，维护购物车(购物车服务)的服务，在一天内承担数千万的请求，并因此导致超过300万结算请求，以及管理十万计的并发活动会话的状态。
~~~
~~~
The main contribution of this work for the research community is the evaluation of how different techniques can be combined to provide a single highly-available system. It demonstrates that an eventually-consistent storage system can be used in production with demanding applications. It also provides insight into the tuning of these techniques to meet the requirements of production systems with very strict performance demands.
~~~
~~~cpp
这项工作对研究社区的主要贡献是评估不同的技术如何能够结合在一起，并最终提供一个单一的高可用性的系统。它表明一个最终一致(eventually-consistent)的存储系统可以在生产环境中采用。它也对这些技术的调优进行了深入的分析，以满足生产系统对性能上的严格需求。
~~~
 
~~~
The paper is structured as follows. Section 2 presents the background and Section 3 presents the related work. Section 4 presents the system design and Section 5 describes the implementation. Section 6 details the experiences and insights gained by running Dynamo in production and Section 7 concludes the paper. There are a number of places in this paper where additional information may have been appropriate but where protecting Amazon’s business interests require us to reduce some level of detail. For this reason, the intra- and inter-datacenter latencies in section 6, the absolute request rates in section 6.2 and outage lengths and workloads in section 6.3 are provided through aggregate measures instead of absolute details.
~~~
~~~cpp
本文的结构如下：第2节背景知识，第3节阐述有关工作，第4节介绍了系统设计，第5描述了实现，第6条详述在生产系统使用运营Dynamo时取得的经验，第7节结束本文。本文中，有些地方可以适当有更多的信息，但出于适当保护Amazon的商业利益，我们保留了部分细节。基于上述原因，第6节数据中心内和跨数据中心之间的延时，第6.2节的相对请求速率，以及第6.3节系统中断(outage)的时长，系统负载，都只是总体测量而不是详细的细节。
~~~

##2.背景
~~~
Amazon’s e-commerce platform is composed of hundreds of services that work in concert to deliver functionality ranging from recommendations to order fulfillment to fraud detection. Each service is exposed through a well defined interface and is accessible over the network. These services are hosted in an infrastructure that consists of tens of thousands of servers located across many data centers world-wide. Some of these services are stateless (i.e., services which aggregate responses from other services) and some are stateful (i.e., a service that generates its response by executing business logic on its state stored in persistent store).
~~~
~~~cpp
Amazon电子商务平台由数百个服务组成，它们协同工作，提供的功能包括建议，完成订单到欺诈检测。每个服务是通过一个明确的公开接口，通过网络访问。这些服务运行在位于世界各地的许多数据中心的数万台服务器组成的基础设施之上。其中一些服务是无状态(比如，一些服务只是收集(aggregate)其他服务的response)，有些是有状态的(比如，通过使用持久性存储区中的状态，，执行业务逻辑，进而生成response)。
~~~
~~~
Traditionally production systems store their state in relational databases. For many of the more common usage patterns of state persistence, however, a relational database is a solution that is far from ideal. Most of these services only store and retrieve data by primary key and do not require the complex querying and management functionality offered by an RDBMS. This excess functionality requires expensive hardware and highly skilled personnel for its operation, making it a very inefficient solution. In addition, the available replication technologies are limited and typically choose consistency over availability. Although many advances have been made in the recent years, it is still not easy to scale-out databases or use smart partitioning schemes for load balancing.
~~~
~~~cpp
传统的生产系统将状态存储在关系数据库中。对于许多更通用的状态存储模式，关系数据库方案是远不够理想。因为这些服务大多只通过数据的主键存储和检索数据，并且不需要RDBMS提供的复杂的查询和管理功能。这多余的功能，其运维需要昂贵的硬件和高技能人才，从而使得RDBMS使其成为一个非常低效的解决方案。此外，现有的复制(replication)技术是有限的，通常选择一致性是以牺牲可用性为代价(typically choose consistency over availability)。虽然最近几年已经提出了许多进展，但数据库水平扩展(scaleout)或使用负载平衡智能划分方案仍然不那么容易。
~~~
~~~
This paper describes Dynamo, a highly available data storage technology that addresses the needs of these important classes of services. Dynamo has a simple key/value interface, is highly available with a clearly defined consistency window, is efficient in its resource usage, and has a simple scale out scheme to address growth in data set size or request rates. Each service that uses Dynamo runs its own Dynamo instances.
~~~
~~~cpp
本文介绍了Dynamo，一个高度可用的数据存储技术，能够满足这些重要类型的服务的需求。Dynamo有一个简单的键/值接口，它是高可用的并同时具有清晰定义的一致性滑动窗口，它在资源利用方面是高效的，并且在解决规模增长或请求率上升时具有一个简单的水平扩展(scaleout)方案。每个使用Dynamo的服务运行它自己的Dynamo实例。
~~~

###2.1系统假设和要求
~~~
The storage system for this class of services has the following requirements:
Query Model: simple read and write operations to a data item that is uniquely identified by a key. State is stored as binary objects (i.e., blobs) identified by unique keys. No operations span multiple data items and there is no need for relational schema. This requirement is based on the observation that a significant portion of Amazon’s services can work with this simple query model and do not need any relational schema. Dynamo targets applications that need to store objects that are relatively small (usually less than 1 MB).
~~~
~~~cpp
这种类型的服务的存储系统具有以下要求：

查询模型：对数据项简单的读、写是通过一个主键来唯一标识的。状态存储为一个由唯一键确定的二进制对象(java的ByteBuffer)。没有横跨多个数据项的操作，也不需要关系型Schema(relational schema)。通过我们观察，相当一部分Amazon的服务可以使用这个简单的查询模型，并不需要任何关系模式。Dynamo的受众应用程序一般需要存储的对象都比较小(通常小于1MB)。
~~~
~~~
ACID Properties: ACID (Atomicity, Consistency, Isolation, Durability) is a set of properties that guarantee that database transactions are processed reliably. In the context of databases, a single logical operation on the data is called a transaction. Experience at Amazon has shown that data stores that provide ACID guarantees tend to have poor availability. This has been widely acknowledged by both the industry and academia [5]. Dynamo targets applications that operate with weaker consistency (the “C” in ACID) if this results in high availability. Dynamo does not provide any isolation guarantees and permits only single key updates.
~~~
~~~cpp
ACID属性：ACID(原子性，一致性，隔离性，持久性)是一种保证数据库事务可靠地处理的属性。从数据库角度来讲，对数据的一次逻辑操作被称作事务。Amazon的经验表明，具备ACID特性的数据存储提往往具有很差的可用性。这已被业界和学术界所公认[5]。Dynamo的目标是程序具有高可用性，弱一致性(ACID中的“C”)。Dynamo不提供任何数据隔离(Isolation)保证，只允许单一的键值更新。
~~~

~~~
Efficiency: The system needs to function on a commodity hardware infrastructure. In Amazon’s platform, services have stringent latency requirements which are in general measured at the 99.9th percentile of the distribution. Given that state access plays a crucial role in service operation the storage system must be capable of meeting such stringent SLAs (see Section 2.2 below). Services must be able to configure Dynamo such that they consistently achieve their latency and throughput requirements. The tradeoffs are in performance, cost efficiency, availability, and durability guarantees.
~~~
~~~cpp
效率：系统需运作在“日用品”(commodity，非常喜欢这个词，因为可以在家做试验！)级的硬件基础设施上。Amazon平台的服务都有着严格的延时要求，一般延时所需要度量到分布的99.9百分位。鉴于在服务操作中,对状态的访问起着至关重要的作用，存储系统必须能够满足那些严格的SLA(见以下2.2)，服务必须能够通过配置Dynamo，使他们不断达到延时和吞吐量的要求。因此，必须在成本效率，可用性和耐用性保证之间做权衡。
~~~
~~~
Other Assumptions: Dynamo is used only by Amazon’s internal services. Its operation environment is assumed to be non-hostile and there are no security related requirements such as authentication and authorization. Moreover, since each service uses its distinct instance of Dynamo, its initial design targets a scale of up to hundreds of storage hosts. We will discuss the scalability limitations of Dynamo and possible scalability related extensions in later sections.
~~~
~~~cpp
其他假设：Dynamo仅被Amazon内部的服务使用。它的操作环境被假定为不怀恶意的(non-hostile)，没有任何安全相关的身份验证和授权的要求。此外，由于每个服务使用其特定的Dynamo实例，它的最初设计目标的规模高达上百的存储主机。我们将在后面的章节讨论Dynamo可扩展性的限制和相关可能的扩展性的延伸。
~~~
 

###2.2服务水平协议(SLA)
~~~
To guarantee that the application can deliver its functionality in a bounded time, each and every dependency in the platform needs to deliver its functionality with even tighter bounds. Clients and services engage in a Service Level Agreement (SLA), a formally negotiated contract where a client and a service agree on several system-related characteristics, which most prominently include the client’s expected request rate distribution for a particular API and the expected service latency under those conditions. An example of a simple SLA is a service guaranteeing that it will provide a response within 300ms for 99.9% of its requests for a peak client load of 500 requests per second.
~~~
~~~cpp
为了保证应用程序可以在限定的(bounded)时间内递送(deliver)其功能，一个平台内的任何一个依赖都在一个更加限定的时间内递送其功能。客户端和服务端采用服务水平协议(SLA)，SLA是客户端和服务端在几个系统相关的特征上达成一致的一个正式协商合约，其中，最突出的包括客户对特定的API的请求速率分布的预期要求，以及根据这些条件，服务的预期延时。一个简单的例子是一个服务的SLA保证：在客户端每秒500个请求负载高峰时，99.9%的响应时间为300毫秒。
~~~
~~~
In Amazon’s decentralized service oriented infrastructure, SLAs play an important role. For example a page request to one of the e-commerce sites typically requires the rendering engine to construct its response by sending requests to over 150 services. These services often have multiple dependencies, which frequently are other services, and as such it is not uncommon for the call graph of an application to have more than one level. To ensure that the page rendering engine can maintain a clear bound on page delivery each service within the call chain must obey its performance contract.
~~~
~~~cpp
在Amazon的去中心化的面向服务的基础设施中，服务水平协议发挥了重要作用。例如，一个页面请求某个电子商务网站，通常需要页面渲染(rendering)引擎需要发送请求到150多个服务来构造其响应。这些服务通常有多个依赖关系，一般都是其他服务，因此，有一层以上调用路径的应用程序通常并不少见。为了确保该网页渲染引擎在递送页面时可以保持明确的时限，调用链内的每个服务必须履行合约中的性能指标。
~~~
~~~
Figure 1 shows an abstract view of the architecture of Amazon’s platform, where dynamic web content is generated by page rendering components which in turn query many other services. A service can use different data stores to manage its state and these data stores are only accessible within its service boundaries. Some services act as aggregators by using several other services to produce a composite response. Typically, the aggregator services are stateless, although they use extensive caching.
~~~
~~~cpp
图1显示了Amazon平台的抽象架构，动态网页的内容是由页面呈现组件生成，该组件进而查询许多其他服务。一个服务可以使用不同的数据存储来管理其状态，这些数据存储仅在其服务范围才能访问。有些服务作为聚合器使用其他一些服务，可产生合成(composite)响应。通常情况下，聚合服务是无状态，虽然他们广泛利用了缓存。
~~~

![haroopad icon](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/algorithm/dynamo-1.jpg)

图1：面向服务的Amazon平台架构。
~~~
A common approach in the industry for forming a performance oriented SLA is to describe it using average, median and expected variance. At Amazon we have found that these metrics are not good enough if the goal is to build a system where all customers have a good experience, rather than just the majority. For example if extensive personalization techniques are used then customers with longer histories require more processing which impacts performance at the high-end of the distribution. An SLA stated in terms of mean or median response times will not address the performance of this important customer segment. To address this issue, at Amazon, SLAs are expressed and measured at the 99.9th percentile of the distribution. The choice for 99.9% over an even higher percentile has been made based on a cost-benefit analysis which demonstrated a significant increase in cost to improve performance that much. Experiences with Amazon’s production systems have shown that this approach provides a better overall experience compared to those systems that meet SLAs defined based on the mean or median.
~~~
~~~cpp
行业中，描述面向性能的SLA的共同做法是：使用平均数(average)，中值(median)和预期差异(expected variance)。在Amazon，如果目标是建立一个对所有，而不是大多数客户都有着良好体验的系统，我们发现，这些指标不够好。例如，如果个性化(personalization)技术被广泛使用，那么有很长的历史的客户需要更多的处理，性能影响将表现在分布的高端。前面所述的基于平均或中值响应时间的SLA不能解决这一重要客户段的性能问题。为了解决这个问题，在Amazon，SLA是基于分布的99.9百分位来表达和测量的。选择百分位99.9的而不是更高是根据成本效益分析，其显示出在99.9之后，要继续提高性能，成本将大幅增加。系统的经验与Amazon的生产表明，相比于那些基于平均或中值定义的SLA的系统，该方法提供了更好的整体体验。
~~~
~~~
In this paper there are many references to this 99.9th percentile of distributions, which reflects Amazon engineers’ relentless focus on performance from the perspective of the customers’ experience. Many papers report on averages, so these are included where it makes sense for comparison purposes. Nevertheless, Amazon’s engineering and optimization efforts are not focused on averages. Several techniques, such as the load balanced selection of write coordinators, are purely targeted at controlling performance at the 99.9th percentile.
~~~
~~~cpp
本文多次提到这种99.9百分位分布，这反映了Amazon工程师从客户体验角度对性能不懈追求。许多论文统计平均数，所以在本论文的一些地方包括它可以用来作比较。然而，Amazon的工程和优化没有侧重于平均数。有几种技术，如作为写协调器(coordinators)的负载均衡的选择，纯粹是针对控制性能在99.9百分位的。
~~~
~~~
Storage systems often play an important role in establishing a service’s SLA, especially if the business logic is relatively lightweight, as is the case for many Amazon services. State management then becomes the main component of a service’s SLA. One of the main design considerations for Dynamo is to give services control over their system properties, such as durability and consistency, and to let services make their own tradeoffs between functionality, performance and cost- effectiveness.
~~~
~~~cpp
存储系统在建立一个服务的SLA中通常扮演重要角色，特别是如果业务逻辑是比较轻量级时，正如许多Amazon的服务的情况。状态管理就成为一个服务的SLA的主要组成部分。对dynamo的主要设计考虑的问题之一就是给各个服务控制权，通过系统属性来控制其持久性和一致性，并让服务自己在功能，性能和成本效益之间进行权衡。
~~~

###2.3设计考虑

Data replication algorithms used in commercial systems traditionally perform synchronous replica coordination in order to provide a strongly consistent data access interface. To achieve this level of consistency, these algorithms are forced to tradeoff the availability of the data under certain failure scenarios. For instance, rather than dealing with the uncertainty of the correctness of an answer, the data is made unavailable until it is absolutely certain that it is correct. From the very early replicated database works, it is well known that when dealing with the possibility of network failures, strong consistency and high data availability cannot be achieved simultaneously [2, 11]. As such systems and applications need to be aware which properties can be achieved under which conditions.

~~~cpp
在商业系统中，数据复制(Data replication)算法一般是执行同步复制，以提供一个强一致性的数据访问接口。为了达到这个水平的一致性，在某些故障情况下，这些算法被迫牺牲了数据可用性。例如，与其不能确定答案的正确性与否，不如让该数据一直不可用，一直到它绝对正确时。从最早期的备份(replicated)数据库，众所周知，当网络故障时，强一致性和高可用性不可能性同时实现[2，11]。因此，系统和应用程序需要知道在何种情况下可以达到哪些属性。
~~~

For systems prone to server and network failures, availability can be increased by using optimistic replication techniques, where changes are allowed to propagate to replicas in the background, and concurrent, disconnected work is tolerated. The challenge with this approach is that it can lead to conflicting changes which must be detected and resolved. This process of conflict resolution introduces two problems: when to resolve them and who resolves them. Dynamo is designed to be an eventually consistent data store; that is all updates reach all replicas eventually.


对于经常出现服务器和网络故障的系统，可使用乐观复制技术来提高系统的可用性，其变化可以在后台传播到副本，同时，并发和断开(disconnected)是可以容忍的。这种方法的挑战在于，它会导致更改冲突，而这些冲突必须能被检测到并协调解决。这种协调冲突的过程引入了两个问题：何时解决它们，谁解决它们。Dynamo被设计成最终一致性(eventually consistent)的数据存储，即所有的更新操作，最终达到所有副本。

~~~
An important design consideration is to decide when to perform the process of resolving update conflicts, i.e., whether conflicts should be resolved during reads or writes. Many traditional data stores execute conflict resolution during writes and keep the read complexity simple [7]. In such systems, writes may be rejected if the data store cannot reach all (or a majority of) the replicas at a given time. On the other hand, Dynamo targets the design space of an “always writeable” data store (i.e., a data store that is highly available for writes). For a number of Amazon services, rejecting customer updates could result in a poor customer experience. For instance, the shopping cart service must allow customers to add and remove items from their shopping cart even amidst network and server failures. This requirement forces us to push the complexity of conflict resolution to the reads in order to ensure that writes are never rejected.
~~~
~~~cpp
一个重要的设计考虑的因素是决定何时去协调更新操作冲突，即是否应该在读或写过程中协调冲突。许多传统数据存储在写的过程中执行协调冲突过程，从而保持读的复杂度相对简单[7]。在这种系统中，如果在给定的时间内数据存储不能达到所要求的所有或大多数副本数，写入可能会被拒绝。另一方面，Dynamo的目标是一个“永远可写”(always writable)的数据存储(即数据存储的“写”是高可用)。对于Amazon许多服务来讲，拒绝客户的更新操作可能导致糟糕的客户体验。例如，即使服务器或网络故障，购物车服务必须让客户仍然可向他们的购物车中添加和删除项。这项规定迫使我们将协调冲突的复杂性推给“读”，以确保“写”永远不会拒绝。
~~~

~~~
The next design choice is who performs the process of conflict resolution. This can be done by the data store or the application. If conflict resolution is done by the data store, its choices are rather limited. In such cases, the data store can only use simple policies, such as “last write wins” [22], to resolve conflicting updates. On the other hand, since the application is aware of the data schema it can decide on the conflict resolution method that is best suited for its client’s experience. For instance, the application that maintains customer shopping carts can choose to “merge” the conflicting versions and return a single unified shopping cart. Despite this flexibility, some application developers may not want to write their own conflict resolution mechanisms and choose to push it down to the data store, which in turn chooses a simple policy such as “last write wins”.
~~~
~~~cpp
下一个设计选择是谁执行协调冲突的过程。这可以通过数据存储或客户应用程序。如果冲突的协调是通过数据存储，它的选择是相当有限的。在这种情况下，数据存储只可能使用简单的策略，如”最后一次写入获胜“(last write wins)[22]，以协调冲突的更新操作。另一方面，客户应用程序，因为应用知道数据方案，因此它可以基于最适合的客户体验来决定协调冲突的方法。例如，维护客户的购物车的应用程序，可以选择“合并”冲突的版本，并返回一个统一的购物车。尽管具有这种灵活性，某些应用程序开发人员可能不希望写自己的协调冲突的机制，并选择下压到数据存储，从而选择简单的策略，例如“最后一次写入获胜”。
~~~
~~~
Other key principles embraced in the design are:
Incremental scalability: Dynamo should be able to scale out one storage host (henceforth, referred to as “node”) at a time, with minimal impact on both operators of the system and the system itself.

Incremental scalability: Dynamo should be able to scale out one storage host (henceforth, referred to as “node”) at a time, with minimal impact on both operators of the system and the system itself.

Symmetry: Every node in Dynamo should have the same set of responsibilities as its peers; there should be no distinguished node or nodes that take special roles or extra set of responsibilities. In our experience, symmetry simplifies the process of system provisioning and maintenance.

Decentralization: An extension of symmetry, the design should favor decentralized peer-to-peer techniques over centralized control. In the past, centralized control has resulted in outages and the goal is to avoid it as much as possible. This leads to a simpler, more scalable, and more available system.

Heterogeneity: The system needs to be able to exploit heterogeneity in the infrastructure it runs on. e.g. the work distribution must be proportional to the capabilities of the individual servers. This is essential in adding new nodes with higher capacity without having to upgrade all hosts at once.
~~~
~~~cpp
设计中包含的其他重要的设计原则是：

增量的可扩展性：Dynamo应能够一次水平扩展一台存储主机(以下，简称为“节点“)，而对系统操作者和系统本身的影响很小。

对称性：每个Dynamo节点应该与它的对等节点(peers)有一样的责任;不应该存在有区别的节点或采取特殊的角色或额外的责任的节。根据我们的经验，对称性(symmetry)简化了系统的配置和维护。

去中心化：是对对称性的延伸，设计应采用有利于去中心化而不是集中控制的技术。在过去，集中控制的设计造成系统中断(outages)，而本目标是尽可能避免它。这最终造就一个更简单，更具扩展性，更可用的系统。

异质性：系统必须能够利用异质性的基础设施运行。例如，负载的分配必须与各个独立的服务器的能力成比例。这样就可以一次只增加一个高处理能力的节点，而无需一次升级所有的主机。
~~~
 

##3.相关工作
###3.1点对点系统
There are several peer-to-peer (P2P) systems that have looked at the problem of data storage and distribution. The first generation of P2P systems, such as Freenet and Gnutella1, were predominantly used as file sharing systems. These were examples of unstructured P2P networks where the overlay links between peers were established arbitrarily. In these networks, a search query is usually flooded through the network to find as many peers as possible that share the data. P2P systems evolved to the next generation into what is widely known as structured P2P networks. These networks employ a globally consistent protocol to ensure that any node can efficiently route a search query to some peer that has the desired data. Systems like Pastry [16] and Chord [20] use routing mechanisms to ensure that queries can be answered within a bounded number of hops. To reduce the additional latency introduced by multi-hop routing, some P2P systems (e.g., [14]) employ O(1) routing where each peer maintains enough routing information locally so that it can route requests (to access a data item) to the appropriate peer within a constant number of hops.

已经有几个点对点(P2P)系统研究过数据存储和分配的问题。如第一代P2P系统Freenet和Gnutella，被主要用作文件共享系统。这些都是由链路任意建立的非结构化P2P的网络的例子。在这些网络中，通常是充斥着通过网络的搜索查询以找到尽可能多地共享数据的对等节点。P2P系统演进到下一代是广泛被称为结构化P2P。这些网络采用了全局一致的协议，以确保任何节点都可以有效率地传送一个搜索查询到那些需要数据的节点。系统，如Pastry[16]和Chord[20]使用路由机制，以确保查询可以在有限的跳数(hops)内得到回答。为了减少多跳路由引入的额外延时，有些P2P系统(例如，[14])采用O(1)路由，每个节点保持足够的路由信息，以便它可以在常数跳数下从本地路由请求(到要访问的数据项)到适当的节点。

Various storage systems, such as Oceanstore [9] and PAST [17] were built on top of these routing overlays. Oceanstore provides a global, transactional, persistent storage service that supports serialized updates on widely replicated data. To allow for concurrent updates while avoiding many of the problems inherent with wide-area locking, it uses an update model based on conflict resolution. Conflict resolution was introduced in [21] to reduce the number of transaction aborts. Oceanstore resolves conflicts by processing a series of updates, choosing a total order among them, and then applying them atomically in that order. It is built for an environment where the data is replicated on an untrusted infrastructure. By comparison, PAST provides a simple abstraction layer on top of Pastry for persistent and immutable objects. It assumes that the application can build the necessary storage semantics (such as mutable files) on top of it.

其他的存储系统，如Oceanstore[9]和PAST[17]是建立在这些交错的路由基础之上的。Oceanstore提供了一个全局性的，事务性，持久性存储服务，支持对广阔的(widely)复制的数据进行序列化更新。为允许并发更新的同时避免与广域锁定(wide-area locking)固有的许多问题，它使用了一个协调冲突的更新模型。[21]介绍了在协调冲突，以减少交易中止数量。Oceanstore协调冲突的方式是：通过处理一系列的更新，为他们选择一个最终的顺序，然后利用这个顺序原子地进行更新。它是为数据被复制到不受信任的环境的基础设施之上而建立。相比之下，PAST提供了一个基于Pastry和​​不可改变的对象的简单的抽象层。它假定应用程序可以在它之上建立必要的存储的语义(如可变文件)。


###3.2分布式文件系统和数据库
Distributing data for performance, availability and durability has been widely studied in the file system and database systems community. Compared to P2P storage systems that only support flat namespaces, distributed file systems typically support hierarchical namespaces. Systems like Ficus [15] and Coda [19] replicate files for high availability at the expense of consistency. Update conflicts are typically managed using specialized conflict resolution procedures. The Farsite system [1] is a distributed file system that does not use any centralized server like NFS. Farsite achieves high availability and scalability using replication. The Google File System [6] is another distributed file system built for hosting the state of Google’s internal applications. GFS uses a simple design with a single master server for hosting the entire metadata and where the data is split into chunks and stored in chunkservers. Bayou is a distributed relational database system that allows disconnected operations and provides eventual data consistency [21].

出于对性能，可用性和耐用性考虑，数据分发已被文件系统和数据库系统的社区广泛研究。对比于P2P存储系统的只支持平展的命名空间，分布式文件系统通常支持分层的命名空间。系统象Ficus[15]和Coda[19]其文件复制是以牺牲一致性为代价而达到高可用性。更新冲突管理通常使用专门的协调冲突程序。Farsite系统[1]是一个分布式文件系统，不使用任何类似NFS的中心服务器。Farsite使用复制来实现高可用性和可扩展性。谷歌文件系统[6]是另一个分布式文件系统用来承载谷歌的内部应用程序的状态。GFS使用简单的设计并采用单一的中心(master)服务器管理整个元数据，并且将数据被分成块存储在chunkservers上。Bayou是一个分布式关系数据库系统允许断开(disconnected)操作，并提供最终的数据一致性[21]。


~~~
Among these systems, Bayou, Coda and Ficus allow disconnected operations and are resilient to issues such as network partitions and outages. These systems differ on their conflict resolution procedures. For instance, Coda and Ficus perform system level conflict resolution and Bayou allows application level resolution. All of them, however, guarantee eventual consistency. Similar to these systems, Dynamo allows read and write operations to continue even during network partitions and resolves updated conflicts using different conflict resolution mechanisms. Distributed block storage systems like FAB [18] split large size objects into smaller blocks and stores each block in a highly available manner. In comparison to these systems, a key-value store is more suitable in this case because: (a) it is intended to store relatively small objects (size < 1M) and (b) key-value stores are easier to configure on a per-application basis. Antiquity is a wide-area distributed storage system designed to handle multiple server failures [23]. It uses a secure log to preserve data integrity, replicates each log on multiple servers for durability, and uses Byzantine fault tolerance protocols to ensure data consistency. In contrast to Antiquity, Dynamo does not focus on the problem of data integrity and security and is built for a trusted environment. Bigtable is a distributed storage system for managing structured data. It maintains a sparse, multi-dimensional sorted map and allows applications to access their data using multiple attributes [2]. Compared to Bigtable, Dynamo targets applications that require only key/value access with primary focus on high availability where updates are not rejected even in the wake of network partitions or server failures.
~~~
~~~cpp
在这些系统中，Bayou,Coda和Ficus允许断开操作以及具备从如网络分裂和中断的问题中复原的弹性。这些系统的不同之处在于协调冲突程序。例如，Coda和Ficus执行系统级协调冲突方案，Bayou允许应用程序级的解决方案。不过，所有这些都保证最终一致性。类似这些系统，Dynamo允许甚至在网络分区 ( partition - network partition, which is a break in the network that prevents one machine in one data center from interacting directly with another machine in other data center)的情况下继续进行读，写操作，以及使用不同的机制来协调有冲突的更新操作。分布式块存储系统，像FAB[18]将大对象分成较小的块，并以高度可用的方式存储。与上述这些系统相比较，一个key-value的存储在这种情况下更合适，因为：(a)它就是为了存放相对小的物体( 大小小于 1M )和(b)key-value存储是以每个应用更容易配置为基础的。Antiquity是一个广域(wide-area)分布式存储系统，专为处理多种服务器故障[23]。它使用一个安全的日志来保持数据的完整性，复制日志到多个服务器以达到耐久性，并使用Byzantine容错协议来保证数据的一致性。相对于antiquity，Dynamo不太注重数据完整性和安全问题，并为一个可信赖的环境而建立的。BigTable是一个管理结构化数据的分布式存储系统。它保留着稀疏的,多维的有序映射，并允许应用程序使用多个属性访问他们的数据[2]。相对于Bigtable中，Dynamo的目标应用程序只需要key/value并 *主要关注高可用性* ，甚至在网络分裂或服务器故障时，更新操作都不会被拒绝。
~~~

~~~
Traditional replicated relational database systems focus on the problem of guaranteeing strong consistency to replicated data. Although strong consistency provides the application writer a convenient programming model, these systems are limited in scalability and availability [7]. These systems are not capable of handling network partitions because they typically provide strong consistency guarantees.
~~~
~~~cpp
传统备份(replicated)关系数据库系统强调保证复制数据的一致性。虽然强一致性给应用编写者提供了一个更方便的应用程序编程模型，但这些系统都只有有限的可伸缩性和可用性[7]。这些系统不能处理网络分裂(partition)，因为它们通常提供强的一致性保证。
~~~
 

###3.3讨论
Dynamo differs from the aforementioned decentralized storage systems in terms of its target requirements. First, Dynamo is targeted mainly at applications that need an “always writeable” data store where no updates are rejected due to failures or concurrent writes. This is a crucial requirement for many Amazon applications. Second, as noted earlier, Dynamo is built for an infrastructure within a single administrative domain where all nodes are assumed to be trusted. Third, applications that use Dynamo do not require support for hierarchical namespaces (a norm in many file systems) or complex relational schema (supported by traditional databases). Fourth, Dynamo is built for latency sensitive applications that require at least 99.9% of read and write operations to be performed within a few hundred milliseconds. To meet these stringent latency requirements, it was imperative for us to avoid routing requests through multiple nodes (which is the typical design adopted by several distributed hash table systems such as Chord and Pastry). This is because multi- hop routing increases variability in response times, thereby increasing the latency at higher percentiles. Dynamo can be characterized as a zero-hop DHT, where each node maintains enough routing information locally to route a request to the appropriate node directly.

与上述去中心化的存储系统相比，Dynamo有着不同的目标需求：首先，Dynamo主要是针对应用程序需要一个“永远可写”数据存储，不会由于故障或并发写入而导致更新操作被拒绝。这是许多Amazon应用的关键要求。其次，如前所述，Dynamo是建立在一个所有节点被认为是值得信赖的单个管理域的基础设施之上。第三，使用Dynamo的应用程序不需要支持分层命名空间(许多文件系统采用的规范)或复杂的(由传统的数据库支持)关系模式的支持。第四，Dynamo是为延时敏感应用程序设计的，需要至少99.9％的读取和写入操作必须在几百毫秒内完成。为了满足这些严格的延时要求，这促使我们必须避免通过多个节点路由请求(这是被多个分布式哈希系统如Chord和Pastry采用典型的设计)。这是因为多跳路由将增加响应时间的可变性，从而导致百分较高的延时的增加。Dynamo可以被定性为零跳(zero-hop)的DHT，每个节点维护足够的路由信息从而直接从本地将请求路由到相应的节点。

##4系统架构

The architecture of a storage system that needs to operate in a production setting is complex. In addition to the actual data persistence component, the system needs to have scalable and robust solutions for load balancing, membership and failure detection, failure recovery, replica synchronization, overload handling, state transfer, concurrency and job scheduling, request marshalling, request routing, system monitoring and alarming, and configuration management. Describing the details of each of the solutions is not possible, so this paper focuses on the core distributed systems techniques used in Dynamo: partitioning, replication, versioning, membership, failure handling and scaling.

一个运行在生产环境里的存储系统的架构是复杂的。除了实际的数据持久化组件，系统需要有负载平衡，成员管理(membership)和故障检测，故障恢复，副本同步，过载处理，状态转移，并发性和工作调度，请求序列化(marshaling)，请求路由，系统监控和报警，以及配置管理等可扩展的且强大的解决方案。描述解决方案的每一个细节是不可能的，因此本文的重点是核心技术在分布式系统中使用Dynamo：分区(partitioning)，复制(replication)，版本(versioning)，会员(membership)，故障处理(failure handling)和伸缩性(scaling)。表1给出了简要的Dynamo使用的技术清单和各自的优势。


| Problem | Technique |  Advantage |
|------|-------|-------|
|Partitioning |Consistent Hashing|Incremental Scalability|
|High Availability for writes|Vector clocks with reconciliation during reads|Version size is decoupled from update rates.|
|Handling temporary failures|Sloppy Quorum and hinted handoff|Provides high availability and durability guarantee when some of the replicas are not available.|
|Recovering from permanent failures|Anti-entropy using Merkle trees|Synchronizes divergent replicas in the background.|
|Membership and failure detection|Gossip-based membership protocol and failure detection.|Preserves symmetry and avoids having a centralized registry for storing membership and node liveness information.|


| 问题 | 技术 |  优势 |
|------|-------|-------|
| 分区|一致性哈希|  增量可伸缩性
|写的高可用性|矢量时钟与读取过程中的协调|版本大小与更新操作速率脱钩。|
|暂时性的失败处理|草率仲裁，并暗示移交|一些副本不可用时,提供高可用性和耐用性的保证。|
|永久故障恢复|使用Merkle树的反熵|在后台同步不同的副本。|
|会员和故障检测|Gossip的成员和故障检测协议。|保持对称性并且避免了一个用于存储会员和节点活性信息的集中注册服务节点。|
表1：Dynamo使用的技术概要和其优势。


###4.1系统接口

Dynamo stores objects associated with a key through a simple interface; it exposes two operations: get() and put(). The get(key) operation locates the object replicas associated with the key in the storage system and returns a single object or a list of objects with conflicting versions along with a context. The put(key, context, object) operation determines where the replicas of the object should be placed based on the associated key, and writes the replicas to disk. The context encodes system metadata about the object that is opaque to the caller and includes information such as the version of the object. The context information is stored along with the object so that the system can verify the validity of the context object supplied in the put request.

~~~cpp
Dynamo通过一个简单的接口将对象与key关联，它暴露了两个操作：get()和put()。get(key)操作在存储系统中定位与key关联的对象副本，并返回一个对象或一个包含冲突的版本和对应的上下文对象列表。put(key,context,object)操作基于关联的key决定将对象的副本放在哪，并将副本写入到磁盘。该context包含对象的系统元数据并对于调用者是不透明的(opaque)，并且包括如对象的版本信息。上下文信息是与对象一起存储，以便系统可以验证请求中提供的上下文的有效性。
~~~
~~~
Dynamo treats both the key and the object supplied by the caller as an opaque array of bytes. It applies a MD5 hash on the key to generate a 128-bit identifier, which is used to determine the storage nodes that are responsible for serving the key.
~~~
~~~cpp
Dynamo将调用者提供的key和对象当成一个不透明的字节数组。它使用MD5对key进行Hash以产生一个128位的标识符，它是用来确定负责(responsible for)那个key的存储节点。
~~~


###4.2划分算法

One of the key design requirements for Dynamo is that it must scale incrementally. This requires a mechanism to dynamically partition the data over the set of nodes (i.e., storage hosts) in the system. Dynamo’s partitioning scheme relies on consistent hashing to distribute the load across multiple storage hosts. In consistent hashing [10], the output range of a hash function is treated as a fixed circular space or “ring” (i.e. the largest hash value wraps around to the smallest hash value). Each node in the system is assigned a random value within this space which represents its “position” on the ring. Each data item identified by a key is assigned to a node by hashing the data item’s key to yield its position on the ring, and then walking the ring clockwise to find the first node with a position larger than the item’s position.Thus, each node becomes responsible for the region in the ring between it and its predecessor node on the ring. The principle advantage of consistent hashing is that departure or arrival of a node only affects its immediate neighbors and other nodes remain unaffected.

Dynamo的关键设计要求之一是必须增量可扩展性。这就需要一个机制来将数据动态划分到系统中的节点(即存储主机)上去。Dynamo的分区方案依赖于一致哈希将负载分发到多个存储主机。在一致的哈希中[10]，一个哈希函数的输出范围被视为一个固定的圆形空间或“环”(即最大的哈希值绕到(wrap)最小的哈希值)。系统中的每个节点被分配了这个空间中的一个随机值，它代表着它的在环上的“位置”。每个由key标识的数据项通过计算数据项的key的hash值来产生其在环上的位置。然后沿顺时针方向找到第一个其位置比计算的数据项的位置大的节点。因此，每个节点变成了环上的一个负责它自己与它的前身节点间的区域(region)。一致性哈希的主要优点是节点的进进出出(departure or arrival)只影响其最直接的邻居，而对其他节点没影响。

The basic consistent hashing algorithm presents some challenges. First, the random position assignment of each node on the ring leads to non-uniform data and load distribution. Second, the basic algorithm is oblivious to the heterogeneity in the performance of nodes. To address these issues, Dynamo uses a variant of consistent hashing (similar to the one used in [10, 20]): instead of mapping a node to a single point in the circle, each node gets assigned to multiple points in the ring. To this end, Dynamo uses the concept of “virtual nodes”. A virtual node looks like a single node in the system, but each node can be responsible for more than one virtual node. Effectively, when a new node is added to the system, it is assigned multiple positions (henceforth, “tokens”) in the ring. The process of fine-tuning Dynamo’s partitioning scheme is discussed in Section 6.

这对基本的一致性哈希算法提出了一些挑战。首先，每个环上的任意位置的节点分配导致非均匀的数据和负荷分布。二，基本算法无视于节点的性能的异质性。为了解决这些问题，Dynamo采用了一致性哈希(类似于[10，20]中使用的)的变体：每个节点被分配到环多点而不是映射到环上的一个单点。为此，Dynamo使用了“虚拟节点”的概念。系统中一个虚拟节点看起来像单个节点，但每个节点可对多个虚拟节点负责。实际上，当一个新的节点添加到系统中，它被分配环上的多个位置(以下简称“标记” Token )。对Dynamo的划分方案进一步细化在第6部分讨论。

Using virtual nodes has the following advantages:
* If a node becomes unavailable (due to failures or routine maintenance), the load handled by this node is evenly dispersed across the remaining available nodes.
* When a node becomes available again, or a new node is added to the system, the newly available node accepts a roughly equivalent amount of load from each of the other available nodes.
* The number of virtual nodes that a node is responsible can decided based on its capacity, accounting for heterogeneity in the physical infrastructure.

使用虚拟节点具有以下优点：

* 如果一个节点不可用(由于故障或日常维护)，这个节点处理的负载将均匀地分散在剩余的可用节点。

* 当一个节点再次可用，或一个新的节点添加到系统中，新的可用节点接受来自其他可用的每个节点的负载量大致相当。

* 一个节点负责的虚拟节点的数目可以根据其处理能力来决定，顾及到物理基础设施的异质性。

###4.3复制

To achieve high availability and durability, Dynamo replicates its data on multiple hosts. Each data item is replicated at N hosts, where N is a parameter configured “per-instance”. Each key, k, is assigned to a coordinator node (described in the previous section). The coordinator is in charge of the replication of the data items that fall within its range. In addition to locally storing each key within its range, the coordinator replicates these keys at the N-1 clockwise successor nodes in the ring. This results in a system where each node is responsible for the region of the ring between it and its Nth predecessor. In Figure 2, node B replicates the key k at nodes C and D in addition to storing it locally. Node D will store the keys that fall in the ranges (A, B], (B, C], and (C, D].

为了实现高可用性和耐用性，Dynamo将数据复制到多台主机上。每个数据项被复制到N台主机，其中N是“每实例”(“per-instance)的配置参数。每个键，K，被分配到一个协调器(coordinator)节点(在上一节所述)。协调器节点掌控其负责范围内的复制数据项。除了在本地存储其范围内的每个key外，协调器节点复制这些key到环上顺时针方向的N-1后继节点。这样的结果是，系统中每个节点负责环上的从其自己到第N个前继节点间的一段区域。在图2中，节点B除了在本地存储键K外，在节点C和D处复制键K。节点D将存储落在范围(A,B],(B,C]和(C,D]上的所有键。

![haroopad icon](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/algorithm/dynamo-2.jpg)
图2：Dynamo的划分和键的复制。

The list of nodes that is responsible for storing a particular key is called the preference list. The system is designed, as will be explained in Section 4.8, so that every node in the system can determine which nodes should be in this list for any particular key. To account for node failures, preference list contains more than N nodes. Note that with the use of virtual nodes, it is possible that the first N successor positions for a particular key may be owned by less than N distinct physical nodes (i.e. a node may hold more than one of the first N positions). To address this, the preference list for a key is constructed by skipping positions in the ring to ensure that the list contains only distinct physical nodes.

一个负责存储一个特定的键的节点列表被称为首选列表(preference list)。该系统的设计，如将4.8节中解释，让系统中每一个节点可以决定对于任意key哪些节点应该在这个清单中。出于对节点故障的考虑，首选清单可以包含超过N个节点。请注意，由于虚拟节点的存在，对于一个特定的key的首批N个后继位置对一个的物理节点可能少于N个(即节点可以持有多个第一个N个位置)。为了解决这个问题，一个key首选列表的构建将跳过环上的一些位置，以确保该列表只包含不同的物理节点。

 

###4.4版本化的数据
Dynamo provides eventual consistency, which allows for updates to be propagated to all replicas asynchronously. A put() call may return to its caller before the update has been applied at all the replicas, which can result in scenarios where a subsequent get() operation may return an object that does not have the latest updates.. If there are no failures then there is a bound on the update propagation times. However, under certain failure scenarios (e.g., server outages or network partitions), updates may not arrive at all replicas for an extended period of time.

Dynamo提供最终一致性，从而允许更新操作可以异步地传播到所有副本。put()调用可能在更新操作被所有的副本执行之前就返回给调用者，这可能会导致一个场景：在随后的get()操作可能会返回一个不是最新的对象。如果没有失败，那么更新操作的传播时间将有一个上限。但是，在某些故障情况下(如服务器故障或网络partitions)，更新操作可能在一个较长时间内无法到达所有的副本。

There is a category of applications in Amazon’s platform that can tolerate such inconsistencies and can be constructed to operate under these conditions. For example, the shopping cart application requires that an “Add to Cart” operation can never be forgotten or rejected. If the most recent state of the cart is unavailable, and a user makes changes to an older version of the cart, that change is still meaningful and should be preserved. But at the same time it shouldn’t supersede the currently unavailable state of the cart, which itself may contain changes that should be preserved. Note that both “add to cart” and “delete item from cart” operations are translated into put requests to Dynamo. When a customer wants to add an item to (or remove from) a shopping cart and the latest version is not available, the item is added to (or removed from) the older version and the divergent versions are reconciled later.

在Amazon的平台，有一种类型的应用可以容忍这种不一致，在这种条件下，可以建造并操作。例如，购物车应用程序要求一个“添加到购物车“动作从来没有被忘记或拒绝。如果购物车的最近的状态是不可用，并且用户对一个较旧版本的购物车做了更改，这种变化仍然是有意义的并且应该保留。但同时它不应取代当前不可用的状态，而这不可用的状态本身可能含有的变化也需要保留。请注意在Dynamo中“添加到购物车“和”从购物车删除项目“这两个操作被转成put请求。当客户希望增加一个项目到购物车(或从购物车删除)但最新的版本不可用时，该项目将被添加到旧版本(或从旧版本中删除)并且不同版本将在后来被协调(reconciled)。

In order to provide this kind of guarantee, Dynamo treats the result of each modification as a new and immutable version of the data. It allows for multiple versions of an object to be present in the system at the same time. Most of the time, new versions subsume the previous version(s), and the system itself can determine the authoritative version (syntactic reconciliation). However, version branching may happen, in the presence of failures combined with concurrent updates, resulting in conflicting versions of an object. In these cases, the system cannot reconcile the multiple versions of the same object and the client must perform the reconciliation in order to collapse multiple branches of data evolution back into one (semantic reconciliation). A typical example of a collapse operation is “merging” different versions of a customer’s shopping cart. Using this reconciliation mechanism, an “add to cart” operation is never lost. However, deleted items can resurface.


为了提供这种保证，Dynamo将每次数据修改的结果当作一个新的且不可改变的数据版本。它允许系统中同一时间出现多个版本的对象。大多数情况，新版本包括（subsume）老的版本，且系统自己可以决定权威版本(语法协调 syntactic reconciliation)。然而，版本分支可能发生在并发的更新操作与失败的同时出现的情况，由此产生冲突版本的对象。在这种情况下，系统无法协调同一对象的多个版本，那么客户端必须执行协调，将多个分支演化后的数据崩塌(collapse)成一个合并的版本(语义协调)。一个典型的崩塌的例子是“合并”客户的不同版本的购物车。使用这种协调机制，一个“添加到购物车”操作是永远不会丢失。但是，已删除的条目可能会”重新浮出水面”(resurface)。


It is important to understand that certain failure modes can potentially result in the system having not just two but several versions of the same data. Updates in the presence of network partitions and node failures can potentially result in an object having distinct version sub-histories, which the system will need to reconcile in the future. This requires us to design applications that explicitly acknowledge the possibility of multiple versions of the same data (in order to never lose any updates).

重要的是要了解某些故障模式有可能导致系统中相同的数据不止两个，而是好几个版本。在网络分裂和节点故障的情况下，可能会导致一个对象有不同的历史分支，系统将需要在未来协调对象。这就要求我们在设计应用程序，明确意识到相同数据的多个版本的可能性(以便从来不会失去任何更新操作)。

Dynamo uses vector clocks [12] in order to capture causality between different versions of the same object. A vector clock is effectively a list of (node, counter) pairs. One vector clock is associated with every version of every object. One can determine whether two versions of an object are on parallel branches or have a causal ordering, by examine their vector clocks. If the counters on the first object’s clock are less-than-or-equal to all of the nodes in the second clock, then the first is an ancestor of the second and can be forgotten. Otherwise, the two changes are considered to be in conflict and require reconciliation.

Dynamo使用矢量时钟[12]来捕捉同一对象不同版本的因果关系。矢量时钟实际上是一个(node,counter)Pair列表(即(节点，计数器)列表)。矢量时钟是与每个对象的每个版本相关联。通过审查其向量时钟，我们可以判断一个对象的两个版本是平行分枝或有因果顺序。如果第一个时钟对象上的计数器在第二个时钟对象上小于或等于其他所有节点的计数器，那么第一个是第二个的祖先，可以被人忽略。否则，这两个变化被认为是冲突，并要求协调。

In Dynamo, when a client wishes to update an object, it must specify which version it is updating. This is done by passing the context it obtained from an earlier read operation, which contains the vector clock information. Upon processing a read request, if Dynamo has access to multiple branches that cannot be syntactically reconciled, it will return all the objects at the leaves, with the corresponding version information in the context. An update using this context is considered to have reconciled the divergent versions and the branches are collapsed into a single new version.

在dynamo中，当客户端更新一个对象，它必须指定它正要更新哪个版本。这是通过传递它从早期的读操作中获得的上下文对象来指定的，它包含了向量时钟信息。当处理一个读请求，如果Dynamo访问到多个不能语法协调(syntactically reconciled)的分支，它将返回分支叶子处的所有对象，其包含与上下文相应的版本信息。使用这种上下文的更新操作被认为已经协调了更新操作的不同版本并且分支都被折叠到一个新的版本。

![haroopad icon](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/algorithm/dynamo-3.jpg)
图3：对象的版本随时间演变。

To illustrate the use of vector clocks, let us consider the example shown in Figure 3. A client writes a new object. The node (say Sx) that handles the write for this key increases its sequence number and uses it to create the data's vector clock. The system now has the object D1 and its associated clock [(Sx, 1)]. The client updates the object. Assume the same node handles this request as well. The system now also has object D2 and its associated clock [(Sx, 2)]. D2 descends from D1 and therefore over-writes D1, however there may be replicas of D1 lingering at nodes that have not yet seen D2. Let us assume that the same client updates the object again and a different server (say Sy) handles the request. The system now has data D3 and its associated clock [(Sx, 2), (Sy, 1)].

为了说明使用矢量时钟，让我们考虑图3所示的例子。

1)客户端写入一个新的对象。节点(比如说Sx)，它处理对这个key的写：序列号递增，并用它来创建数据的向量时钟。该系统现在有对象D1和其相关的时钟[(Sx，1)]。

2)客户端更新该对象。假定也由同样的节点处理这个要求。现在该系统有对象D2和其相关的时钟[(Sx，2)]。D2继承自D1，因此覆写D1，但是节点中或许存在还没有看到D2的D1的副本。

3)让我们假设，同样的客户端更新这个对象但不同的服务器(比如Sy)处理了该请求。目前该系统具有数据D3及其相关的时钟[(Sx，2)，(Sy，1)]。

Next assume a different client reads D2 and then tries to update it, and another node (say Sz) does the write. The system now has D4 (descendant of D2) whose version clock is [(Sx, 2), (Sz, 1)]. A node that is aware of D1 or D2 could determine, upon receiving D4 and its clock, that D1 and D2 are overwritten by the new data and can be garbage collected. A node that is aware of D3 and receives D4 will find that there is no causal relation between them. In other words, there are changes in D3 and D4 that are not reflected in each other. Both versions of the data must be kept and presented to a client (upon a read) for semantic reconciliation.

4)接下来假设不同的客户端读取D2，然后尝试更新它，并且另一个服务器节点(如Sz)进行写操作。该系统现在具有D4(D2的子孙)，其版本时钟[(Sx，2)，(Sz，1)]。一个对D1或D2有所了解的节点可以决定，在收到D4和它的时钟时，新的数据将覆盖D1和D2，可以被垃圾收集。一个对D3有所了解的节点，在接收D4时将会发现，它们之间不存在因果关系。换句话说，D3和D4都有更新操作，但都未在对方的变化中反映出来。这两个版本的数据都必须保持并提交给客户端(在读时)进行语义协调。

Now assume some client reads both D3 and D4 (the context will reflect that both values were found by the read). The read's context is a summary of the clocks of D3 and D4, namely [(Sx, 2), (Sy, 1), (Sz, 1)]. If the client performs the reconciliation and node Sx coordinates the write, Sx will update its sequence number in the clock. The new data D5 will have the following clock: [(Sx, 3), (Sy, 1), (Sz, 1)].

5)现在假定一些客户端同时读取到D3和D4(上下文将反映这两个值是由read操作发现的)。读的上下文包含有D3和D4时钟的概要信息，即[(Sx，2)，(Sy，1)，(Sz，1)]的时钟总结。如果客户端执行协调，且由节点Sx来协调这个写操作，Sx将更新其时钟的序列号。D5的新数据将有以下时钟：[(Sx，3)，(Sy，1)，(Sz，1)]。

A possible issue with vector clocks is that the size of vector clocks may grow if many servers coordinate the writes to an object. In practice, this is not likely because the writes are usually handled by one of the top N nodes in the preference list. In case of network partitions or multiple server failures, write requests may be handled by nodes that are not in the top N nodes in the preference list causing the size of vector clock to grow. In these scenarios, it is desirable to limit the size of vector clock. To this end, Dynamo employs the following clock truncation scheme: Along with each (node, counter) pair, Dynamo stores a timestamp that indicates the last time the node updated the data item. When the number of (node, counter) pairs in the vector clock reaches a threshold (say 10), the oldest pair is removed from the clock. Clearly, this truncation scheme can lead to inefficiencies in reconciliation as the descendant relationships cannot be derived accurately. However, this problem has not surfaced in production and therefore this issue has not been thoroughly investigated.

关于向量时钟一个可能的问题是，如果许多服务器协调对一个对象的写，向量时钟的大小可能会增长。实际上，这是不太可能的，因为写入通常是由首选列表中的前N个节点中的一个节点处理。在网络分裂或多个服务器故障时，写请求可能会被不是首选列表中的前N个节点中的一个处理的，因此会导致矢量时钟的大小增长。在这种情况下，值得限制向量时钟的大小。为此，Dynamo采用了以下时钟截断方案：伴随着每个(节点，计数器)对，Dynamo存储一个时间戳表示最后一次更新的时间。当向量时钟中(节点，计数器)对的数目达到一个阈值(如10)，最早的一对将从时钟中删除。显然，这个截断方案会导至在协调时效率低下，因为后代关系不能准确得到。不过，这个问题还没有出现在生产环境，因此这个问题没有得到彻底研究。

 

###4.5执行get()和put()操作

Any storage node in Dynamo is eligible to receive client get and put operations for any key. In this section, for sake of simplicity, we describe how these operations are performed in a failure-free environment and in the subsequent section we describe how read and write operations are executed during failures.

Dynamo中的任何存储节点都有资格接收客户端的任何对key的get和put操作。在本节中，对简单起见，我们将描述如何在一个从不失败的(failure-free)环境中执行这些操作，并在随后的章节中，我们描述了在故障的情况下读取和写入操作是如何执行。

Both get and put operations are invoked using Amazon’s infrastructure-specific request processing framework over HTTP. There are two strategies that a client can use to select a node: (1) route its request through a generic load balancer that will select a node based on load information, or (2) use a partition-aware client library that routes requests directly to the appropriate coordinator nodes. The advantage of the first approach is that the client does not have to link any code specific to Dynamo in its application, whereas the second strategy can achieve lower latency because it skips a potential forwarding step.

GET和PUT操作都使用基于Amazon基础设施的特定请求，通过HTTP的处理框架来调用。一个客户端可以用有两种策略之一来选择一个节点：(1)通过一个普通的负载平衡器路由请求，它将根据负载信息选择一个节点，或(2)使用一个分区(partition)敏感的客户端库直接路由请求到适当的协调程序节点。第一个方法的优点是，客户端没有链接(link)任何Dynamo特定的代码在到其应用中，而第二个策略，Dynamo可以实现较低的延时，因为它跳过一个潜在的转发步骤。

A node handling a read or write operation is known as the coordinator. Typically, this is the first among the top N nodes in the preference list. If the requests are received through a load balancer, requests to access a key may be routed to any random node in the ring. In this scenario, the node that receives the request will not coordinate it if the node is not in the top N of the requested key’s preference list. Instead, that node will forward the request to the first among the top N nodes in the preference list.

处理读或写操作的节点被称为协调员。通常，这是首选列表中跻身前N个节点中的第一个。如果请求是通过负载平衡器收到，访问key的请求可能被路由到环上任何随机节点。在这种情况下，如果接收到请求节点不是请求的key的首选列表中前N个节点之一，它不会协调处理请求。相反，该节点将请求转发到首选列表中第一个跻身前N个节点。

Read and write operations involve the first N healthy nodes in the preference list, skipping over those that are down or inaccessible. When all nodes are healthy, the top N nodes in a key’s preference list are accessed. When there are node failures or network partitions, nodes that are lower ranked in the preference list are accessed.

读取和写入操作涉及到首选清单中的前N个健康节点，跳过那些瘫痪的(down)或者不可达(inaccessible)的节点。当所有节点都健康，key的首选清单中的前N个节点都将被访问。当有节点故障或网络分裂，首选列表中排名较低的节点将被访问。

To maintain consistency among its replicas, Dynamo uses a consistency protocol similar to those used in quorum systems. This protocol has two key configurable values: R and W. R is the minimum number of nodes that must participate in a successful read operation. W is the minimum number of nodes that must participate in a successful write operation. Setting R and W such that R + W > N yields a quorum-like system. In this model, the latency of a get (or put) operation is dictated by the slowest of the R (or W) replicas. For this reason, R and W are usually configured to be less than N, to provide better latency.

为了保持副本的一致性，Dynamo使用的一致性协议类似于仲裁(quorum)。该协议有两个关键配置值：R和W. R是必须参与一个成功的读取操作的最少数节点数目。W是必须参加一个成功的写操作的最少节点数。设定R和W，使得R+W>N产生类似仲裁的系统。在此模型中，一个get(or out)操作延时是由最慢的R(或W)副本决定的。基于这个原因，R和W通常配置为小于Ｎ，为客户提供更好的延时。

Upon receiving a put() request for a key, the coordinator generates the vector clock for the new version and writes the new version locally. The coordinator then sends the new version (along with the new vector clock) to the N highest-ranked reachable nodes. If at least W-1 nodes respond then the write is considered successful.
当收到对key的put()请求时，协调员生成新版本向量时钟并在本地写入新版本。协调员然后将新版本(与新的向量时钟一起)发送给首选列表中的排名前Ｎ个的可达节点。如果至少Ｗ-1个节点返回了响应，那么这个写操作被认为是成功的。

Similarly, for a get() request, the coordinator requests all existing versions of data for that key from the N highest-ranked reachable nodes in the preference list for that key, and then waits for R responses before returning the result to the client. If the coordinator ends up gathering multiple versions of the data, it returns all the versions it deems to be causally unrelated. The divergent versions are then reconciled and the reconciled version superseding the current versions is written back.

同样，对于一个get()请求，协调员为key从首选列表中排名前Ｎ个可达节点处请求所有现有版本的数据，然后等待Ｒ个响应，然后返回结果给客户端。如果最终协调员收集的数据的多个版本，它返回所有它认为没有因果关系的版本。不同版本将被协调，并且取代当前的版本，最后写回。


###4.6故障处理：暗示移交(Hinted Handoff)

If Dynamo used a traditional quorum approach it would be unavailable during server failures and network partitions, and would have reduced durability even under the simplest of failure conditions. To remedy this it does not enforce strict quorum membership and instead it uses a “sloppy quorum”; all read and write operations are performed on the first N healthy nodes from the preference list, which may not always be the first N nodes encountered while walking the consistent hashing ring.

Dynamo如果使用传统的仲裁(quorum)方式,在服务器故障和网络分裂的情况下它将是不可用，即使在最简单的失效条件下也将降低耐久性。为了弥补这一点，它不严格执行仲裁，即使用了“马虎仲裁”(“sloppy quorum”)，所有的读，写操作是由首选列表上的前N个健康的节点执行的，它们可能不总是在散列环上遇到的那前Ｎ个节点。

Consider the example of Dynamo configuration given in Figure 2 with N=3. In this example, if node A is temporarily down or unreachable during a write operation then a replica that would normally have lived on A will now be sent to node D. This is done to maintain the desired availability and durability guarantees. The replica sent to D will have a hint in its metadata that suggests which node was the intended recipient of the replica (in this case A). Nodes that receive hinted replicas will keep them in a separate local database that is scanned periodically. Upon detecting that A has recovered, D will attempt to deliver the replica to A. Once the transfer succeeds, D may delete the object from its local store without decreasing the total number of replicas in the system.

考虑在图2例子中Dynamo的配置，给定N=3。在这个例子中，如果写操作过程中节点A暂时Down或无法连接，然后通常本来在A上的一个副本现在将发送到节点D。这样做是为了保持期待的可用性和耐用性。发送到D的副本在其原数据中将有一个暗示，表明哪个节点才是在副本预期的接收者(在这种情况下Ａ)。接收暗示副本的节点将数据保存在一个单独的本地存储中，他们被定期扫描。在检测到了A已经复苏，D会尝试发送副本到A。一旦传送成功，D可将数据从本地存储中删除而不会降低系统中的副本总数。

Using hinted handoff, Dynamo ensures that the read and write operations are not failed due to temporary node or network failures. Applications that need the highest level of availability can set W to 1, which ensures that a write is accepted as long as a single node in the system has durably written the key it to its local store. Thus, the write request is only rejected if all nodes in the system are unavailable. However, in practice, most Amazon services in production set a higher W to meet the desired level of durability. A more detailed discussion of configuring N, R and W follows in section 6.

使用暗示移交，Dynamo确保读取和写入操作不会因为节点临时或网络故障而失败。需要最高级别的可用性的应用程序可以设置W为1，这确保了只要系统中有一个节点将key已经持久化到本地存储 ,　一个写是可以接受(即一个写操作完成即意味着成功)。因此，只有系统中的所有节点都无法使用时写操作才会被拒绝。然而，在实践中，大多数Amazon生产服务设置了更高的W来满足耐久性极别的要求。对N, R和W的更详细的配置讨论在后续的第6节。

It is imperative that a highly available storage system be capable of handling the failure of an entire data center(s). Data center failures happen due to power outages, cooling failures, network failures, and natural disasters. Dynamo is configured such that each object is replicated across multiple data centers. In essence, the preference list of a key is constructed such that the storage nodes are spread across multiple data centers. These datacenters are connected through high speed network links. This scheme of replicating across multiple datacenters allows us to handle entire data center failures without a data outage.

一个高度可用的存储系统具备处理整个数据中心故障的能力是非常重要的。数据中心由于断电，冷却装置故障，网络故障和自然灾害发生故障。Dynamo可以配置成跨多个数据中心地对每个对象进行复制。从本质上讲，一个key的首选列表的构造是基于跨多个数据中心的节点的。这些数据中心通过高速网络连接。这种跨多个数据中心的复制方案使我们能够处理整个数据中心故障。

 

###4.7处理永久性故障：副本同步

Hinted handoff works best if the system membership churn is low and node failures are transient. There are scenarios under which hinted replicas become unavailable before they can be returned to the original replica node. To handle this and other threats to durability, Dynamo implements an anti-entropy (replica synchronization) protocol to keep the replicas synchronized.
Hinted Handoff在系统成员流动性(churn)低，节点短暂的失效的情况下工作良好。有些情况下，在hinted副本移交回原来的副本节点之前，暗示副本是不可用的。为了处理这样的以及其他威胁的耐久性问题，Dynamo实现了反熵(anti-entropy，或叫副本同步)协议来保持副本同步。

To detect the inconsistencies between replicas faster and to minimize the amount of transferred data, Dynamo uses Merkle trees [13]. A Merkle tree is a hash tree where leaves are hashes of the values of individual keys. Parent nodes higher in the tree are hashes of their respective children. The principal advantage of Merkle tree is that each branch of the tree can be checked independently without requiring nodes to download the entire tree or the entire data set. Moreover, Merkle trees help in reducing the amount of data that needs to be transferred while checking for inconsistencies among replicas. For instance, if the hash values of the root of two trees are equal, then the values of the leaf nodes in the tree are equal and the nodes require no synchronization. If not, it implies that the values of some replicas are different. In such cases, the nodes may exchange the hash values of children and the process continues until it reaches the leaves of the trees, at which point the hosts can identify the keys that are “out of sync”. Merkle trees minimize the amount of data that needs to be transferred for synchronization and reduce the number of disk reads performed during the anti-entropy process.

为了更快地检测副本之间的不一致性，并且减少传输的数据量，Dynamo采用MerkleTree[13]。MerkleTree是一个哈希树(Hash Tree)，其叶子是各个key的哈希值。树中较高的父节点均为其各自孩子节点的哈希。该merkleTree的主要优点是树的每个分支可以独立地检查，而不需要下载整个树或整个数据集。此外，MerkleTree有助于减少为检查副本间不一致而传输的数据的大小。例如，如果两树的根哈希值相等，且树的叶节点值也相等，那么节点不需要同步。如果不相等，它意味着，一些副本的值是不同的。在这种情况下，节点可以交换children的哈希值，处理直到它到达了树的叶子，此时主机可以识别出“不同步”的key。MerkleTree减少为同步而需要转移的数据量，减少在反熵过程中磁盘执行读取的次数。

Dynamo uses Merkle trees for anti-entropy as follows: Each node maintains a separate Merkle tree for each key range (the set of keys covered by a virtual node) it hosts. This allows nodes to compare whether the keys within a key range are up-to-date. In this scheme, two nodes exchange the root of the Merkle tree corresponding to the key ranges that they host in common. Subsequently, using the tree traversal scheme described above the nodes determine if they have any differences and perform the appropriate synchronization action. The disadvantage with this scheme is that many key ranges change when a node joins or leaves the system thereby requiring the tree(s) to be recalculated. This issue is addressed, however, by the refined partitioning scheme described in Section 6.2.
Dynamo在反熵中这样使用MerkleTree：每个节点为它承载的每个key范围(由一个虚拟节点覆盖 key 集合)维护一个单独的MerkleTree。这使得节点可以比较key range中的key是否是最新。在这个方案中，两个节点交换MerkleTree的根，对应于它们承载的共同的键范围。其后，使用上面所述树遍历方法，节点确定他们是否有任何差异和执行适当的同步行动。方案的缺点是，当节点加入或离开系统时有许多key rangee变化，从而需要重新对树进行计算。通过由6.2节所述的更精炼partitioning方案，这个问题得到解决。

 

###4.8会员和故障检测
####4.8.1环会员(Ring Membership)

In Amazon’s environment node outages (due to failures and maintenance tasks) are often transient but may last for extended intervals. A node outage rarely signifies a permanent departure and therefore should not result in rebalancing of the partition assignment or repair of the unreachable replicas. Similarly, manual error could result in the unintentional startup of new Dynamo nodes. For these reasons, it was deemed appropriate to use an explicit mechanism to initiate the addition and removal of nodes from a Dynamo ring. An administrator uses a command line tool or a browser to connect to a Dynamo node and issue a membership change to join a node to a ring or remove a node from a ring. The node that serves the request writes the membership change and its time of issue to persistent store. The membership changes form a history because nodes can be removed and added back multiple times. A gossip-based protocol propagates membership changes and maintains an eventually consistent view of membership. Each node contacts a peer chosen at random every second and the two nodes efficiently reconcile their persisted membership change histories.

Amazon环境中，节点中断(由于故障和维护任务)常常是暂时的，但持续的时间间隔可能会延长。一个节点故障很少意味着一个节点永久离开，因此应该不会导致对已分配的分区重新平衡(rebalancing)和修复无法访问的副本。同样，人工错误可能导致意外启动新的Dynamo节点。基于这些原因，应当适当使用一个明确的机制来发起节点的增加和从环中移除节点。管理员使用命令行工具或浏览器连接到一个节点，并发出成员改变(membership change)指令指示一个节点加入到一个环或从环中删除一个节点。接收这一请求的节点写入成员变化以及适时写入持久性存储。该成员的变化形成了历史，因为节点可以被删除，重新添加多次。一个基于Gossip的协议传播成员变动，并维持成员的最终一致性。每个节点每间隔一秒随机选择随机的对等节点，两个节点有效地协调他们持久化的成员变动历史。

When a node starts for the first time, it chooses its set of tokens (virtual nodes in the consistent hash space) and maps nodes to their respective token sets. The mapping is persisted on disk and initially contains only the local node and token set. The mappings stored at different Dynamo nodes are reconciled during the same communication exchange that reconciles the membership change histories. Therefore, partitioning and placement information also propagates via the gossip-based protocol and each storage node is aware of the token ranges handled by its peers. This allows each node to forward a key’s read/write operations to the right set of nodes directly.

当一个节点第一次启动时，它选择它的Token(在虚拟空间的一致哈希节点) 并将节点映射到各自的Token集(Token set)。该映射被持久到磁盘上，最初只包含本地节点和Token集。在不同的节点中存储的映射（节点到token set 的映射）将在协调成员的变化历史的通信过程中一同被协调。因此，划分和布局信息也是基于Gossip协议传播的，因此每个存储节点都了解对等节点所处理的标记范围。这使得每个节点可以直接转发一个key的读/写操作到正确的数据集节点。


####4.8.2外部发现

The mechanism described above could temporarily result in a logically partitioned Dynamo ring. For example, the administrator could contact node A to join A to the ring, then contact node B to join B to the ring. In this scenario, nodes A and B would each consider itself a member of the ring, yet neither would be immediately aware of the other. To prevent logical partitions, some Dynamo nodes play the role of seeds. Seeds are nodes that are discovered via an external mechanism and are known to all nodes. Because all nodes eventually reconcile their membership with a seed, logical partitions are highly unlikely. Seeds can be obtained either from static configuration or from a configuration service. Typically seeds are fully functional nodes in the Dynamo ring.

上述机制可能会暂时导致逻辑分裂的Dynamo环。例如，管理员可以将节点A加入到环，然后将节点B加入环。在这种情况下，节点A和B各自都将认为自己是环的一员，但都不会立即了解到其他的节点(也就是Ａ不知道B的存在，B也不知道A的存在，这叫逻辑分裂)。为了防止逻辑分裂，有些Dynamo节点扮演种子节点的角色。种子的发现(discovered)是通过外部机制来实现的并且所有其他节点都知道(实现中可能直接在配置文件中指定seed node的IP，或者实现一个动态配置服务,seed register)。因为所有的节点，最终都会和种子节点协调成员关系，逻辑分裂是极不可能的。种子可从静态配置或配置服务获得。通常情况下，种子在Dynamo环中是一个全功能节点。


####4.8.3故障检测
Failure detection in Dynamo is used to avoid attempts to communicate with unreachable peers during get() and put() operations and when transferring partitions and hinted replicas. For the purpose of avoiding failed attempts at communication, a purely local notion of failure detection is entirely sufficient: node A may consider node B failed if node B does not respond to node A’s messages (even if B is responsive to node C's messages). In the presence of a steady rate of client requests generating inter- node communication in the Dynamo ring, a node A quickly discovers that a node B is unresponsive when B fails to respond to a message; Node A then uses alternate nodes to service requests that map to B's partitions; A periodically retries B to check for the latter's recovery. In the absence of client requests to drive traffic between two nodes, neither node really needs to know whether the other is reachable and responsive.

Dynamo中，故障检测是用来避免在进行get()和put()操作时尝试联系无法访问节点，同样还用于分区转移(transferring partition)和暗示副本的移交。为了避免在通信失败的尝试，一个纯本地概念的失效检测完全足够了：如果节点B不对节点A的信息进行响应(即使B响应节点C的消息)，节点A可能会认为节点B失败。在一个客户端请求速率相对稳定并产生节点间通信的Dynamo环中，一个节点A可以快速发现另一个节点B不响应时，节点A则使用映射到B的分区的备用节点服务请求，并定期检查节点B后来是否后来被复苏。在没有客户端请求推动两个节点之间流量的情况下，节点双方并不真正需要知道对方是否可以访问或可以响应。

Decentralized failure detection protocols use a simple gossip-style protocol that enable each node in the system to learn about the arrival (or departure) of other nodes. For detailed information on decentralized failure detectors and the parameters affecting their accuracy, the interested reader is referred to [8]. Early designs of Dynamo used a decentralized failure detector to maintain a globally consistent view of failure state. Later it was determined that the explicit node join and leave methods obviates the need for a global view of failure state. This is because nodes are notified of permanent node additions and removals by the explicit node join and leave methods and temporary node failures are detected by the individual nodes when they fail to communicate with others (while forwarding requests).

去中心化的故障检测协议使用一个简单的Gossip式的协议，使系统中的每个节点可以了解其他节点到达(或离开)。有关去中心化的故障探测器和影响其准确性的参数的详细信息，感兴趣的读者可以参考[8]。早期Dynamo的设计使用去中心化的故障检测器以维持一个失败状态的全局性的视图。后来认为，显式的节点加入和离开的方法排除了对一个失败状态的全局性视图的需要。这是因为节点是是可以通过节点的显式加入和离开的方法知道节点永久性(permanent)增加和删除，而短暂的(temporary)节点失效是由独立的节点在他们不能与其他节点通信时发现的(当转发请求时)。


####4.9添加/删除存储节点
When a new node (say X) is added into the system, it gets assigned a number of tokens that are randomly scattered on the ring. For every key range that is assigned to node X, there may be a number of nodes (less than or equal to N) that are currently in charge of handling keys that fall within its token range. Due to the allocation of key ranges to X, some existing nodes no longer have to some of their keys and these nodes transfer those keys to X. Let us consider a simple bootstrapping scenario where node X is added to the ring shown in Figure 2 between A and B. When X is added to the system, it is in charge of storing keys in the ranges (F, G], (G, A] and (A, X]. As a consequence, nodes B, C and D no longer have to store the keys in these respective ranges. Therefore, nodes B, C, and D will offer to and upon confirmation from X transfer the appropriate set of keys. When a node is removed from the system, the reallocation of keys happens in a reverse process.

当一个新的节点(例如X)添加到系统中时，它被分配一些随机散落在环上的Token。对于每一个分配给节点X的key range，当前负责处理落在其key range中的key的节点数可能有好几个(小于或等于N)。由于key range的分配指向X，一些现有的节点不再需要存储他们的一部分key，这些节点将这些key传给X，让我们考虑一个简单的引导(bootstrapping)场景，节点X被添加到图2所示的环中A和B之间，当X添加到系统，它负责的key范围为(F,G]，(G，A]和(A，X]。因此，节点B,C和D都各自有一部分不再需要储存key范围(在X加入前，B负责(F,G], (G,A], (A,B]; C负责(G,A], (A,B], (B,C]; D负责(A,B], (B,C], (C,D]。而在X加入后，B负责(G,A], (A,X], (X,B]; C负责(A,X], (X,B], (B,C]; D负责(X,B], (B,C], (C,D])。因此，节点B，C和D，当收到从X来的确认信号时将供出(offer)适当的key。当一个节点从系统中删除，key的重新分配情况按一个相反的过程进行。

Operational experience has shown that this approach distributes the load of key distribution uniformly across the storage nodes, which is important to meet the latency requirements and to ensure fast bootstrapping. Finally, by adding a confirmation round between the source and the destination, it is made sure that the destination node does not receive any duplicate transfers for a given key range.

实际经验表明，这种方法可以将负载均匀地分布到存储节点，其重要的是满足了延时要求，且可以确保快速引导。最后，在源和目标间增加一轮确认(confirmation round)以确保目标节点不会重复收到任何一个给定的key range转移。

 

##5.实现
In Dynamo, each storage node has three main software components: request coordination, membership and failure detection, and a local persistence engine. All these components are implemented in Java.
在dynamo中，每个存储节点有三个主要的软件组件：请求协调，成员(membership)和故障检测，以及本地持久化引擎。所有这些组件都由Java实现。

Dynamo’s local persistence component allows for different storage engines to be plugged in. Engines that are in use are Berkeley Database (BDB) Transactional Data Store2, BDB Java Edition, MySQL, and an in-memory buffer with persistent backing store. The main reason for designing a pluggable persistence component is to choose the storage engine best suited for an application’s access patterns. For instance, BDB can handle objects typically in the order of tens of kilobytes whereas MySQL can handle objects of larger sizes. Applications choose Dynamo’s local persistence engine based on their object size distribution. The majority of Dynamo’s production instances use BDB Transactional Data Store.
Dynamo的本地持久化组件允许插入不同的存储引擎，如：Berkeley数据库(BDB版本)交易数据存储，BDB Java版，MySQL，以及一个具有持久化后备存储的内存缓冲。设计一个可插拔的持久化组件的主要理由是要按照应用程序的访问模式选择最适合的存储引擎。例如，BDB可以处理的对象通常为几十千字节的数量级，而MySQL能够处理更大尺寸的对象。应用根据其对象的大小分布选择相应的本地持久性引擎。生产中，Dynamo多数使用BDB事务处理数据存储。

The request coordination component is built on top of an event- driven messaging substrate where the message processing pipeline is split into multiple stages similar to the SEDA architecture [24]. All communications are implemented using Java NIO channels. The coordinator executes the read and write requests on behalf of clients by collecting data from one or more nodes (in the case of reads) or storing data at one or more nodes (for writes). Each client request results in the creation of a state machine on the node that received the client request. The state machine contains all the logic for identifying the nodes responsible for a key, sending the requests, waiting for responses, potentially doing retries, processing the replies and packaging the response to the client. Each state machine instance handles exactly one client request. For instance, a read operation implements the following state machine: (i) send read requests to the nodes, (ii) wait for minimum number of required responses, (iii) if too few replies were received within a given time bound, fail the request, (iv) otherwise gather all the data versions and determine the ones to be returned and (v) if versioning is enabled, perform syntactic reconciliation and generate an opaque write context that contains the vector clock that subsumes all the remaining versions. For the sake of brevity the failure handling and retry states are left out.

请求协调组成部分是建立在事件驱动通讯基础上的，其中消息处理管道分为多个阶段类似SEDA的结构[24]。所有的通信都使用Java NIO Channels。协调员执行读取和写入：通过收集从一个或多个节点数据(在读的情况下)，或在一个或多个节点存储的数据(写入)。每个客户的请求中都将导致在收到客户端请求的节点上一个状态机的创建。每一个状态机包含以下逻辑：标识负责一个key的节点，发送请求，等待回应，可能的重试处理，加工和包装返回客户端响应。每个状态机实例只处理一个客户端请求。例如，一个读操作实现了以下状态机：(i)发送读请求到相应节点，(ii)等待所需的最低数量的响应，(iii)如果在给定的时间内收到的响应太少，那么请求失败，(iv)否则，收集所有数据的版本，并确定要返回的版本 (v)如果启用了版本控制，执行语法协调，并产生一个对客户端不透明写上下文，其包括一个涵括所有剩余的版本的矢量时钟。为了简洁起见，没有包含故障处理和重试逻辑。

After the read response has been returned to the caller the state machine waits for a small period of time to receive any outstanding responses. If stale versions were returned in any of the responses, the coordinator updates those nodes with the latest version. This process is called read repair because it repairs replicas that have missed a recent update at an opportunistic time and relieves the anti-entropy protocol from having to do it.

在读取响应返回给调用方后，状态机等待一小段时间以接受任何悬而未决的响应。如果任何响应返回了过时了的(stale)的版本，协调员将用最新的版本更新这些节点(当然是在后台了)。这个过程被称为读修复(read repair)，因为它是用来修复一个在某个时间曾经错过更新操作的副本，同时read repair可以消除不必的反熵操作。

As noted earlier, write requests are coordinated by one of the top N nodes in the preference list. Although it is desirable always to have the first node among the top N to coordinate the writes thereby serializing all writes at a single location, this approach has led to uneven load distribution resulting in SLA violations. This is because the request load is not uniformly distributed across objects. To counter this, any of the top N nodes in the preference list is allowed to coordinate the writes. In particular, since each write usually follows a read operation, the coordinator for a write is chosen to be the node that replied fastest to the previous read operation which is stored in the context information of the request. This optimization enables us to pick the node that has the data that was read by the preceding read operation thereby increasing the chances of getting “read-your-writes” consistency. It also reduces variability in the performance of the request handling which improves the performance at the 99.9 percentile.

如前所述，写请求是由首选列表中某个排名前N的节点来协调的。虽然总是选择前N节点中的第一个节点来协调是可以的，但在单一地点序列化所有的写的做法会导致负荷分配不均，进而导致违反SLA。为了解决这个问题，首选列表中的前N的任何节点都允许协调。特别是，由于写通常跟随在一个读操作之后，写操作的协调员将由节点上最快答复之前那个读操作的节点来担任，这是因为这些信息存储在请求的上下文中(指的是write操作的请求)。这种优化使我们能够选择那个存有同样被之前读操作使用过的数据的节点，从而提高“读你的写”(read-your-writes)一致性(译：我不认为这个描述是有道理的，因为作者这里描述明明是write-follows-read,要了解read-your-writes一致性的读者参见作者另一篇文章:eventually consistent)。它也减少了为了将处理请求的性能提高到99.9百分位时性能表现的差异。


#6.经验与教训

Dynamo is used by several services with different configurations. These instances differ by their version reconciliation logic, and read/write quorum characteristics. The following are the main patterns in which Dynamo is used:
* Business logic specific reconciliation: This is a popular use case for Dynamo. Each data object is replicated across multiple nodes. In case of divergent versions, the client application performs its own reconciliation logic. The shopping cart service discussed earlier is a prime example of this category. Its business logic reconciles objects by merging different versions of a customer’s shopping cart.

* Timestamp based reconciliation: This case differs from the previous one only in the reconciliation mechanism. In case of divergent versions, Dynamo performs simple timestamp based reconciliation logic of “last write wins”; i.e., the object with the largest physical timestamp value is chosen as the correct version. The service that maintains customer’s session information is a good example of a service that uses this mode.

* High performance read engine: While Dynamo is built to be an “always writeable” data store, a few services are tuning its quorum characteristics and using it as a high performance read engine. Typically, these services have a high read request rate and only a small number of updates. In this configuration, typically R is set to be 1 and W to be N. For these services, Dynamo provides the ability to partition and replicate their data across multiple nodes thereby offering incremental scalability. Some of these instances function as the authoritative persistence cache for data stored in more heavy weight backing stores. Services that maintain product catalog and promotional items fit in this category.


Dynamo由几个不同的配置的服务使用。这些实例有着不同的版本协调逻辑和读/写仲裁(quorum)的特性。以下是Dynamo的主要使用模式：

* 业务逻辑特定的协调：这是一个普遍使用的Dynamo案例。每个数据对象被复制到多个节点。在版本发生分岔时，客户端应用程序执行自己的协调逻辑。前面讨论的购物车服务是这一类的典型例子。其业务逻辑是通过合并不同版本的客户的购物车来协调不同的对象。

* 基于时间戳的协调：此案例不同于前一个在于协调机制。在出现不同版本的情况下，Dynamo执行简单的基于时间戳的协调逻辑：“最后的写获胜”，也就是说，具有最大时间戳的对象被选为正确的版本。一些维护客户的会话信息的服务是使用这种模式的很好的例子。

* 高性能读取引擎：虽然Dynamo被构建成一个“永远可写”数据存储，一些服务通过调整其仲裁的特性把它作为一个高性能读取引擎来使用。通常，这些服务有很高的读取请求速率但只有少量的更新操作。在此配置中，通常R是设置为1，且W为N。对于这些服务，Dynamo提供了划分和跨多个节点的复制能力，从而提供增量可扩展性(incremental scalability)。一些这样的实例被当成权威数据缓存用来缓存重量级后台存储的数据。那些保持产品目录及促销项目的服务适合此种类别。

The main advantage of Dynamo is that its client applications can tune the values of N, R and W to achieve their desired levels of performance, availability and durability. For instance, the value of N determines the durability of each object. A typical value of N used by Dynamo’s users is 3.

Dynamo的主要优点是它的客户端应用程序可以调的N，R和W的值，以实现其期待的性能, 可用性和耐用性的水平。例如，N的值决定了每个对象的耐久性。Dynamo用户使用的一个典型的N值是3。

The values of W and R impact object availability, durability and consistency. For instance, if W is set to 1, then the system will never reject a write request as long as there is at least one node in the system that can successfully process a write request. However, low values of W and R can increase the risk of inconsistency as write requests are deemed successful and returned to the clients even if they are not processed by a majority of the replicas. This also introduces a vulnerability window for durability when a write request is successfully returned to the client even though it has been persisted at only a small number of nodes.

W和R影响对象的可用性，耐用性和一致性。举例来说，如果W设置为1，只要系统中至少有一个节点活就可以成功地处理一个写请求，那么系统将永远不会拒绝写请求。不过，低的W和R值会增加不一致性的风险，因为写请求被视为成功并返回到客户端，即使他们还未被大多数副本处理。这也引入了一个耐用性漏洞(vulnerability)窗口：即使它只是在少数几个节点上持久化了但写入请求成功返回到客户端。

Traditional wisdom holds that durability and availability go hand- in-hand. However, this is not necessarily true here. For instance, the vulnerability window for durability can be decreased by increasing W. This may increase the probability of rejecting requests (thereby decreasing availability) because more storage hosts need to be alive to process a write request.

传统的观点认为，耐用性和可用性关系总是非常紧密(hand-in-hand手牵手^-^)。但是，这并不一定总是真的。例如，耐用性漏洞窗口可以通过增加W来减少，但这将增加请求被拒绝的机率(从而减少可用性)，因为为处理一个写请求需要更多的存储主机需要活着。

The common (N,R,W) configuration used by several instances of Dynamo is (3,2,2). These values are chosen to meet the necessary levels of performance, durability, consistency, and availability SLAs.

被好几个Dynamo实例采用的(N，R，W)配置通常为(3,2,2)。选择这些值是为满足性能，耐用性，一致性和可用性SLAs的需求。

All the measurements presented in this section were taken on a live system operating with a configuration of (3,2,2) and running a couple hundred nodes with homogenous hardware configurations. As mentioned earlier, each instance of Dynamo contains nodes that are located in multiple datacenters. These datacenters are typically connected through high speed network links. Recall that to generate a successful get (or put) response R (or W) nodes need to respond to the coordinator. Clearly, the network latencies between datacenters affect the response time and the nodes (and their datacenter locations) are chosen such that the applications target SLAs are met.

所有在本节中测量的是一个在线系统，其工作在(3,2,2)配置并运行在几百个同质硬件配置上。如前所述，每一个实例包含位于多个数据中心的Dynamo节点。这些数据中心通常是通过高速网络连接。回想一下，产生一个成功的get(或put)响应，R(或W)个节点需要响应协调员。显然，数据中心之间的网络延时会影响响应时间，因此节点(及其数据中心位置)的选择要使得应用的目标SLAs得到满足。

##6.1平衡性能和耐久性

While Dynamo’s principle design goal is to build a highly available data store, performance is an equally important criterion in Amazon’s platform. As noted earlier, to provide a consistent customer experience, Amazon’s services set their performance targets at higher percentiles (such as the 99.9th or 99.99th percentiles). A typical SLA required of services that use Dynamo is that 99.9% of the read and write requests execute within 300ms.

虽然Dynamo的主要的设计目标是建立一个高度可用的数据存储，性能是在Amazon平台中是一个同样重要的衡量标准。如前所述，为客户提供一致的客户体验，Amazon的服务定在较高的百分位(如99.9或99.99)，一个典型的使用Dynamo的服务的SLA要求99.9％的读取和写入请求在300毫秒内完成。

Since Dynamo is run on standard commodity hardware components that have far less I/O throughput than high-end enterprise servers, providing consistently high performance for read and write operations is a non-trivial task. The involvement of multiple storage nodes in read and write operations makes it even more challenging, since the performance of these operations is limited by the slowest of the R or W replicas. Figure 4 shows the average and 99.9th percentile latencies of Dynamo’s read and write operations during a period of 30 days. As seen in the figure, the latencies exhibit a clear diurnal pattern which is a result of the diurnal pattern in the incoming request rate (i.e., there is a significant difference in request rate between the daytime and night). Moreover, the write latencies are higher than read latencies obviously because write operations always results in disk access. Also, the 99.9th percentile latencies are around 200 ms and are an order of magnitude higher than the averages. This is because the 99.9th percentile latencies are affected by several factors such as variability in request load, object sizes, and locality patterns.


由于Dynamo是运行在标准的日用级硬件组件上，这些组件的I/O吞吐量远比不上高端企业级服务器，因此提供一致性的高性能的读取和写入操作并不是一个简单的任务。再加上涉及到多个存​​储节点的读取和写入操作，让我们更加具有挑战性，因为这些操作的性能是由最慢的R或W副本限制的。图4显示了Dynamo为期30天的读/写的平均和99.9百分位的延时。正如图中可以看出，延时表现出明显的昼夜模式这是因为进来的请求速率存在昼夜模式的结果造成的(即请求速率在白天和黑夜有着显着差异)。此外，写延时明显高于读取延时，因为写操作总是导致磁盘访问。此外，99.9百分位的延时大约是200毫秒，比平均水平高出一个数量级。这是因为99.9百分位的延时受几个因素，如请求负载，对象大小和位置格局的变化影响。

![haroopad icon](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/algorithm/dynamo-4.jpg)
图4：读，写操作的平均和99.9百分点延时，2006年12月高峰时的请求。在X轴的刻度之间的间隔相当于连续12小时。延时遵循昼夜模式类似请求速率,99.9百分点比平均水平高出一个数量级。

While this level of performance is acceptable for a number of services, a few customer-facing services required higher levels of performance. For these services, Dynamo provides the ability to trade-off durability guarantees for performance. In the optimization each storage node maintains an object buffer in its main memory. Each write operation is stored in the buffer and gets periodically written to storage by a writer thread. In this scheme, read operations first check if the requested key is present in the buffer. If so, the object is read from the buffer instead of the storage engine.

虽然这种性能水平是可以被大多数服务所接受，一些面向客户的服务需要更高的性能。针对这些服务，Dynamo能够牺牲持久性来保证性能。在这个优化中，每个存储节点维护一个内存中的对象缓冲区(BigTable 中的memtable)。每次写操作都存储在缓冲区，“写”线程定期将缓冲写到存储中。在这个方案中，读操作首先检查请求的 key 是否存在于缓冲区。如果是这样，对象是从缓冲区读取，而不是存储引擎。

This optimization has resulted in lowering the 99.9th percentile latency by a factor of 5 during peak traffic even for a very small buffer of a thousand objects (see Figure 5). Also, as seen in the figure, write buffering smoothes out higher percentile latencies. Obviously, this scheme trades durability for performance. In this scheme, a server crash can result in missing writes that were queued up in the buffer. To reduce the durability risk, the write operation is refined to have the coordinator choose one out of the N replicas to perform a “durable write”. Since the coordinator waits only for W responses, the performance of the write operation is not affected by the performance of the durable write operation performed by a single replica.

这种优化的结果是99.9百位在流量高峰期间的延时降低达5倍之多，即使是一千个对象(参见图5)的非常小的缓冲区。此外，如图中所示，写缓冲在较高百分位具有平滑延时。显然，这个方案是平衡耐久性来提高性能的。在这个方案中，服务器崩溃可能会导致写操作丢失，即那些在缓冲区队列中的写(还未持久化到存储中的写)。为了减少耐用性风险，更细化的写操作要求协调员选择N副本中的一个执行“持久写”。由于协调员只需等待W个响应(译，这里讨论的这种情况包含W-1个缓冲区写，1个持久化写)，写操作的性能不会因为单一一个副本的持久化写而受到影响。



![haroopad icon](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/algorithm/dynamo-5.jpg)

图5：24小时内的99.9百分位延时缓冲和非缓冲写的性能比较。在x轴的刻度之间的间隔连续为一小时。


###6.2确保均匀的负载分布

Dynamo uses consistent hashing to partition its key space across its replicas and to ensure uniform load distribution. A uniform key distribution can help us achieve uniform load distribution assuming the access distribution of keys is not highly skewed. In particular, Dynamo’s design assumes that even where there is a significant skew in the access distribution there are enough keys in the popular end of the distribution so that the load of handling popular keys can be spread across the nodes uniformly through partitioning. This section discusses the load imbalance seen in Dynamo and the impact of different partitioning strategies on load distribution.

Dynamo采用一致性的散列将key space(键空间)分布在其所有的副本上，并确保负载均匀分布。假设对key的访问分布不会高度偏移，一个统一的key分配可以帮助我们达到均匀的负载分布。特别地, Dynamo设计假定，即使访问的分布存在显着偏移，只要在流行的那端(popular end)有足够多的keys，那么对那些流行的key的处理的负载就可以通过partitioning均匀地分散到各个节点。本节讨论Dynamo中所出现负载不均衡和不同的划分策略对负载分布的影响。


To study the load imbalance and its correlation with request load, the total number of requests received by each node was measured for a period of 24 hours - broken down into intervals of 30 minutes. In a given time window, a node is considered to be “in- balance”, if the node’s request load deviates from the average load by a value a less than a certain threshold (here 15%). Otherwise the node was deemed “out-of-balance”. Figure 6 presents the fraction of nodes that are “out-of-balance” (henceforth, “imbalance ratio”) during this time period. For reference, the corresponding request load received by the entire system during this time period is also plotted. As seen in the figure, the imbalance ratio decreases with increasing load. For instance, during low loads the imbalance ratio is as high as 20% and during high loads it is close to 10%. Intuitively, this can be explained by the fact that under high loads, a large number of popular keys are accessed and due to uniform distribution of keys the load is evenly distributed. However, during low loads (where load is 1/8th of the measured peak load), fewer popular keys are accessed, resulting in a higher load imbalance.
为了研究负载不平衡与请求负载的相关性，通过测量各个节点在24小时内收到的请求总数-细分为30分钟一段。在一个给定的时间窗口，如果该节点的请求负载偏离平均负载没有超过某个阈值(这里15％)，认为一个节点被认为是“平衡的”。否则，节点被认为是“失去平衡”。图6给出了一部分在这段时间内“失去平衡”的节点(以下简称“失衡比例”)。作为参考，整个系统在这段时间内收到的相应的请求负载也被绘制。正如图所示，不平衡率随着负载的增加而下降。例如，在低负荷时，不平衡率高达20％，在高负荷高接近10％。直观地说，这可以解释为，在高负荷时大量流行键(popular key)访问且由于key的均匀分布，负载最终均匀分布。然而，在(其中负载为高峰负载的八分之一)低负载下，当更少的流行键被访问，将导致一个比较高的负载不平衡。

![haroopad icon](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/algorithm/dynamo-6.jpg)

图6：部分失去平衡的节点(即节点的请求负载高于系统平均负载的某一阈值)和其相应的请求负载。

X轴刻度间隔相当于一个30分钟的时间。

This section discusses how Dynamo’s partitioning scheme has evolved over time and its implications on load distribution.

本节讨论Dynamo的划分方案(partitioning scheme)是如何随着时间和负载分布的影响进行演化的。

Strategy 1: T random tokens per node and partition by token value: This was the initial strategy deployed in production (and described in Section 4.2). In this scheme, each node is assigned T tokens (chosen uniformly at random from the hash space). The tokens of all nodes are ordered according to their values in the hash space. Every two consecutive tokens define a range. The last token and the first token form a range that "wraps" around from the highest value to the lowest value in the hash space. Because the tokens are chosen randomly, the ranges vary in size. As nodes join and leave the system, the token set changes and consequently the ranges change. Note that the space needed to maintain the membership at each node increases linearly with the number of nodes in the system.

策略1：每个节点T个随机Token和基于Token值进行分割：这是最早部署在生产环境的策略(在4.2节中描述)。在这个方案中，每个节点被分配T 个Tokens(从哈希空间随机均匀地选择)。所有节点的token，是按照其在哈希空间中的值进行排序的。每两个连续的Token定义一个范围。最后的Token与最开始的Token构成一区域(range)：从哈希空间中最大值绕(wrap)到最低值。由于Token是随机选择，范围大小是可变的。节点加入和离开系统导致Token集的改变，最终导致ranges的变化，请注意，每个节点所需的用来维护系统的成员的空间与系统中节点的数目成线性关系。

While using this strategy, the following problems were encountered. First, when a new node joins the system, it needs to “steal” its key ranges from other nodes. However, the nodes handing the key ranges off to the new node have to scan their local persistence store to retrieve the appropriate set of data items. Note that performing such a scan operation on a production node is tricky as scans are highly resource intensive operations and they need to be executed in the background without affecting the customer performance. This requires us to run the bootstrapping task at the lowest priority. However, this significantly slows the bootstrapping process and during busy shopping season, when the nodes are handling millions of requests a day, the bootstrapping has taken almost a day to complete. Second, when a node joins/leaves the system, the key ranges handled by many nodes change and the Merkle trees for the new ranges need to be recalculated, which is a non-trivial operation to perform on a production system. Finally, there was no easy way to take a snapshot of the entire key space due to the randomness in key ranges, and this made the process of archival complicated. In this scheme, archiving the entire key space requires us to retrieve the keys from each node separately, which is highly inefficient.

在使用这一策略时，遇到了以下问题。首先，当一个新的节点加入系统时，它需要“窃取”(steal)其他节点的键范围。然而，这些需要移交key ranges给新节点的节点必须扫描他们的本地持久化存储来得到适当的数据项。请注意，在生产节点上执行这样的扫描操作是非常复杂，因为扫描是资源高度密集的操作，他们需要在后台执行，而不至于影响客户的性能。这就要求我们必须将引导工作设置为最低的优先级。然而，这将大大减缓了引导过程，在繁忙的购物季节，当节点每天处理数百万的请求时，引导过程可能需要几乎一天才能完成。第二，当一个节点加入/离开系统，由许多节点处理的key range的变化以及新的范围的MertkleTree需要重新计算，在生产系统上，这不是一个简单的操作。最后，由于key range的随机性，没有一个简单的办法为整个key space做一个快照，这使得归档过程复杂化。在这个方案中，归档整个key space 需要分别检索每个节点的key，这是非常低效的。

The fundamental issue with this strategy is that the schemes for data partitioning and data placement are intertwined. For instance, in some cases, it is preferred to add more nodes to the system in order to handle an increase in request load. However, in this scenario, it is not possible to add nodes without affecting data partitioning. Ideally, it is desirable to use independent schemes for partitioning and placement. To this end, following strategies were evaluated:

Strategy 2: T random tokens per node and equal sized partitions:
In this strategy, the hash space is divided into Q equally sized partitions/ranges and each node is assigned T random tokens. Q is usually set such that Q >> N and Q >> S*T, where S is the number of nodes in the system. In this strategy, the tokens are only used to build the function that maps values in the hash space to the ordered lists of nodes and not to decide the partitioning. A partition is placed on the first N unique nodes that are encountered while walking the consistent hashing ring clockwise from the end of the partition. Figure 7 illustrates this strategy for N=3. In this example, nodes A, B, C are encountered while walking the ring from the end of the partition that contains key k1. The primary advantages of this strategy are: (i) decoupling of partitioning and partition placement, and (ii) enabling the possibility of changing the placement scheme at runtime.

这个策略的根本问题是，数据划分和数据安置的计划交织在一起。例如，在某些情况下，最好是添加更多的节点到系统，以应对处理请求负载的增加。但是，在这种情况下，添加节点(导致数据安置)不可能不影响数据划分。理想的情况下，最好使用独立划分和安置计划。为此，对以下策略进行了评估：

策略2：每个节点T个随机token和同等大小的分区：在此策略中，节点的哈希空间分为Q个同样大小的分区/范围，每个节点被分配T个随机Token。Q是通常设置使得Q>>N和Q>>S*T，其中S为系统的节点个数。在这一策略中，Token只是用来构造一个映射函数该函数将哈希空间的值映射到一个有序列的节点列表，而不决定分区。分区是放置在从分区的末尾开始沿着一致性hash环顺时针移动遇到的前N个独立的节点上。图7说明了这一策略当N=3时的情况。在这个例子中，节点A，B，C是从分区的末尾开始沿着一致性hash环顺时针移动遇到的包含key K1的节点。这一策略的主要优点是：(i)划分和分区布局脱耦 (ii)使得在运行时改变安置方案成为可能。

![haroopad icon](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/algorithm/dynamo-7.jpg)

图7：三个策略的分区和key的位置。甲，乙，丙描述三个独立的节点，形成keyk1在一致性哈希环上的首选列表(N＝3)。阴影部分表示节点A，B和C形式的首选列表负责的keyrangee。黑色箭头标明各节点的Token的位置。

Strategy 3: Q/S tokens per node, equal-sized partitions: Similar to strategy 2, this strategy divides the hash space into Q equally sized partitions and the placement of partition is decoupled from the partitioning scheme. Moreover, each node is assigned Q/S tokens where S is the number of nodes in the system. When a node leaves the system, its tokens are randomly distributed to the remaining nodes such that these properties are preserved. Similarly, when a node joins the system it "steals" tokens from nodes in the system in a way that preserves these properties.


策略3：每个节点Q/S个Token，大小相等的分区：类似策略2，这一策略空间划分成同样大小为Q的散列分区，以及分区布局(placement of partition)与划分方法(partitioning scheme)脱钩。此外，每个节点被分配Q/S个Token其中S是系统的节点数。当一个节点离开系统，为使这些属性被保留，它的Token随机分发到其他节点。同样，当一个节点加入系统，新节点将通过一种可以保留这种属性的方式从系统的其他节点“偷”Token。

The efficiency of these three strategies is evaluated for a system with S=30 and N=3. However, comparing these different strategies in a fair manner is hard as different strategies have different configurations to tune their efficiency. For instance, the load distribution property of strategy 1 depends on the number of tokens (i.e., T) while strategy 3 depends on the number of partitions (i.e., Q). One fair way to compare these strategies is to evaluate the skew in their load distribution while all strategies use the same amount of space to maintain their membership information. For instance, in strategy 1 each node needs to maintain the token positions of all the nodes in the ring and in strategy 3 each node needs to maintain the information regarding the partitions assigned to each node.
对这三个策略的效率评估使用S=30和N=3配置的系统。然而，以一个比较公平的方式这些不同的策略是很难的，因为不同的策略有不同的配置来调整他们的效率。例如，策略1取决于负荷的适当分配(即T)，而策略3信赖于分区的个数(即Q)。一个公平的比较方式是在所有策略中使用相同数量的空间来维持他们的成员信息时，通过评估负荷分布的偏斜. 例如，策略1每个节点需要维护所有环内的Token位置，策略3每个节点需要维护分配到每个节点的分区信息。

In our next experiment, these strategies were evaluated by varying the relevant parameters (T and Q). The load balancing efficiency of each strategy was measured for different sizes of membership information that needs to be maintained at each node, where Load balancing efficiency is defined as the ratio of average number of requests served by each node to the maximum number of requests served by the hottest node.

在我们的下一个实验，通过改变相关的参数(T 和 Q),对这些策略进行了评价。每个策略的负载均衡的效率是根据每个节点需要维持的成员信息的大小的不同来测量，负载平衡效率是指每个节点服务的平均请求数与最忙(hottest)的节点服务的最大请求数之比。

The results are given in Figure 8. As seen in the figure, strategy 3 achieves the best load balancing efficiency and strategy 2 has the worst load balancing efficiency. For a brief time, Strategy 2 served as an interim setup during the process of migrating Dynamo instances from using Strategy 1 to Strategy 3. Compared to Strategy 1, Strategy 3 achieves better efficiency and reduces the size of membership information maintained at each node by three orders of magnitude. While storage is not a major issue the nodes gossip the membership information periodically and as such it is desirable to keep this information as compact as possible. In addition to this, strategy 3 is advantageous and simpler to deploy for the following reasons: (i) Faster bootstrapping/recovery: Since partition ranges are fixed, they can be stored in separate files, meaning a partition can be relocated as a unit by simply transferring the file (avoiding random accesses needed to locate specific items). This simplifies the process of bootstrapping and recovery. (ii) Ease of archival: Periodical archiving of the dataset is a mandatory requirement for most of Amazon storage services. Archiving the entire dataset stored by Dynamo is simpler in strategy 3 because the partition files can be archived separately. By contrast, in Strategy 1, the tokens are chosen randomly and, archiving the data stored in Dynamo requires retrieving the keys from individual nodes separately and is usually inefficient and slow. The disadvantage of strategy 3 is that changing the node membership requires coordination in order to preserve the properties required of the assignment.

结果示于图8。正如图中看到，策略3达到最佳的负载平衡效率，而策略2最差负载均衡的效率。一个短暂的时期，在将Dynamo实例从策略1到策略3的迁移过程中，策略2曾作为一个临时配置。相对于策略1，策略3达到更好的效率并且在每个节点需要维持的信息的大小规模降低了三个数量级。虽然存储不是一个主要问题，但节点间周期地Gossip成员信息，因此最好是尽可能保持这些信息紧凑。除了这个，策略3有利于且易于部署，理由如下：(i)更快的bootstrapping/恢复：由于分区范围是固定的，它们可以被保存在单独的文件，这意味着一个分区可以通过简单地转移文件并作为一个单位重新安置(避免随机访问需要定位具体项目)。这简化了引导和恢复过程。(ii)易于档案：对数据集定期归档是Amazon存储服务提出的强制性要求。Dynamo在策略3下归档整个数据集很简单，因为分区的文件可以被分别归档。相反，在策略1，Token是随机选取的，归档存储的数据需要分别检索各个节点的key，这通常是低效和缓慢的。策略3的缺点是，为维护分配所需的属性改变节点成员时需要协调，。

![haroopad icon](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/algorithm/dynamo-8.jpg)
图8：比较30个维持相同数量的元数据节的点,N=3的系统不同策略的负载分布效率。系统的规模和副本的数量的值是按照我们部署的大多数服务的典型配置。



###6.3不同版本：何时以及有多少？
Divergent Versions: When and How Many?

As noted earlier, Dynamo is designed to tradeoff consistency for availability. To understand the precise impact of different failures on consistency, detailed data is required on multiple factors: outage length, type of failure, component reliability, workload etc. Presenting these numbers in detail is outside of the scope of this paper. However, this section discusses a good summary metric: the number of divergent versions seen by the application in a live production environment.

如前所述，Dynamo被设计成为获得可用性而牺牲了一致性。为了解不同的一致性失败导致的确切影响，多方面的详细的数据是必需的：中断时长，失效类型，组件可靠性，负载量等。详细地呈现所有这些数字超出本文的范围。不过，本节讨论了一个很好的简要的度量尺度：在现场生产环境中的应用所出现的不同版本的数量。

Divergent versions of a data item arise in two scenarios. The first is when the system is facing failure scenarios such as node failures, data center failures, and network partitions. The second is when the system is handling a large number of concurrent writers to a single data item and multiple nodes end up coordinating the updates concurrently. From both a usability and efficiency perspective, it is preferred to keep the number of divergent versions at any given time as low as possible. If the versions cannot be syntactically reconciled based on vector clocks alone, they have to be passed to the business logic for semantic reconciliation. Semantic reconciliation introduces additional load on services, so it is desirable to minimize the need for it.


不同版本的数据项出现在两种情况下。首先是当系统正面临着如节点失效故障的情况下, 数据中心的故障和网络分裂。二是当系统的并发处理大量写单个数据项，并且最终多个节点同时协调更新操作。无论从易用性和效率的角度来看，都应首先确保在任何特定时间内不同版本的数量尽可能少。如果版本不能单独通过矢量时钟在语法上加以协调，他们必须被传递到业务逻辑层进行语义协调。语义协调给服务应用引入了额外的负担，因此应尽量减少它的需要。

In our next experiment, the number of versions returned to the shopping cart service was profiled for a period of 24 hours. During this period, 99.94% of requests saw exactly one version; 0.00057% of requests saw 2 versions; 0.00047% of requests saw 3 versions and 0.00009% of requests saw 4 versions. This shows that divergent versions are created rarely.

Experience shows that the increase in the number of divergent versions is contributed not by failures but due to the increase in number of concurrent writers. The increase in the number of concurrent writes is usually triggered by busy robots (automated client programs) and rarely by humans. This issue is not discussed in detail due to the sensitive nature of the story.

在我们的下一个实验中，返回到购物车服务的版本数量是基于24小时为周期来剖析的。在此期间，99.94％的请求恰好看到了1个版本。0.00057％的请求看到2个版本，0.00047％的请求看到3个版本和0.00009％的请求看到4个版本。这表明，不同版本创建的很少。

经验表明，不同版本的数量的增加不是由于失败而是由于并发写操作的数量增加造成的。数量递增的并发写操作通常是由忙碌的机器人(busy robot-自动化的客户端程序)导致而很少是人为触发。由于敏感性，这个问题还没有详细讨论。

 

###6.4客户端驱动或服务器驱动协调

As mentioned in Section 5, Dynamo has a request coordination component that uses a state machine to handle incoming requests. Client requests are uniformly assigned to nodes in the ring by a load balancer. Any Dynamo node can act as a coordinator for a read request. Write requests on the other hand will be coordinated by a node in the key’s current preference list. This restriction is due to the fact that these preferred nodes have the added responsibility of creating a new version stamp that causally subsumes the version that has been updated by the write request. Note that if Dynamo’s versioning scheme is based on physical timestamps, any node can coordinate a write request.

如第5条所述，Dynamo有一个请求协调组件，它使用一个状态机来处理进来的请求。客户端的请求均匀分配到环上的节点是由负载平衡器完成的。Dynamo的任何节点都可以充当一个读请求协调员。另一方面，写请求将由key的首选列表中的节点来协调。此限制是由于这一事实－－这些首选节点具有附加的责任：即创建一个新的版本标识，使之与写请求更新的版本建立因果关系(译：呜呜，这个很难！Causally subsumes)。请注意，如果Dynamo的版本方案是建基于物理时间戳(译：一个在本文中没解释的概念：[Timestamp Semantics and Representation]Many database management systems and operating systems provide support for time values. This support is present at both the logical and physical levels. The logical level is the user's view of the time values and the query level operations permitted on those values, while the physical level concerns the bit layout of the time values and the bit level operations on those values. The physical level serves as a platform for the logical level but is inaccessible to the average user.)的话，任何节点都可以协调一个写请求。

An alternative approach to request coordination is to move the state machine to the client nodes. In this scheme client applications use a library to perform request coordination locally. A client periodically picks a random Dynamo node and downloads its current view of Dynamo membership state. Using this information the client can determine which set of nodes form the preference list for any given key. Read requests can be coordinated at the client node thereby avoiding the extra network hop that is incurred if the request were assigned to a random Dynamo node by the load balancer. Writes will either be forwarded to a node in the key’s preference list or can be coordinated locally if Dynamo is using timestamps based versioning.

另一种请求协调的方法是将状态机移到客户端节点。在这个方案中，客户端应用程序使用一个库在本地执行请求协调。客户端定期随机选取一个节点，并下载其当前的Dynamo成员状态视图。利用这些信息,客户端可以从首选列表中为给定的key选定相应的节点集。读请求可以在客户端节点进行协调，从而避免了额外一跳的网络开销(network hop)，比如，如果请求是由负载平衡器分配到一个随机的Dynamo节点，这种情况会招致这样的额外一跳。如果Dynamo使用基于时间戳的版本机制，写要么被转发到在key的首选列表中的节点，也可以在本地协调。

An important advantage of the client-driven coordination approach is that a load balancer is no longer required to uniformly distribute client load. Fair load distribution is implicitly guaranteed by the near uniform assignment of keys to the storage nodes. Obviously, the efficiency of this scheme is dependent on how fresh the membership information is at the client. Currently clients poll a random Dynamo node every 10 seconds for membership updates. A pull based approach was chosen over a push based one as the former scales better with large number of clients and requires very little state to be maintained at servers regarding clients. However, in the worst case the client can be exposed to stale membership for duration of 10 seconds. In case, if the client detects its membership table is stale (for instance, when some members are unreachable), it will immediately refresh its membership information.

一个客户端驱动的协调方式的重要优势是不再需要一个负载平衡器来均匀分布客户的负载。公平的负载分布隐含地由近乎平均的分配key到存储节点的方式来保证的。显然，这个方案的有效性是信赖于客户端的成员信息的新鲜度的。目前客户每10秒随机地轮循一Dynamo节点来更新成员信息。一个基于抽取(pull)而不是推送(push)的方被采用，因为前一种方法在客户端数量比较大的情况下扩展性好些，并且服务端只需要维护一小部分关于客户端的状态信息。然而，在最坏的情况下，客户端可能持有长达10秒的陈旧的成员信息。如果客户端检测其成员列表是陈旧的(例如，当一些成员是无法访问)情况下，它会立即刷新其成员信息。

Table 2 shows the latency improvements at the 99.9th percentile and averages that were observed for a period of 24 hours using client-driven coordination compared to the server-driven approach. As seen in the table, the client-driven coordination approach reduces the latencies by at least 30 milliseconds for 99.9th percentile latencies and decreases the average by 3 to 4 milliseconds. The latency improvement is because the client- driven approach eliminates the overhead of the load balancer and the extra network hop that may be incurred when a request is assigned to a random node. As seen in the table, average latencies tend to be significantly lower than latencies at the 99.9th percentile. This is because Dynamo’s storage engine caches and write buffer have good hit ratios. Moreover, since the load balancers and network introduce additional variability to the response time, the gain in response time is higher for the 99.9th percentile than the average.


表2显示了24小时内观察到的，对比于使用服务端协调方法，使用客户端驱动的协调方法，在99.9百分位延时和平均延时的改善。如表所示，客户端驱动的协调方法，99.9百分位减少至少30毫秒的延时，以及降低了3到4毫秒的平均延时。延时的改善是因为客户端驱动的方法消除了负载平衡器额外的开销以及网络一跳，这在请求被分配到一个随机节点时将导致的开销。如表所示，平均延时往往要明显比99.9百分位延时低。这是因为Dynamo的存储引擎缓存和写缓冲器具有良好的命中率。此外，由于负载平衡器和网络引入额外的对响应时间的可变性，在响应时间方面，99.9th百分位这这种情况下(即使用负载平衡器)获得好处比平均情况下要高。


|99.9th百分读延时(毫秒)|99.9th百分写入延时(毫秒)|平均读取延时时间(毫秒)|平均写入延时(毫秒)|
|---|---|---|---|---|
|服务器驱动|68.9|68.5|3.9|4.02|
|客户驱动|30.4|30.4|1.55|1.9|
表二：客户驱动和服务器驱动的协调方法的性能。

###6.5权衡后台和前台任务
Balancing background vs. foreground tasks

Each node performs different kinds of background tasks for replica synchronization and data handoff (either due to hinting or adding/removing nodes) in addition to its normal foreground put/get operations. In early production settings, these background tasks triggered the problem of resource contention and affected the performance of the regular put and get operations. Hence, it became necessary to ensure that background tasks ran only when the regular critical operations are not affected significantly. To this end, the background tasks were integrated with an admission control mechanism. Each of the background tasks uses this controller to reserve runtime slices of the resource (e.g. database),shared across all background tasks. A feedback mechanism based on the monitored performance of the foreground tasks is employed to change the number of slices that are available to the background tasks.

每个节点除了正常的前台put/get操作，还将执行不同的后台任务，如数据的副本的同步和数据移交(handoff)(由于暗示(hinting)或添加/删除节点导致)。在早期的生产设置中，这些后台任务触发了资源争用问题，影响了正常的put和get操作的性能。因此，有必要确保后台任务只有在不会显著影响正常的关键操作时运行。为了达到这个目的，所有后台任务都整合了管理控制机制。每个后台任务都使用此控制器，以预留所有后台任务共享的时间片资源(如数据库)。采用一个基于对前台任务进行监控的反馈机制来控制用于后台任务的时间片数。

The admission controller constantly monitors the behavior of resource accesses while executing a "foreground" put/get operation. Monitored aspects include latencies for disk operations, failed database accesses due to lock-contention and transaction timeouts, and request queue wait times. This information is used to check whether the percentiles of latencies (or failures) in a given trailing time window are close to a desired threshold. For example, the background controller checks to see how close the 99th percentile database read latency (over the last 60 seconds) is to a preset threshold (say 50ms). The controller uses such comparisons to assess the resource availability for the foreground operations. Subsequently, it decides on how many time slices will be available to background tasks, thereby using the feedback loop to limit the intrusiveness of the background activities. Note that a similar problem of managing background tasks has been studied in [4].


管理控制器在进行前台put/get操作时不断监测资源访问的行为，监测数据包括对磁盘操作延时，由于锁争用导致的失败的数据库访问和交易超时，以及请求队列等待时间。此信息是用于检查在特定的后沿时间窗口延时(或失败)的百分位是否接近所期望的阀值。例如，背景控制器检查，看看数据库的99百分位的读延时(在最后60秒内)与预设的阈值(比如50毫秒)的接近程度。该控制器采用这种比较来评估前台业务的资源可用性。随后，它决定多少时间片可以提供给后台任务，从而利用反馈环来限制背景活动的侵扰。请注意，一个与后台任务管理类似的问题已经在[4]有所研究。

 

###6.6讨论

This section summarizes some of the experiences gained during the process of implementation and maintenance of Dynamo. Many Amazon internal services have used Dynamo for the past two years and it has provided significant levels of availability to its applications. In particular, applications have received successful responses (without timing out) for 99.9995% of its requests and no data loss event has occurred to date.

本节总结了在实现和维护Dynamo过程中获得的一些经验。很多Amazon的内部服务在过去二年中已经使用了Dynamo，它给应用提供了很高级别的可用性。特别是，应用程序的99.9995％的请求都收到成功的响应(无超时)，到目前为止，无数据丢失事件发生。

Moreover, the primary advantage of Dynamo is that it provides the necessary knobs using the three parameters of (N,R,W) to tune their instance based on their needs.. Unlike popular commercial data stores, Dynamo exposes data consistency and reconciliation logic issues to the developers. At the outset, one may expect the application logic to become more complex. However, historically, Amazon’s platform is built for high availability and many applications are designed to handle different failure modes and inconsistencies that may arise. Hence, porting such applications to use Dynamo was a relatively simple task. For new applications that want to use Dynamo, some analysis is required during the initial stages of the development to pick the right conflict resolution mechanisms that meet the business case appropriately. Finally, Dynamo adopts a full membership model where each node is aware of the data hosted by its peers. To do this, each node actively gossips the full routing table with other nodes in the system. This model works well for a system that contains couple of hundreds of nodes. However, scaling such a design to run with tens of thousands of nodes is not trivial because the overhead in maintaining the routing table increases with the system size. This limitation might be overcome by introducing hierarchical extensions to Dynamo. Also, note that this problem is actively addressed by O(1) DHT systems(e.g., [14]).


此外，Dynamo的主要优点是，它提供了使用三个参数的(N，R，W)，根据自己的需要来调整它们的实例。不同于流行的商业数据存储，Dynamo将数据一致性与协调的逻辑问题暴露给开发者。开始，人们可能会认为应用程序逻辑会变得更加复杂。然而，从历史上看，Amazon平台都为高可用性而构建，且许多应用内置了处理不同的失效模式和可能出现的不一致性。因此，移植这些应用程序到使用Dynamo是一个相对简单的任务。对于那些希望使用Dynamo的应用，需要开发的初始阶段做一些分析，以选择正确的冲突的协调机制以适当地满足业务情况。最后，Dynamo采用全成员(full membership)模式，其中每个节点都知道其对等节点承载的数据。要做到这一点，每个节点都需要积极地与系统中的其他节点Gossip完整的路由表。这种模式在一个包含数百个节点的系统中运作良好，然而，扩展这样的设计以运行成千上万节点并不容易，因为维持路由表的开销将随着系统的大小的增加而增加。克服这种限制可能需要通过对 Dynamo引入分层扩展。此外，请注意这个问题正在积极由O(1)DHT的系统解决(例如，[14])。

 

##7结论
This paper described Dynamo, a highly available and scalable data store, used for storing state of a number of core services of Amazon.com’s e-commerce platform. Dynamo has provided the desired levels of availability and performance and has been successful in handling server failures, data center failures and network partitions. Dynamo is incrementally scalable and allows service owners to scale up and down based on their current
request load. Dynamo allows service owners to customize their storage system to meet their desired performance, durability and consistency SLAs by allowing them to tune the parameters N, R, and W.
The production use of Dynamo for the past year demonstrates that decentralized techniques can be combined to provide a single highly-available system. Its success in one of the most challenging application environments shows that an eventual- consistent storage system can be a building block for highly- available applications.
ACKNOWLEDGEMENTS
The authors would like to thank Pat Helland for his contribution to the initial design of Dynamo. We would also like to thank Marvin Theimer and Robert van Renesse for their comments. Finally, we would like to thank our shepherd, Jeff Mogul, for his detailed comments and inputs while preparing the camera ready version that vastly improved the quality of the paper.

本文介绍了Dynamo，一个高度可用和可扩展的数据存储系统，被Amazon.com电子商务平台用来存储许多核心服务的状态。Dynamo已经提供了所需的可用性和性能水平，并已成功处理服务器故障，数据中心故障和网络分裂。Dynamo是增量扩展，并允许服务的拥有者根据请求负载按比例增加或减少。Dynamo让服务的所有者通过调整参数N,R和W来达到他们渴求的性能，耐用性和一致性的SLA。

在过去的一年生产系统使用Dynamo表明，分散技术可以结合起来提供一个单一的高可用性系统。其成功应用在最具挑战性的应用环境之一中表明，最终一致性的存储系统可以是一个高度可用的应用程序的构建块。

 

鸣谢
作者在此要感谢PatHelland，他贡献了Dynamo的初步设计。我们还要感谢MarvinTheimer和RobertvanRenesse的评注。最后，我们要感谢我们的指路人(shepherd)，JeffMogul,他的详细的评注和input(不知道怎样说了？词穷)在准备camera ready版本时，大大提高了本文的质量。


