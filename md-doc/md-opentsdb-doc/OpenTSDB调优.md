---
typora-copy-images-to: ./
---

# 1.OpenTSDB TroubleShooting

### 1. OpenTSDB compactions 触发很大的.tmp文件 和 RegionServer crash.

在默认的情况下，TSD会缓存一个metric在一个小时内的数据，然后组合数据一次性刷写到HBase。 如果时间戳使用的ms级别，那么一次性刷写会导致巨大量的流量到HBase， 数量庞大的qualifier和rowkey会达达超过`hfile.index.block.max.size`形成大量.tmp文件，并且一次性写入大量数据可能会冲垮RS。

### 2. 在RS发生Region Split或者Region Migration时，TSD 反应延迟很大

因为region split或者migration的时候，region不可用，那么TSD的asyncHBaseClient会缓存请求RPC到内存，等到Region可用的时候，region会承受缓存的大量的RPC请求，这个会导致response变慢；如果多个region挂掉时，TSD缓存的太多的RPC请求，内存压力大会导致GC问题，反应变慢。 
如果有这种情况发生，增大内存 或者 降低`hbase.nsre.high_watermark`数值。

### 3. TSD stuck in GC & crash due to OOM

几种情况： 
* **见上述2** 

* **写TSD ops太高**：尝试关闭tsd compaction 

- **大查询**：查询太多time series或者有long range scan. 此时 尽量分批查询。



### 4.How does OpenTSDB Compaction save space? 

https://groups.google.com/forum/#!topic/opentsdb/2xhg_ur8gW8

> On looking at [the documentation](http://opentsdb.net/docs/build/html/user_guide/backends/hbase.html#data-table-schema) in the Compaction section, from what I understand, OpenTSDB compacts data by appending together column qualifiers and the values with the other column qualifiers and values for each row.
>
> So won't it take the same amount of space that was used earlier, as the data was being appended into each column?

TSDB compaction won't save space over TSDB Append's, they're effectively equivalent. But the default config for OpenTSDB is to write individual columns per data point and enable compactions. Compactions and appends save a TON of space over the individual columns because:

1) Each column has an 8 byte timestamp in storage associated with the write time (or dp time in tsdb 2.4 with date-tiered hbase compactions).

2) During serialization, HBase returns the row key with every column which is really inefficient for us.

So the recommended configurations depend on priorities:

A) If you need to save space as much as possible but have lots of CPU and IO, try OpenTSDB's appends with compaction disabled. But watch the region server's resources to make sure they aren't running out of IO.

B) If you need as much write throughput as possible, use OpenTSDB's default puts and disable compactions.

C) If you have a low write throughput and enough TSDs and region servers, then try appends or just use puts and compactions.

Yahoo is working on a better append co-processor that gives us the space savings without the region server impact.

 

> The docs also use the phrase, "Since we know each data point qualifier is 2 bytes". However, right before that, they have said that a column qualifier can be "The qualifier is comprised of 2 or 4 bytes that encode an offset from the row's base time and flags to determine if the value is an integer or a decimal value."
>
> Can someone help clear this up too?

 Yeah, I need to update that doc. The qualifier for data points was only two bytes when OpenTSDB only supported second timestamp resolution. When we added millisecond resolution, the qualifiers could be 4 or 2 bytes depending on whether the DP had a second or millisecond resolution. (When we add nanoseconds it may be 6 or 8 bytes).

So the offset and flag encoding is similar but the resolution is different.

Hope that helps :)



# 2.近来关于openTSDB/HBase的一些杂七杂八的调优

https://zhengheng.me/2016/03/07/opentsdb-hbase-tuning/



### 背景

过年前，寂寞哥给我三台机器，说搞个新的openTSDB集群。机器硬件是8核16G内存、3个146G磁盘做数据盘。

我说这太抠了，寂寞哥说之前的TSDB集群运行了两年，4台同样配置的机器，目前hdfs才用了40%，所以前期先用着这三台机器，不够再加。

于是我只好默默地搭好了CDH5、openTSDB(**2.1版本，请注意此版本号**)、bosun，并在20台左右的机器上部署了scollector用来测试，然后将`dfs.replication`改为了2，一切正常。

过完年回来后，开始批量在主要业务机器上部署scollector，大概增加到了160台左右。运行了一个星期后，正在添加各种grafana dashboard的时候，就开始发现各种异常了。

这篇文章主要是总结了处理openTSDB各种状况的过程。需要注意的是，这是个持续的过程，中间修改了非常多的参数，有些问题是做了某项修改后，有很明显的改善，而有些问题的解决其实回头看的时候并不能知道**究竟是哪些修改解决了问题**，也还没有时间重新去修改验证。因此，本文并不能作为一份解决问题的FAQ文档，纯当本人处理问题的记录罢了。

### 打开文件数

首先是发现了无法打开`master:4242`页面了，查看了openTSDB的日志，很显眼的`too many open file`，于是检查了`ulimit`，还是默认的1024，这肯定不够用的。需要修改系统的资源限制：

首先是`/etc/security/limits.conf`

```
# 添加
* soft nofile 102400
* hard nofile 102400
```

然后是`/etc/profile`

```
# 文件末尾添加
ulimit -HSn 102400  
```

重新登录，重启openTSDB，然后检查进程的打开文件数：

```
# cat /proc/$(ps -ef | /bin/grep 'opentsd[b]' | awk '{print $2}')/limits | /bin/grep 'open files'
Max open files            102400              102400              files  
```

修改已生效，打开文件数的问题解决了。

### 内核参数

接着没过1小时，就出现了查询非常久都出不来的情况，于是就注意观察openTSDB的打开文件数，然而只有1500左右，怀疑可能是并发数和backlog的问题，检查了连接数以及当前的内核参数，发现此时连接数并没有超过内核设置。不过本着尽量排除影响的原则，还是改大了这两个内核参数：

```
net.ipv4.tcp_max_syn_backlog = 16384  
net.core.somaxconn = 65535  
```

没有什么新的线索的情况下，只好又重启了TSDB。重启后又恢复正常了。于是用watch命令持续盯着openTSDB的连接数，一边用grafana每30秒刷新页面。发现正常情况时，连接数大概是900以下，而出现问题时(grafana刷不出来图时)，连接数突然就上升到1200以上，然后1分钟内蹦到了2900。赶紧使用netstat看下是什么状态的连接数上涨了，发现是TIME-WAIT。检查`net.ipv4.tcp_tw_reuse`已经是1了，然后这天由于有别的事情要忙，就暂时没再看了。

### regionserver java堆栈大小

第二天早上的时候发现又是grafana刷新不出图，后来上CDH管理页面才发现是其中有一个regionserver(tsdb3)挂了：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160305_1.png)经大数据组大神提醒，多半是GC导致regionserver挂了。查看果然有个小时级别的GC：

![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160305_2.png)

查看了HBase Regionserver java堆栈大小，原来才设置为2.44G。请教了大神，说是因为我部署的这个CDH集群，除了HBase之外还有其他的其实对于openTSDB没有用的Hive、Hue、Oozie、Sqoop2等，所以CDH会根据角色的情况分配java堆栈大小，但是我们完全可以将这些没有用到的服务关掉，手动将HBase Regionserver java堆栈大小设置为物理内存的一半，也就是8G。

改完配置重启了HBase，貌似一切又正常了。而且，由于有更重要的问题要处理，因此就先暂时放一放了。

### HBase表压缩

那件更重要的事情，就是磁盘空间用得太快了。才160台机器一星期的数据，磁盘空间就才90%下降到80%，我一开始觉得不可能，之前的旧集群，用了两年才用了40%，新的怎么一星期就用了10%呢，这么下去难道只能用10个星期？后来看了下CDH的图，每秒datanode写入都有1M左右，146G的硬盘前途堪忧啊！原来是旧的tcollector是自己写的收集脚本，一台机器收集的metric平均才30来个，而scollector不算自己写的外部脚本，本身就提供了上百个metric，再加上自己写的外部收集脚本，这么下来两者的数据根本就不是一个数量级了。

想起之前在初始化TSDB的HBase的时候，没有采用压缩：

```
env COMPRESSION=NONE HBASE_HOME=/usr/lib/hbase /usr/share/opentsdb/tools/create_table.sh  
```

不知道现在还能不能开启表压缩，赶紧去请教大神。大神甩给我一个文档：

```
# hbase shell
# 查看表是否启用压缩
describe "tsdb"  
# 暂停对外服务，将指定的HBase表disable
disable "tsdb"  
# 更改压缩类型
alter 'tsdb', NAME => 't', COMPRESSION => 'gz'  
# 重新启用表
# enable "tsdb"
# 确认是否开启了压缩
discribe "tsdb"  
```

我说，用`gz`压缩是不是对性能影响大啊，不是很多地方都在用`snappy`吗？大神解释说，`gz`是压缩比最高的，只对CPU资源有所损耗；按你这个集群的情况，CPU还有负载，都还是比较闲的，加上磁盘资源又这么紧张，最好还是用`gz`吧，如果有影响，再改为`snappy`呗。

我想也是，于是就开启了表压缩，磁盘空间问题解决了：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160306_5.png)从上图可以很明显地看到开启压缩前后的对比。

### HBase memstore flush and store files compaction

解决完压缩问题后，回头来再看openTSDB查询偶尔没有及时响应的问题。通过查看bosun的Errors页面，经常会出现`net/http: request canceled`的报错，这是因为我设置了bosun每分钟检查一次某些metric作为报警的来源，这些报错是因为读取数据时候超时了，那么应该重点看下openTSDB为何反应慢。看了下TSDB的日志，并没有发现什么异常，于是把目光集中在了HBase上。

Google了一下`tsdb hbase performance`，发现了[这篇文章](http://www.xmsxmx.com/opentsdb-performance-improvement/)，里面的这张图(以下简称为cycle图)总结得很好： 
![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160306_8.png)那么HBase反应慢，应该是这两种情况：

1. 1->8->9
2. 1->2->3->4->5

可以看到`memstore flush`很明显在这张图中处于一个核心地位，那么减少`memstore flush`是否可以改善呢？于是将hbase.hregion.memstore.flush.size从默认的128M改大为1G。

```
CDH中的hbase.hregion.memstore.flush.size作用解释如下：如memstore大小超过此值（字节数），Memstore将刷新到磁盘。通过运行由hbase.server.thread.wakefrequency指定的频率的线程检查此值。  
```

但是这样修改后有个很严重的副作用：GC时间更长了(箭头指向为修改前后的比较)：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160306_9.png)后来就把这个参数恢复为128M这个默认值了。

至于compaction方面的参数，看着解释貌似不好修改，于是就没有改了：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160306_10.png)

### HBase GC 调优

再来看下GC的时间规律，发现都是集中在**一个小时的0分左右**。查看了scollector自带的`hbase.region.gc.CollectionTime`这个指标值，确实在**一个小时的0分左右**就有一个GC的高峰：(红线为ParNew，而蓝线为ConcurrentMarkSweep)![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160306_6.png)大神一看，说每小时开始的时候集群网络IO会飙高，问我TSDB这时候在干什么。![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160306_7.png)

于是我在发生问题的时候用`iftop -NP`检查网络IO，发现基本上都是60020和50010端口之间的流量：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_22-1530891981413.png)看不出来openTSDB在做什么，之前用的TSDB都不用怎么改配置直接用默认值的。

大神沉思下，说我还有办法可以优化下GC时间，优化调整的目的应该是削平GC的波峰，让整个系统的正常服务时间最大化。 大神在`HBase RegionServer 的 Java 配置选项`加上以下参数：

```
-XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:/data/logs/gc.log
```

接着修改了以下HBase参数：

- HBase Server 线程唤醒步骤 (hbase.server.thread.wakefrequency)：该值决定了Hbase Memstore刷新的检测频率，该值默认值为10s，在数据高峰时，每秒写入的数据达到20M左右，调整该值到5s，来帮助尽量使得MemStore的值不超过128M。
- RegionServer 中所有 Memstore 的最大大小 (hbase.regionserver.global.memstore.upperLimit)：该值是小数形式的百分比，默认为0.4，该值与Hfile缓存块大小的总和不能超过0.8，不然会造成HBase启动失败，其目的是为了在Memstore占用的内存达到Java堆栈的该百分比时强制执行刷新数据到磁盘的操作，这里我们将Memstore的百分比设置为0.5，目的是为了尽量避免强制刷新。还有一个最小大小的值(hbase.regionserver.global.memstore.lowerLimit)为0.38，表示要刷新到0.38才结束刷新，未做修改，后续可以调整。
- HFile 块缓存大小 (hfile.block.cache.size)： 该值默认值为0.4，调整为0表示Hbase的磁盘写入不使用内存缓存，测试发现调整为0后性能有一定的退化，尤其是在数据刷新操作的过程中消耗的时间有所上升，这里我们把该值调整为0.3(因为hbase.regionserver.global.memstore.upperLimit已改为了0.5)。
- HStore 阻塞存储文件 (hbase.hstore.blockingStoreFiles)：该值默认为10，如在任意 HStore 中有超过此数量的 HStoreFiles，则会阻止对此 HRegion 的更新，直到完成压缩或直到超过为 'hbase.hstore.blockingWaitTime' 指定的值。将该值改大到100，就是为了减少cycle图中的第9步。

然后根据GC打印日志的分析，还修改了HBase RegionServer的Java配置选项：

> 默认情况下Hbase在新代中采用的GC方式是UseParNewGC，在老代中采用的GC方式为UseConcMarkSweepGC，这两种GC方法都支持多线程的方法，CMS的GC耗费的时间比新代中的GC长，同时如果内存占满还会触发Full GC，我们的优化方向是让GC尽量在新代中进行，通过GC日志发现新代的内存大小只有600M，而总的Java堆栈大小为8G，官方的推荐是新代内存占用为总堆栈的3/8，于是在这里增加参数-Xmn3000m，来扩大新代的大小。

经过大神的一番调优后，可以明显看到GC时间明显下降了：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160306_11.png)

### 文件系统读写优化

考虑到是写入数据太多，查看系统的磁盘所有写所消耗的时间确实都比较高，尝试了以下优化方案：(以sdb为例)

- ext4挂载参数：

```
mount -o remount,noatime,nodelalloc,barrier=0 /dev/sdb1  
```

- 调度电梯算法改为deadline

```
echo "deadline" > /sys/block/sdb/queue/scheduler  
```

- 开启文件系统的预读缓存

```
blockdev --setra 32768 /dev/sdb  
```

经过一番修改后，`linux.disk.msec_write`(Total number of ms spent by all writes) 这个指标值大幅下降，但**不清楚是GC调优还是文件系统读写优化的效果**，但根据经验，ext4性能相比ext3是有所回退的，这么改动应该是有效果的，但`barrier=0`这个挂载参数需谨慎使用。![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_23.png)

### 增加机器/hotspotting

虽然GC时间下降了，但是还是在挺高的数值上，而且bosun的Errors还是偶尔出现，为此我增加了bosun的Errors监控指标，通过bosun的api，专门监控bosun出现`request canceled`的情况：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160306_12-1530891999530.png)可以看到出现此类错误的时候，基本都是在每个小时的0分附近。

反反复复试过了很多HBase调优之后，依然有偶尔出现`request canceled`的问题，且每到整点左右时，用grafana查询也是要1到2分钟才将一个dashboard刷新完，就是说1/30的时间是不可服务的。

既然之前搜到的那篇文章中有提到最好的方式是增加机器了，刚好旧的TSDB集群刚好也压缩过了，用两个regionserver就可以了，于是将剩下的第三台机器加到了新的TSDB集群中来。

新的regioserver加进来后，发现情况还是不是太好，依旧是每到一小时的0分附近就是各种slow response。查看了`master:60010`页面，各regionserver之间的request分配并不平均，通常是新加入的regionserver只是旧的两台的30%不到，这可能会导致大量的写入集中在两台旧的regionserver上，因而造成slow response。所以先尝试手动平衡下region。

比如从`master:60010`上看数据，发现这个region的request比较多，因此决定将它手动迁移到新的tsdb4(tsdb4.domain.com,60020,1457011734982)

```
tsdb,\x00\x00\xF5V\xA6\x9A\xE0\x00\x00\x01\x00\x00\xDA\x00\x00\x13\x00\x00c\x00\x00\x15\x00\x00\xDE,1454242184814.29987604fab49d4fd4a0313c6cf3b1b6.  
```

操作如下：

```
# hbase shell
move "29987604fab49d4fd4a0313c6cf3b1b6" "tsdb4.domain.com,60020,1457011734982"  
balance_switch false  
```

**记得关闭balance，否则过5分钟，被移动的region又自动回来了**

可以看到修改完后，3个regionserver的writeRequestCount比之前平均多了:![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_18.png)

### 增加内存

每到整点就是各种slow response的问题依然存在。没办法之下只好给regionserver增加内存进行测试了。将3台regionserver的内存都增加到了32G，然后将java堆栈大小改为16G。

### 其他优化

主要参考了[这个slide](http://www.slideshare.net/xefyr/hbasecon2014-low-latency)和[这个slide](http://www.slideshare.net/lhofhansl/h-base-tuninghbasecon2015ok)![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_25.png)![img](https://zhengheng.me/content/images/2016/03/Snip20160307_26.png)

另外看到bosun的Errors里还出现了`10000 RPCs waiting on ... to come back online`，怀疑可能是处理程序计数不足，于是调大了这两个参数：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_27.png)![img](https://zhengheng.me/content/images/2016/03/Snip20160307_28.png)

### 最终解？TSDB compaction

各种方法都试过的情况下，还是出现这个“每到整点左右就是各种slow response的问题”。

既然系统、hadoop、HBase方面都调整过了，最后只能找openTSDB和bosun下手了。看到scollector已经提供了相当多的tsd指标值，于是增加了一个TSDB的grafana dashboard把所有的相关指标值都显示出来，看看每个小时0分的时候是否有些蛛丝马迹可循。

最明显的一个指标值就是`tsd.hbase.rpcs`，每到整点的时候才出现这些`delete`、`get`操作，平时都是0，那么100%可以确定这些操作是有关联的：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_37.png)

直接上Google搜`tsdb delete`，结果都是问怎么从TSDB中删除数据的。

回过头来再看其他的指标值，发现`tsd.compaction.count`也是在整点的时候特别高：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_38.png)**需要注意的是TSDB的compaction和HBase的compaction其实不是同一个概念。**

然后用`tsdb compaction`作为关键词一搜，出来了[官网的文档](http://opentsdb.net/docs/build/html/user_guide/backends/hbase.html)，仔细看下这一段：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_39.png)

是不是很符合我们的情况，确实是compaction在前而delete在后。原来是TSDB在每小时整点的时候将上个小时的数据读出来(`get`)，然后compact成一个row，写入(`put`)到HBase，然后删除(`delete`)原始数据，所以我们看整点时的RPC请求，这些操作都会出现个凸起。

接着在[这个链接](https://groups.google.com/forum/?hl=fil#!topic/opentsdb/cQ5pNFst8wo)(貌似都是小集群容易出现这个问题)里看到开发者的讨论：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_45.png)

于是看下openTSDB的[配置文档](http://opentsdb.net/docs/build/html/user_guide/configuration.html)，果然发现了相关配置：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_46.png)都是在openTSDB-2.2才新增的配置，赶紧升级了openTSDB。

那么将tsdb的compaction给关闭了有什么副作用呢？[这个链接](https://groups.google.com/forum/#!topic/opentsdb/A9C4G4Wz9d0)中开发者解释了有两点坏处：

1. 磁盘使用率会增加5到10倍；
2. 读操作会变慢。

这两点显然都是不可取的，而采用修改compaction间隔等效果并不明显，最后采用了`tsd.storage.enable_appends = true`的方式，终于将此问题解决了：

看下修改前后的性能比较：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_49.png)GC时间下降明显。

![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_51.png)memstore变化平缓多了。

![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_53.png)slow put 也不再出现了。

在整点的时候经常出现的`request cancle`不再出现了，整点时使用grafana也不会出现转圈圈到timeout的情况了。

不过值得注意的是，磁盘容量也有所下降(bosun图的时间是UTC时区)：![img](C:\Work\Source\docs\md-doc\md-opentsdb-doc\Snip20160307_54.png)

### 经验总结

整理信息的时候偶然Google发现了这篇[2014年的文章](http://tech.meituan.com/opentsdb_hbase_compaction_problem.html)，其实是一样的问题，这次虽然经历了很多波折，但是也是一个难得的学习机会。虽然作为运维，我们能接触到大量优秀的开源产品，但有多少是能仔细看完一个开源产品的文档的？一个软件，不同的情景下面就有不同的表现，不能一概而论。直到遇到坑才去填，明显是本末倒置的行为，以后要戒掉这种浮躁的心态，同一份配置不要以为之前没有问题就能一直用。





# 3.OpenTSDB/HBase - Improving Stability And Performance

http://www.xmsxmx.com/opentsdb-performance-improvement/



[OpenTSDB](http://opentsdb.net/) is a time series database based on HBase. It is used as a distributed monitoring system where metrics are collected and pinged to openTSDB from servers/machines. The metrics collected can be anything from CPU, RAM usage to Application Cache Hit ratios to Message Queue statistics. It can scale to *millions of writes per second* as per its homepage and in theory it can.

Anyone who has used Hbase substantially in the past knows that it can prove to be a nasty beast to tame. Specially due to how [row keys work](http://hbase.apache.org/book.html#rowkey.design). A poorly designed Hbase row key can lead to all reads and/or writes going to a single node even when we have a cluster. This is called **region hotspotting** and it causes sub-optimal cluster utilisation with poor read/write times. We should, ideally, utilise the cluster to its full potential and in the process make sure that reads and writes from a system meet a SLA.

OpenTSDB avoids problems of hotspotting by [designing its key intelligently](http://opentsdb.net/docs/build/html/user_guide/backends/hbase.html). It is distributed, fault tolerant and highly available as well. It is used at various places in [production](https://github.com/OpenTSDB/opentsdb/wiki/Companies-using-OpenTSDB-in-production).

It indeed works well until one starts to take it for granted.

### Broken promises?

One of the projects i have been involved is started with a notion of broken promises from hadoop/hbase/opentsdb. I was told "we love tsdb, it has been the most used tool in the org BUT it has become slow, we are missing data." The system in question was a 4 node Opentsdb cluster with:

- 32 GB RAM: 12 GB for HBase, 3 GB for Data Node
- 12 Cores, 2.0 GHz Intel CPU
- 8 * 1 TB SATA HDDs
- 10 GB network connections between nodes
- Around a ~ million distinct metrics getting into tsdb regularly.
- The cluster runs on default tsdb and hbase configurations.

Read response time was very erratic. At times it would be below 2 seconds or above 10 seconds or **mostly 25-50 seconds**. Queries to tsdb would take forever to complete. Clients would timeout waiting for results.

It was known that there was some **hotspotting** but no one knew why it was happening or how to fix it. Also, no one wants to play with a production cluster.

"Clearly tsdb is not keeping up with our work load". Right, Time to do some digging and see if it is really a case of broken promises. Before we start....

### HBase in a minute

HBase is a nosql columnar store. It has a concept of table which contains column families. Usually there are 1 or 2 column families per table. Data in a table is stored in a sorted order based on a row-key.

Each table is divided into several regions (based on amount of data it holds). Each region contains a store per column family which is turn contains: a memstore and several store files.

### Why do we have a hotspot? We use tsdb!

A general understanding with tsdb is that we will not have hotspots as keys are created intelligently. So what is happening with this cluster? We need monitoring of the cluster itself, meta-monitoring in a sense. We need to monitor the cluster that helps monitor everything else.

Guess what, in this case, tsdb is monitoring itself. So we get to all sorts of metric and graphs for our cluster.

##### Hotspot

![A hotspot in hbase](C:\Work\Source\docs\md-doc\md-opentsdb-doc\initialHotspot_w58nrn.png)

As we can see in the image above, out of 4 machines, one is getting way more write requests (1500-2000 a second) than others. Others are at least half or less. One box (green) is not even visible anywhere!

If it was a hbase cluster created by one of the in-house teams, chances were that the team got the row-key design wrong and ended up with this cluster state. This is a openTSDB hbase cluster and keys are well designed. With tsdb this can possibly happen only if:

- we started with many different keys and somehow stopped writing to all of them.
- we write to a few keys a lot more than others.

Even when we write to few keys a lot more than others; hbase by design, at some point, would split the region and move some part of the hot region to a random machine. With that we would get, in theory, writes going to at least two different machines. That is clearly not the case here.

There are many unknowns here before we can decide on as to why this happened. May be we are really writing a lot to a few keys.

Let's have a look at the reading pattern: 
![Hbase reading pattern](C:\Work\Source\docs\md-doc\md-opentsdb-doc\initialHotspotReads_tpecot.png)

Two important things to note in the image above:

```
1. Reads follow write pattern. A machine with high writes gets high reads as well.
2. Reads are going to only two machines and are not well distributed. 
```

It seems that most of the clients are reading data which was written just now. **Readers are overwhelmingly interested in recent data**.

And this is how the response times to queries looked: 
![Response times](C:\Work\Source\docs\md-doc\md-opentsdb-doc\initialHotspotResponseTimes_lk5z9k.png)The one word that describes this graph is **erratic**. Response times are way beyond civilized limits of 1-2 seconds.

Based on this much info, we gather that there is a problem with response times and it looks like it is related to hotspotting.

We had another tool at our disposal to look at the underlying Hbase cluster: [**Hannibal**](https://github.com/sentric/hannibal). Hannibal is an excellent tool to visualise the state of hbase cluster and look into region details. We can look at the region distribution across nodes.

![Hbase Region Distribution](C:\Work\Source\docs\md-doc\md-opentsdb-doc\regions_xszahp.png)

This reinforces our understanding that one of the server is a hotspot. One of the machines is holding almost twice as much data as others. This brings us to a key question: **How many regions each of the nodes holding**? The answer was **700+** and they **are evenly distributed** across all nodes. That clearly means that we are using something called hbase **balancer**.

Hbase balancer makes sure that the regions are evenly distributed across all nodes. It does a great job But.. in this case does not look like it. Even when the region counts are equal, the amount of data is not. And that is something the current balancer cannot do.

### What is the problem exactly?

700+ regions on a node with 12GB RAM is [far too many](http://archive.cloudera.com/cdh5/cdh/5/hbase/book/regions.arch.html), in fact the recommended region count is 20-100. No doubt this cluster has problems. Let's see how and why.

First some default settings in HBase:

```
    hbase.hregion.memstore.mslab.enabled=true
    hbase.hregion.memstore.mslab.chunksize=2MB
    hbase.hregion.memstore.flush.size=128MB
    hbase.hregion.max.filesize=10GB
    hbase.regionserver.global.memstore.upperLimit=0.4
```

[MSLAB](http://blog.cloudera.com/blog/2011/03/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-3/) is a memory area maintained per region internally by Hbase for avoiding full GCs. As we can see it is 2MB by default for each region. For 700 regions it is ~ 1.4 GB of heap before any data is written!

##### How many regions per RS?

Each region in HBase has a memstore per column family, which can grow up to **hbase.hregion.memstore.flush.size**. We also have a upperLimit on the total memstore size in terms of available heap. Based on this, a good starting point for number of regions per region server is:

```
(RS memory)*(total memstore fraction)/((memstore size)*(# column families))
```

For our clusters, this works out to be:

```
12000*0.4/128*1 
```

i.e. ~ 38. This is assuming that all the regions are filled up at the same rate, Which is not the case (we will see below why). We can go with as many as 3-4 times this number which would be ~ 160. So, for our cluster we can have 160 odd regions safely on each region server. We have 700 on each. How do we find out if that really is a problem?

### Too many regions?

With 12GB of heap and default **hbase.regionserver.global.memstore.upperLimit** of 0.4, we have 4.8GB of heap before **writes are blocked and flushes are forced**. We have 700 regions, 4.8GB for memstore: the cluster would make **tiny flushes of ~ 7MB** (instead of configured 128MB) or even less; if all regions were active. Even if all the regions are not active but upperLimit for combined memstore is reached, memory flushes would happen.

On this cluster, there were tiny flushes of 1-10 MB and at times large flushes of 50-128MB as well. All the 700 regions on this cluster were not **active**. Defining active here is little difficult. A region can be considered active if it attracts reads or writes or both. But since we are focusing on memory flushes, let's say a region is active if attracts writes.

Here is one region on one of the busy nodes: 
![Tiny memory flushes](C:\Work\Source\docs\md-doc\md-opentsdb-doc\tinyMemFlushes_bjozui.png)

Notice the memory flush pattern, it is almost at 7MB. Here is another region with flushes. This one has larger flush size.

![Another region with flushes](C:\Work\Source\docs\md-doc\md-opentsdb-doc\anotherRegionFlush_g70vyk-1530892539331.png)

Tiny memory flushes would create many smaller files and those small files would keep on triggering minor compactions (ruled by several compaction and file related configuration parameters) all the time. Many smaller files can also cause writes to accumulate in memory as per **hbase.hstore.blockingStoreFiles** config which defaults to **10**. Therefore, if we have 10 or more store files (small or large) updates are blocked for that region or until **hbase.hstore.blockingWaitTime**. After blockingWaitTime upates are allowed to a max of **hbase.hregion.memstore.block.multiplier \* hbase.hregion.memstore.flush.size** which is 1024 MB with default settings.

As we can see, this is a vicious cycle. 
![Vicious Cycle](C:\Work\Source\docs\md-doc\md-opentsdb-doc\ViciousCycle_b0igcw.png)

Memory pressure results in smaller files which results in more writes into memory which again results in more files! Many files on many regions would lead to a substantial compaction queue size. Compaction queue is the compactions tasks on regions that Hbase needs to perform. As to when a compaction is carried on a region is dependent on Hbase internals. One of the reasons that could force quick compaction is many writes and global memory pressure. One can check below line in region server logs:

```
16:24:55,686 DEBUG org.apache.hadoop.hbase.regionserver.MemStoreFlusher: Under global heap pressure: Region tsdb, ...... 
```

This is how the compaction queue size looks on the hot region server: 
![Compaction Queue Size](C:\Work\Source\docs\md-doc\md-opentsdb-doc\compactionQueueSizeHot_vne0io.png)

The box with most writes is having a huge compaction queue size. This clearly indicates that the box is under memory pressure.

(Curious minds would question the sudden drop thrice in the queue size, upon investigation i found out that the region server became too busy compacting and had a [Juliet pause](http://blog.cloudera.com/blog/2011/02/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-1/). It could not contact zookeeper in configured timeout and was declared dead! The region server shut itself down BUT there was a magic restart script running on the box that restarted it. We know how to fix a problem, don't we :-)

In the logs we can see below line:

```
09:41:00,615 WARN org.apache.hadoop.hbase.util.Sleeper: We slept 23274ms instead of 3000ms, this is likely due to a long garbage collecting pause and it's usually bad, see http://hbase.apache.org/book.html#trouble.rs.runtime.zkexpired
```

)

And this is **why we need to avoid frequent memory flushes**.

Other than writes, some heap would be used to serve reads as well. Many reads would touch several regions or regions not currently being written. Especially for openTSDB, a query for past 6 months may touch many regions.

Coming back to the cluster in question, we now know that one of the nodes seems to be holding majority of the active (or hot) regions. And that node comes under memory pressure due to high writes and reads resulting in poor response times for queries.

We now understand as to why we have slow response times, how do we solve this? Also, how do we distribute the load more evenly?

### Serendipity in failure

While trying to understand the problem in full, I looked at data going back as far a month. And found this:

![Region Crossover](C:\Work\Source\docs\md-doc\md-opentsdb-doc\crossoverWrites_yregcl.png)

In the past, writes have been switching from one box to another. In the picture above, a drop in purple was actually a GC pause (had to look at region server logs and zookeeper logs) and region server was declared dead. By the time this region server was up and running again, all the hot regions were moved to another node. How did that happen? The Hbase balancer. The balancer detects a node failure and moves regions to other nodes in the cluster. Here, i guess, due to [Hbase-57](https://issues.apache.org/jira/browse/HBASE-57), regions are assigned to a box with a replica.

This crossover happened several times in the past. Here is another. 
![Writes Crossover](C:\Work\Source\docs\md-doc\md-opentsdb-doc\crossoverWrites2_hdcqx8.png)

The surprise was how the balancer was moving regions after RS restarts. And this explains how we could have ended up in the current situation. Random RS going off causes it regions to fly to other RSs and it may have overloaded already overloaded node.

### A balancing Act

Now that we understood as to why we are where we are and why response times are slow, we needed to fix it. Couple of options:

```
1. add more nodes in the cluster and hope heavy writes go to new node
2. Manually move regions to lightly loaded nodes
```

Before adding new hardware, i wanted to check if the current one was sufficient. So went with option 2.

We needed to move some of the writes and reads to other 3 nodes of the cluster. One node was doing too much while others were sitting idle. One solution was to move some hot regions to other nodes manually. How do we find hot regions?

On the **region server status** web page i.e. http://:60030/rs-status, we get a list of all the regions this server holds with other stats per region like numberOfStoreFiles, numberOfReads, numberOfWrites etc.

These stats are **reset on server restart**. So, they do not represent data from the time a region came into being. Still, these stats give us valuable information. I grabbed the table that lists all the regions and paste it into an excel sheet. Sort on two key metric: read and write count. Resultant ordering may not be the very accurate view of hot/active regions but it gives us lots to play with. We cannot move a region manually at hbase shell:

```
move '9ba6c6158e2b3026b22d623db74cb488','region-server-domain-address,60020,region-server-start-key'

Where region-server-domain-address is the node where we want to move given region to. One can get the start-key from master status page.
```

I tried moving one of the regions manually to a less loaded node and it worked. Only to find out some time later (5 mins) that it had moved back! WTF?! Turned out it was the **Hbase Balancer** at work. In its obsession with region counts, it deduced that the just moved region has skewed the region balance and hence moved it back.

What's the solution then? We turn the balancer off. And that seems logical as well given our reads and writes are skewed. Our load is not equally distributed across regions therefore keeping same regions counts on all nodes is not the best strategy.

After turning the load balancer off at the hbase shell

```
balance_switch true
```

I moved top 25-30 regions from the busiest node to other nodes randomly. This move was a mix of high writes and high reads.

![Region Moves](C:\Work\Source\docs\md-doc\md-opentsdb-doc\regionMigrationHighLevel_qabszv-1530892574633.png)

We can see that the writes and reads move to other nodes. We can now see all colours in the graph. There was a change in response times as well.

![Response time change](http://res.cloudinary.com/xmsxmx-com/image/upload/v1428491951/regionMoveAndResponseTimes_adls6a.png)

There was a gradual reduction in response time while regions were being moved. The next day though, the response times were much more acceptable. Heap usage, Total GC times on the loaded node improved too.

But as things looked to stabilise, one of the machines had a stop-the-world GC pause and all its regions started moving to other 3 nodes. In the absence of balancer, they could have gone anywhere and may have ended up one node filling up all its disks. So i had to turn the balancer on! And we were back at where we started. 
This led to a adding at least one new node to the cluster. 
![New Machine Added](C:\Work\Source\docs\md-doc\md-opentsdb-doc\extraMachineAddedAutoBalancedRegions_cubhrx-1530892886547.png)The balancer moved data from all other nodes to this new node keeping the overall region count same on all nodes. Writes were still skewed. 
![Skewed writes](C:\Work\Source\docs\md-doc\md-opentsdb-doc\extraMachineAddedWritesRecent_zlj4fi-1530893170213.png) We switched off the balancer now and moved regions manually again. And here is how things looked. We move regions with heavy writes to two least utilised nodes. We can see a drop in writes for one of the loaded nodes below. 
![Region Move](http://res.cloudinary.com/xmsxmx-com/image/upload/v1428493771/regionMoveWithNewBox_s1uprj.png)Several further moves resulted in evenly balanced writes on all nodes. 
![More Moves](C:\Work\Source\docs\md-doc\md-opentsdb-doc\balanced_xkrc32.png) A full day looked like this: 
![Full Day](C:\Work\Source\docs\md-doc\md-opentsdb-doc\fullDay_qea6vo.png)And this is how response times changed. This is a view of 10 day preiod when times were worst and moved to better. 
![10 day response times](C:\Work\Source\docs\md-doc\md-opentsdb-doc\10day_oh3gxj.png) Looking more closely, 
![Improved times](C:\Work\Source\docs\md-doc\md-opentsdb-doc\rtimesOneDay_yh0f3d.png)As we can see, manually moving active/hot regions to underutilised nodes have worked for us. We have a balanced cluster and improved response times.

### We love tsdb, again!

Everyone is happy and people love tsdb again. There were several other things that happened in terms of disks, Hadoop etc. I will write that in a separate post. 
Bottom line, Hadoop eco-system tools and frameworks are designed to work under extreme conditions but there is a limit up to which we can stretch them. A tool cannot keep on working for a eternity on initial capacity. Extra capacity and tweaks are needed as the system grows over time.

Also, it is great to invest in higher level abstractions, understanding a level underneath is crucial for the abstraction to continue to work for us.