备注：本文结合了网上的一些资料和源码分析的过程。主要参考了http://www.nosqlnotes.com/technotes/opentsdb-schema/系列文章。图解释的很清晰，但是缺少源码和数据计算逻辑。

#  1. TSDB元数据

## 1.1 什么是时序数据？

Wiki中关于”**时间序列（Time Series）**“的定义： 

> **时间序列**（Time Series）是一组按照时间发生先后顺序进行排列的**数据点序列**，通常一组时间序列的时间间隔为一恒定值（如1秒，5分钟，1小时等）。 

**时间序列数据**可被简称为**时序数据**。

**实时监控**系统所收集的监控指标数据，通常就是**时序数据** 。时序数据具有如下特点： 

- 每一个**时间序列**通常为某一**固定类型**的**数值** 
- 数据按一定的时间间隔持续产生，每条数据拥有自己的**时间戳**信息 
- 通常只会不断的**写入**新的数据，几乎不会有**更新**、**删除**的场景 
- 在读取上，也往往倾向于**读取最近写入**的数据。 



正是因为这些特点，通常使用专门的时序数据库来存储，因为这类数据库更能理解**时序数据((TSDB))**的特点，而且在读写上做一些针对性的优化。时序数据广泛应用于物联网（IoT）设备监控系统 ，企业能源管理系统（EMS），生产安全监控系统，电力检测系统等行业场景。 相信在在即将大范围普及的**物联网(IoT)**应用场景中，**时序数据库(TSDB)**会得到更加广泛的应用。

以下为时序数据库常见名词解释：

**TSDB ：**Time Series Database，时序数据库，用于保存时间序列（按时间顺序变化）的海量数据。

**度量（metric）：**数据指标的类别，如发动机的温度、发动机转速、模拟量等。

**时间戳（timestamp）：**数据产生的时间点。

**数值（value）：**度量对应的数值，如56°C、1000r/s等（实际中不带单位）。

**标签（tag）：**一个标签是一个key-value对，用于提供额外的信息，如"设备号=95D8-7913"、“型号=ABC123”、“出厂编号=1234567890”等。

**数据点（data point）：**“1个metric+1个timestamp+1个value + n个tag（n>=1）”唯一定义了一个数据点。当写入的metric、timestamp、n个tag都相同时，后写入的value会覆盖先写入的value。

**时间序列 ：**“1个metric +n个tag（n>=1）”定义了一个时间序列。单域和多域的数据点和时间序列由下图所示：



如下是源自OpenTSDB官方资料中的**时序数据样例**： 

> sys.cpu.user host=webserver01 1356998400 50 
>
> sys.cpu.user host=webserver01,cpu=0 1356998400 1 
>
> sys.cpu.user host=webserver01,cpu=1 1356998400 0 
>
> sys.cpu.user host=webserver01,cpu=2 1356998400 2 
>
> sys.cpu.user host=webserver01,cpu=3 1356998400 0 
>
> ………… 
>
> sys.cpu.user host=webserver01,cpu=63 1356998400 1 

对于上面的任意一行数据，在OpenTSDB中称之为一个**时间序列**中的一个**Data Point**。以最后一行为例我们说明一下OpenTSDB中关于Data Point的每一部分组成定义如下： 

|   构成信息   |   TSD名称   |
| :----------: | :---------: |
| sys.cpu.user | **metrics** |
|     host     |   tagKey    |
| webserver01  |  tagValue   |
|     cpu      |   tagKey    |
|      63      |  tagValue   |
|  1356998400  |  timestamp  |
|      1       |    value    |

可以看出来，每一个Data Point，都关联一个metrics名称，但可能关联多组<tagKey,tagValue>信息。而关于时间序列，事实上就是具有相同的metrics名称以及相同的<tagKey,tagValue>组信息的Data Points的集合。在存储这些Data Points的时候，大家也很容易可以想到，可以将这些metrics名称以及<tagKey,tagValue>信息进行特殊编码来优化存储，否则会带来极大的数据冗余。OpenTSDB中为每一个metrics名称，tagKey以及tagValue都定义了一个唯一的数字类型的标识码(UID)。 



## 1.2 TSDB元数据模型

### UID设计



UID的全称为Unique Identifier。这些UID信息被保存在OpenTSDB的元数据表中，默认表名为”tsdb-uid”。 

OpenTSDB分配UID时遵循如下规则： 

- metrics、tagKey和tagValue的UID分别独立分配 
- 每个metrics名称（tagKey/tagValue）的UID值都是唯一。不存在不同的metrics（tagKey/tagValue）使用相同的UID，也不存在同一个metrics（tagKey/tagValue）使用多个不同的UID 
- UID值的范围是0x000000到0xFFFFFF，即metrics（或tagKey、tagValue）最多只能存在16777216个不同的值。  



### 元数据HBase表设计

为了从UID索引到metrics（或tagKey、tagValue），同时也要从metrics（或tagKey、tagValue）索引到UID，OpenTSDB同时保存这两种映射关系数据。 在元数据表中，把这两种数据分别保存到两个名为”id”与”name”的Column Family中，Column Family描述信息如下所示： 

> {NAME => ‘id’, BLOOMFILTER => ‘ROW’, COMPRESSION => ‘SNAPPY’}
>
>  {NAME =>’name’,BLOOMFILTER => ‘ROW’, COMPRESSION => ‘SNAPPY’, MIN_VERSIONS => ‘0’, BLOCKCACHE => ‘true’, BLOCKSIZE => ‘65536’, REPLICATION_SCOPE => ‘0’} 

下面为Hbase shell一段代码的执行过程：

> hbase(main):009:0> scan 'tsdb-uid'
> ROW                                        COLUMN+CELL     
>
>  \x00                                      column=id:metrics, timestamp=1530288910326, value=\x00\x00\x00\x00\x00\x00\x00\x12                                          
>  \x00                                      column=id:tagk, timestamp=1529652270537, value=\x00\x00\x00\x00\x00\x00\x00\x0A                                             
>  \x00                                      column=id:tagv, timestamp=1530288910343, value=\x00\x00\x00\x00\x00\x00\x00\x12    
>
> ​                                                                                                                                              \x00\x00\x01                              column=name:metrics, timestamp=1528879705241, value=sys.cpu.user                                                             \x00\x00\x01                              column=name:tagk, timestamp=1528879705280, value=host                                                                        \x00\x00\x01                              column=name:tagv, timestamp=1528879705313, value=web01                                                                       \x00\x00\x02                              column=name:metrics, timestamp=1528880100259, value=sys.cpu.sys                                                              \x00\x00\x02                              column=name:tagk, timestamp=1528879705347, value=user                                                                        \x00\x00\x02                              column=name:tagv, timestamp=1528879705377, value=10001                                                                       
>
> 10001                                     column=id:tagv, timestamp=1528879705387, value=\x00\x00\x02                                                                  host                                      column=id:tagk, timestamp=1528879705287, value=\x00\x00\x01                                                                  sys.cpu.sys                               column=id:metrics, timestamp=1528880100264, value=\x00\x00\x02                                                               sys.cpu.user                              column=id:metrics, timestamp=1528879705251, value=\x00\x00\x01                                                               user                                      column=id:tagk, timestamp=1528879705356, value=\x00\x00\x02                                                                  web01                                     column=id:tagv, timestamp=1528879705321, value=\x00\x00\x01                                                                  

由于HBase的存储数据类型是Bytes，所以UID在存储时会被转换为3个字节长度的Bytes数组进行存储。 从上面的数据可以看出hbase是如何保存uid数据的。

Opentsdb预留了rowkey 0x00作为特殊用途，用来生成metrics/tagk/tagv的UID；从上面数据可以看出，当前系统中共有0x12个metrics，0x0A个tagk，0x12个tagv。



Opentsdb在生成metrics的UID时，有一个选项：

- tsd.core.uid.random_metrics

表示是否使用随机数来生成metrics的id，但是针对tagk和tagv都不是使用随机数的。

这一点从下面的代码中可以看出：

> net.opentsdb.core.TSDB#TSDB(org.hbase.async.HBaseClient, net.opentsdb.utils.Config)

```java
if (config.getBoolean("tsd.core.uid.random_metrics")) {
  metrics = new UniqueId(this, uidtable, METRICS_QUAL, METRICS_WIDTH, true);
} else {
  metrics = new UniqueId(this, uidtable, METRICS_QUAL, METRICS_WIDTH, false);
}
tag_names = new UniqueId(this, uidtable, TAG_NAME_QUAL, TAG_NAME_WIDTH, false);
tag_values = new UniqueId(this, uidtable, TAG_VALUE_QUAL, TAG_VALUE_WIDTH, false);
```

另外，UniqueId类也不是代表了单独的一个ID ，而是对所有的ID的管理，实现了UniqueId的CRUD，所以UniqueId应该叫UniqueIdS才对，而net.opentsdb.tools.UidManager则是提供了几个工具命令来实现UID的某些操作，比如grep，assign，rename，delete(metrics/tag)，fsck，metasync等，所以UidManager应该叫UidTool才对。

### TSUID

对每一个Data Point，metrics、timestamp、tagKey和tagValue都是必要的构成元素。除timestamp外，metrics、tagKey和tagValue的UID就可组成一个TSUID，每一个TSUID关联一个**时间序列**，如下所示： 

> *<metrics_UID><tagKey1_UID><tagValue1_UID>[…<tagKeyN_UID><tagValueN_UID>]* 

TSDB在生成TSUID时，针对TagKey按照tagk的UID进行了二进制排序。理论上，TSUID的全集就是所有的tagk的所有的tagv的uid的排列。可以了解为关系数据库里面所有列的所有值的组合。



### TSDB的标签个数限制

本身不加限制，但是实施时最好加上。超过8个标签会有性能上的问题。

```
if (config.hasProperty("tsd.storage.max_tags")) {
  Const.setMaxNumTags(config.getShort("tsd.storage.max_tags"));
}
```

```
/**
 * -------------- WARNING ----------------
 * Package private method to override the maximum number of tags.
 * 8 is an aggressive limit on purpose to avoid performance issues.
 * @param tags The number of tags to allow
 * @throws IllegalArgumentException if the number of tags is less
 * than 1 (OpenTSDB requires at least one tag per metric).
 */
static void setMaxNumTags(final short tags) {
  if (tags < 1) {
    throw new IllegalArgumentException("tsd.storage.max_tags must be greater than 0");
  }
  MAX_NUM_TAGS = tags;
}
```

Metrics/tagk/tagv的字节都是三个：

```
private static final String METRICS_QUAL = "metrics";
private static short METRICS_WIDTH = 3;
private static final String TAG_NAME_QUAL = "tagk";
private static short TAG_NAME_WIDTH = 3;
private static final String TAG_VALUE_QUAL = "tagv";
private static short TAG_VALUE_WIDTH = 3;
```



net.opentsdb.tsd.RpcManager#initializeBuiltinRpcs

```
if (enableApi) {
  http.put("api/aggregators", aggregators);
  http.put("api/annotation", annotation_rpc);
  http.put("api/annotations", annotation_rpc);
  http.put("api/config", new ShowConfig());
  http.put("api/dropcaches", dropcaches);
  http.put("api/query", new QueryRpc());
  http.put("api/search", new SearchRpc());
  http.put("api/serializers", new Serializers());
  http.put("api/stats", stats);
  http.put("api/suggest", suggest_rpc);
  http.put("api/tree", new TreeRpc());
  http.put("api/uid", new UniqueIdRpc());
  http.put("api/version", version);
}
```



### UID的生成

> net.opentsdb.uid.UniqueId#UniqueId(net.opentsdb.core.TSDB, byte[], java.lang.String, int, boolean)

```
/** Generates either a random or a serial ID. If random, we need to
 * make sure that there isn't a UID collision.
 */
private Deferred<Long> allocateUid() {
  LOG.info("Creating " + (randomize_id ? "a random " : "an ") + 
      "ID for kind='" + kind() + "' name='" + name + '\'');

  state = CREATE_REVERSE_MAPPING;
  if (randomize_id) {
    return Deferred.fromResult(RandomUniqueId.getRandomUID());
  } else {
    return client.atomicIncrement(new AtomicIncrementRequest(table, 
                                  MAXID_ROW, ID_FAMILY, kind));
  }
}

public static long getRandomUID(final int width) {
    if (width > MAX_WIDTH) {
      throw new IllegalArgumentException("Expecting to return an unsigned long "
          + "random integer, it can not be larger than " + MAX_WIDTH + 
          " bytes wide");
    }
    final byte[] bytes = new byte[width];
    random_generator.nextBytes(bytes);

    long value = 0;
    for (int i = 0; i<bytes.length; i++){
      value <<= 8;
      value |= bytes[i] & 0xFF;
    }

    // make sure we never return 0 as a UID
    return value != 0 ? value : value + 1;
  }
```

ID的生成逻辑很简单：如果是随机数，那么就生成3个自己的随机数；如果不是，那么就对uid表对一个的列发起一个AtomicIncrementRequest操作，实现ID的递增目的；

### Qualifier的设计

Qualifier用于保存一个或多个DataPoint中的时间戳、数据类型、数据长度等信息。

由于时间戳中的小时级别的信息已经保存在RowKey中了，所以Qualifier只需要保存一个小时中具体某秒或某毫秒的信息即可，这样可以减少数据占用的空间。

一个小时中的某一秒（少于3600）最多需要2个字节即可表示，而某一毫秒（少于3600000）最多需要4个字节才可以表示。为了节省空间，OpenTSDB没有使用统一的长度，而是对特定的类型采用特性的编码方法。Qualifer的数据模型主要分为如下三种情况：秒、毫秒、秒和毫秒混合。



### 秒类型

当OpenTSDB接收到一个新的DataPoint的时候，如果请求中的时间戳是秒，那么就会插入一个如下模型的数据。

判断请求中的时间戳为秒或毫秒的方法是基于时间戳数值的大小，如果时间戳的值的超过无符号整数的最大值（即4个字节的长度），那么该时间戳是毫秒，否则为秒。

![Qualifier-Second](/Users/zhangwusheng/Documents/GitHub/docs/md-doc/md-opentsdb-doc/Qualifier-Second.png)

- Value长度：Value的实际长度是Qualifier的最后3个bit的值加1，即(qualifier & 0x07) + 1。表示该时间戳对应的值的字节数。所以，值的字节数的范围是1到8个字节。
- Value类型：Value的类型由Qualifier的倒数第4个bit表示，即(qualifier & 0x08)。如果值为1，表示Value的类型为float；如果值为0，表示Value的类型为long。
- 时间戳：时间戳的值由Qualifier的第1到第12个bit表示，即(qualifier & 0xFFF0) >>>4。由于秒级的时间戳最大值不会大于3600，所以qualifer的第1个bit肯定不会是1。

### 毫秒类型

当OpenTSDB接收到一个新的DataPoint的时候，如果请求中的时间戳是毫秒，那么就会插入一个如下模型的数据。

![Qualifier-milisecond](/Users/zhangwusheng/Documents/GitHub/docs/md-doc/md-opentsdb-doc/Qualifier-milisecond.png)

- Value长度：与秒类型相同。
- Value类型：与秒类型相同。
- 时间戳： 时间戳的值由Qualifier的第5到第26个bit表示，即(qualifier & 0x0FFFFFC0) >>>6。
- 标志位：标志位由Qualifier的前4个bit表示。当该Qualifier表示毫秒级数据时，必须全为1，即(qualifier[0] & 0xF0) == 0xF0。
- 第27到28个bit未使用。

### 混合类型

当同一小时的数据发生合并后，就会形成混合类型的Qualifier。

合并的方法很简单，就是按照时间戳顺序进行排序后，从小到大依次拼接秒类型和毫秒类型的Qualifier即可。

![Qualifier-mix](/Users/zhangwusheng/Documents/GitHub/docs/md-doc/md-opentsdb-doc/Qualifier-mix.png)

- 秒类型和毫秒类型的数量没有限制，并且可以任意组合。
- 不存在相同时间戳的数据，包括秒和毫秒的表示方式。
- 遍历混合类型中的所有DataPoint的方法是：
  - 从左到右，先判断前4个bit是否为0xF
  - 如果是，则当前DataPoint是毫秒型的，读取4个字节形成一个毫秒型的DataPoint
  - 如果否，则当前DataPoint是秒型的，读取2个字节形成一个秒型的DataPoint
  - 以此迭代即可遍历所有的DataPoint





> net.opentsdb.core.Const

```java
public static final short MS_FLAG_BITS = 6;
public static final int MS_FLAG = 0xF0000000;
public static final short MAX_TIMESPAN = 3600;
public static final short FLAG_BITS = 4;
```

> net.opentsdb.core.Internal#buildQualifier

long类型的flags：

	序列化为bytes数组的长度减一
Double类型的flags：
  	final short flags = Const.FLAG_FLOAT | 0x7;  // A float stored on 8 bytes.

```
public static byte[] buildQualifier(final long timestamp, final short flags) {
  final long base_time;
  if ((timestamp & Const.SECOND_MASK) != 0) {
    // drop the ms timestamp to seconds to calculate the base timestamp
    base_time = ((timestamp / 1000) - ((timestamp / 1000) 
        % Const.MAX_TIMESPAN));
    final int qual = (int) (((timestamp - (base_time * 1000) 
        << (Const.MS_FLAG_BITS)) | flags) | Const.MS_FLAG);
    return Bytes.fromInt(qual);
  } else {
    base_time = (timestamp - (timestamp % Const.MAX_TIMESPAN));
    final short qual = (short) ((timestamp - base_time) << Const.FLAG_BITS
        | flags);
    return Bytes.fromShort(qual);
  }
}
```



- 比较

实际操作是，判断Qualifier代表的是秒（2字节）还是毫秒（4字节），还原成整形，规整为毫秒数，比较毫秒数的大小，这个毫秒数叫offset，这个offset就是基于rowkey里面的整小时的offset。秒最多是3600秒，所以两个字节就够了。

```
/**
 * Compares two data point byte arrays with offsets.
 * Can be used on:
 * <ul><li>Single data point columns</li>
 * <li>Compacted columns</li></ul>
 * <b>Warning:</b> Does not work on Annotation or other columns
 * @param a The first byte array to compare
 * @param offset_a An offset for a
 * @param b The second byte array
 * @param offset_b An offset for b
 * @return 0 if they have the same timestamp, -1 if a is less than b, 1 
 * otherwise.
 * @since 2.0
 */
public static int compareQualifiers(final byte[] a, final int offset_a, 
    final byte[] b, final int offset_b) {
  final long left = Internal.getOffsetFromQualifier(a, offset_a);
  final long right = Internal.getOffsetFromQualifier(b, offset_b);
  if (left == right) {
    return 0;
  }
  return (left < right) ? -1 : 1;
}
```

```
/**
 * Returns the offset in milliseconds from the row base timestamp from a data
 * point qualifier at the given offset (for compacted columns)
 * @param qualifier The qualifier to parse
 * @param offset An offset within the byte array
 * @return The offset in milliseconds from the base time
 * @throws IllegalDataException if the qualifier is null or the offset falls 
 * outside of the qualifier array
 * @since 2.0
 */
public static int getOffsetFromQualifier(final byte[] qualifier, 
    final int offset) {
  validateQualifier(qualifier, offset);
  if ((qualifier[offset] & Const.MS_BYTE_FLAG) == Const.MS_BYTE_FLAG) {
    return (int)(Bytes.getUnsignedInt(qualifier, offset) & 0x0FFFFFC0) 
      >>> Const.MS_FLAG_BITS;
  } else {
    final int seconds = (Bytes.getUnsignedShort(qualifier, offset) & 0xFFFF) 
      >>> Const.FLAG_BITS;
    return seconds * 1000;
  }
}
```

- 获取数据长度（数据的长度保存在Qualifier里面）

```
/**
 * Returns the length of the value, in bytes, parsed from the qualifier
 * @param qualifier The qualifier to parse
 * @param offset An offset within the byte array
 * @return The length of the value in bytes, from 1 to 8.
 * @throws IllegalArgumentException if the qualifier is null or the offset falls
 * outside of the qualifier array
 * @since 2.0
 */
public static byte getValueLengthFromQualifier(final byte[] qualifier, 
    final int offset) {
  validateQualifier(qualifier, offset);    
  short length;
  if ((qualifier[offset] & Const.MS_BYTE_FLAG) == Const.MS_BYTE_FLAG) {
    length = (short) (qualifier[offset + 3] & Internal.LENGTH_MASK); 
  } else {
    length = (short) (qualifier[offset + 1] & Internal.LENGTH_MASK);
  }
  return (byte) (length + 1);
}
```



# 2. 写流程

Opentsdb数据的写入方式有两种：一种是通过Http或者Telnet实现单条或者多条数据的导入，一种是使用import实现数据的批量导入，首先分析HTTP请求的写入流程



## 2.1 请求的解析

> net.opentsdb.tsd.PutDataPointRpc
>
> execute(net.opentsdb.core.TSDB, net.opentsdb.tsd.HttpQuery)

```
public void execute(final TSDB tsdb, final HttpQuery query) 
  throws IOException {

 // only accept POST
  .....
  final List<IncomingDataPoint> dps;
  try {
    dps = query.serializer()
        .parsePutV1(IncomingDataPoint.class, HttpJsonSerializer.TR_INCOMING);
  } catch (BadRequestException e) {
    illegal_arguments.incrementAndGet();
    throw e;
  }
  processDataPoint(tsdb, query, dps);
}
```

```
public <T extends IncomingDataPoint> void processDataPoint(final TSDB tsdb, 
    final HttpQuery query, final List<T> dps) {
  .....
  
  final List<Deferred<Boolean>> deferreds = synchronous ? 
      new ArrayList<Deferred<Boolean>>(dps.size()) : null;
  ...一系列操作,不同的分支，我们这里仅仅研究最普通的数据上传...
  
   
       deferred = tsdb.addPoint(dp.getMetric(), dp.getTimestamp(),
                Tags.parseLong(dp.getValue()), dp.getTags())
                .addCallback(new SuccessCB())
                .addErrback(new PutErrback());
  }
```
## 2.2 RowKey的生成

metrics数据的HBase RowKey中包含主要组成部分为：盐值（Salt）、metrics名称、时间戳、tagKey、tagValue等部分。为了统一各个值的长度以及节省空间，对metrics名称、tagKey和tagValue分配了UID信息。所以，在HBase RowKey中实际写入的metrics UID、tagKey UID和tagValue UID。

rowkey的格式：

​	salt值（可配置）+3字节metrics +4字节整点时间戳+N*(3字节tagk编码+3字节tagv编码)

值：

​	分为float和long类型，数据类型通过设置到qualifier里面的flags来区分。

 

数据的写入流程从addPoint函数开始：

> net.opentsdb.core.TSDB#addPoint

这里注意一下flags

long类型的flags：

	序列化为bytes数组的长度减一
Double类型的flags：
  	final short flags = Const.FLAG_FLOAT | 0x7;  // A float stored on 8 bytes.

```
public Deferred<Object> addPoint(final String metric,
                                 final long timestamp,
                                 final long value,
                                 final Map<String, String> tags) {
  final byte[] v;
  if (Byte.MIN_VALUE <= value && value <= Byte.MAX_VALUE) {
    v = new byte[] { (byte) value };
  } else if (Short.MIN_VALUE <= value && value <= Short.MAX_VALUE) {
    v = Bytes.fromShort((short) value);
  } else if (Integer.MIN_VALUE <= value && value <= Integer.MAX_VALUE) {
    v = Bytes.fromInt((int) value);
  } else {
    v = Bytes.fromLong(value);
  }

  final short flags = (short) (v.length - 1);  // Just the length.
  return addPointInternal(metric, timestamp, v, tags, flags);
}
```

addPointInternal：

1.生成rowkey

2.生成Qualifier

3.调用写入数据库函数

4.float类型的数值，会通过如下变为int类型的表示：

	Bytes.fromInt(Float.floatToRawIntBits(value))

```
final Deferred<Object> addPointInternal(final String metric,
    final long timestamp,
    final byte[] value,
    final Map<String, String> tags,
    final short flags) {

  checkTimestampAndTags(metric, timestamp, value, tags, flags);
  final byte[] row = IncomingDataPoints.rowKeyTemplate(this, metric, tags);
  
  final byte[] qualifier = Internal.buildQualifier(timestamp, flags);

  return storeIntoDB(metric, timestamp, value, tags, flags, row, qualifier);
}
```

net.opentsdb.core.IncomingDataPoints#rowKeyTemplate

这里很简单：

1.跳过如果有SALT，则跳过SALT的位置

2.把metric的uid拷贝进去

3.跳过TIMESTAMP_BYTES

4.把所有的tagk/tagv对copy进去

5.如果metric和tagk/tagv对应的UID没有，那么创建

6.注意在tags.resolveAllInternal的最后一行:把所有的tagk的uid进行了二进制排序！

```
static byte[] rowKeyTemplate(final TSDB tsdb, final String metric,
    final Map<String, String> tags) {
  final short metric_width = tsdb.metrics.width();
  final short tag_name_width = tsdb.tag_names.width();
  final short tag_value_width = tsdb.tag_values.width();
  final short num_tags = (short) tags.size();

  int row_size = (Const.SALT_WIDTH() + metric_width + Const.TIMESTAMP_BYTES 
      + tag_name_width * num_tags + tag_value_width * num_tags);
  final byte[] row = new byte[row_size];

  short pos = (short) Const.SALT_WIDTH();

  copyInRowKey(row, pos,
      (tsdb.config.auto_metric() ? tsdb.metrics.getOrCreateId(metric)
          : tsdb.metrics.getId(metric)));
  pos += metric_width;

  pos += Const.TIMESTAMP_BYTES;

  for (final byte[] tag : Tags.resolveOrCreateAll(tsdb, tags)) {
    copyInRowKey(row, pos, tag);
    pos += tag.length;
  }
  return row;
}

net.opentsdb.core.Tags
  static ArrayList<byte[]> resolveOrCreateAll(final TSDB tsdb,
                                              final Map<String, String> tags) {
    return resolveAllInternal(tsdb, tags, true);
  }
  private
  static ArrayList<byte[]> resolveAllInternal(final TSDB tsdb,
                                              final Map<String, String> tags,
                                              final boolean create)
    throws NoSuchUniqueName {
    final ArrayList<byte[]> tag_ids = new ArrayList<byte[]>(tags.size());
    for (final Map.Entry<String, String> entry : tags.entrySet()) {
      final byte[] tag_id = (create && tsdb.getConfig().auto_tagk()
                             ? tsdb.tag_names.getOrCreateId(entry.getKey())
                             : tsdb.tag_names.getId(entry.getKey()));
      final byte[] value_id = (create && tsdb.getConfig().auto_tagv()
                               ? tsdb.tag_values.getOrCreateId(entry.getValue())
                               : tsdb.tag_values.getId(entry.getValue()));
      final byte[] thistag = new byte[tag_id.length + value_id.length];
      System.arraycopy(tag_id, 0, thistag, 0, tag_id.length);
      System.arraycopy(value_id, 0, thistag, tag_id.length, value_id.length);
      tag_ids.add(thistag);
    }
    // Now sort the tags.
    Collections.sort(tag_ids, Bytes.MEMCMP);
    return tag_ids;
  }
  
```

## 

## 2.3 Salt的生成

1.分成固定的20个桶（SALT_BUCKETS=20）

2.global annotation 的Salt全为0

3.根据rowkey里面的metrics和tagk/tagv对计算hash值，对SALT_BUCKETS取模

4.对取模后的值进行位运算，获取最后的SALT key

```
public static void prefixKeyWithSalt(final byte[] row_key) {
  if (Const.SALT_WIDTH() > 0) {
    if (row_key.length < (Const.SALT_WIDTH() + TSDB.metrics_width()) || 
      (Bytes.memcmp(row_key, new byte[Const.SALT_WIDTH() + TSDB.metrics_width()], 
          Const.SALT_WIDTH(), TSDB.metrics_width()) == 0)) {
      // ^ Don't salt the global annotation row, leave it at zero
      return;
    }
    final int tags_start = Const.SALT_WIDTH() + TSDB.metrics_width() + 
        Const.TIMESTAMP_BYTES;
    
    // we want the metric and tags, not the timestamp
    final byte[] salt_base = 
        new byte[row_key.length - Const.SALT_WIDTH() - Const.TIMESTAMP_BYTES];
    System.arraycopy(row_key, Const.SALT_WIDTH(), salt_base, 0, TSDB.metrics_width());
    System.arraycopy(row_key, tags_start,salt_base, TSDB.metrics_width(), 
        row_key.length - tags_start);
    int modulo = Arrays.hashCode(salt_base) % Const.SALT_BUCKETS();
    if (modulo < 0) {
      // make sure we return a positive salt.
      modulo = modulo * -1;
    }
  
    final byte[] salt = getSaltBytes(modulo);
    System.arraycopy(salt, 0, row_key, 0, Const.SALT_WIDTH());
  } // else salting is disabled so it's a no-op
}

  public static byte[] getSaltBytes(final int bucket) {
    final byte[] bytes = new byte[Const.SALT_WIDTH()];
    int shift = 0;
    for (int i = 1;i <= Const.SALT_WIDTH(); i++) {
      bytes[Const.SALT_WIDTH() - i] = (byte) (bucket >>> shift);
      shift += 8;
    }
    return bytes;
  }
```

## 2.4 数据保存

> storeIntoDB
>

分为三步：

1。把第一步剩下的salt和timesatmp填充到rowkey

2. 如果开启Append模式，调用Append写入；否则，是普通的数据写入，生成PutRequest
3. 如果开启元数据，写入时间线元数据

```
private final Deferred<Object> storeIntoDB(final String metric, 
                                           final long timestamp, 
                                           final byte[] value,
                                           final Map<String, String> tags, 
                                           final short flags,
                                           final byte[] row, 
                                           final byte[] qualifier) {
  final long base_time;

  if ((timestamp & Const.SECOND_MASK) != 0) {
    // drop the ms timestamp to seconds to calculate the base timestamp
    base_time = ((timestamp / 1000) -
        ((timestamp / 1000) % Const.MAX_TIMESPAN));
  } else {
    base_time = (timestamp - (timestamp % Const.MAX_TIMESPAN));
  }

  /** Callback executed for chaining filter calls to see if the value
   * should be written or not. */
  final class WriteCB implements Callback<Deferred<Object>, Boolean> {
    @Override
    public Deferred<Object> call(final Boolean allowed) throws Exception {
      if (!allowed) {
        rejected_dps.incrementAndGet();
        return Deferred.fromResult(null);
      }
      
      //第一步：填充完全RowKey
      
      //第二步：处理Append模式
      //或者：处理普通的数据写入
      //或者： 直方图数据的写入
        
      } else {
        scheduleForCompaction(row, (int) base_time);
        final PutRequest histo_point = new PutRequest(table, row, FAMILY, qualifier, value);
        result = client.put(histo_point);
      }

      // Count all added datapoints, not just those that came in through PUT rpc
      // Will there be others? Well, something could call addPoint programatically right?
      datapoints_added.incrementAndGet();

      // TODO(tsuna): Add a callback to time the latency of HBase and store the
      // timing in a moving Histogram (once we have a class for this).

      if (!config.enable_realtime_ts() && !config.enable_tsuid_incrementing() &&
          !config.enable_tsuid_tracking() && rt_publisher == null) {
        return result;
      }
      
      //第三步：写入时间线到元数据
      final byte[] tsuid = UniqueId.getTSUIDFromKey(row, METRICS_WIDTH,
          Const.TIMESTAMP_BYTES);

      // if the meta cache plugin is instantiated then tracking goes through it
      if (meta_cache != null) {
        meta_cache.increment(tsuid);
      } else {
        if (config.enable_tsuid_tracking()) {
          if (config.enable_realtime_ts()) {
            if (config.enable_tsuid_incrementing()) {
              TSMeta.incrementAndGetCounter(TSDB.this, tsuid);
            } else {
              TSMeta.storeIfNecessary(TSDB.this, tsuid);
            }
          } else {
            final PutRequest tracking = new PutRequest(meta_table, tsuid,
                TSMeta.FAMILY(), TSMeta.COUNTER_QUALIFIER(), Bytes.fromLong(1));
            client.put(tracking);
          }
        }
      }

      if (rt_publisher != null) {
        if (isHistogram(qualifier)) {
          rt_publisher.publishHistogramPoint(metric, timestamp, value, tags, tsuid);
        } else {
          rt_publisher.sinkDataPoint(metric, timestamp, value, tags, tsuid, flags);
        }
      }
      return result;
    }
    @Override
    public String toString() {
      return "addPointInternal Write Callback";
    }
  }

  if (ts_filter != null && ts_filter.filterDataPoints()) {
    if (isHistogram(qualifier)) {
      return ts_filter.allowHistogramPoint(metric, timestamp, value, tags)
              .addCallbackDeferring(new WriteCB());
    } else {
      return ts_filter.allowDataPoint(metric, timestamp, value, tags, flags)
              .addCallbackDeferring(new WriteCB());
    }
  }
  return Deferred.fromResult(true).addCallbackDeferring(new WriteCB());
}
```

第一步：填充完全RowKey

      //第一步：填充完全RowKey
      Bytes.setInt(row, (int) base_time, metrics.width() + Const.SALT_WIDTH());
      RowKey.prefixKeyWithSalt(row);

第二步：处理Append模式

Append的QUALIFIER 就是0x050000

```
  /** The full column qualifier for append columns */
  public static final byte[] APPEND_COLUMN_QUALIFIER = new byte[] {
    APPEND_COLUMN_PREFIX, 0x00, 0x00};


      if (!isHistogram(qualifier) && config.enable_appends()) {
        if(config.use_otsdb_timestamp()) {
            LOG.error("Cannot use Date Tiered Compaction with AppendPoints. Please turn off either of them.");
        }
        final AppendDataPoints kv = new AppendDataPoints(qualifier, value);
        final AppendRequest point = new AppendRequest(table, row, FAMILY,
                AppendDataPoints.APPEND_COLUMN_QUALIFIER, kv.getBytes());
        result = client.append(point);
      } 
```

或者处理普通模式：

```
scheduleForCompaction(row, (int) base_time);
final PutRequest point = RequestBuilder.buildPutRequest(config, table, row, FAMILY, qualifier, value, timestamp);
result = client.put(point);

这里针对时间戳进行了特殊处理：
    public static PutRequest buildPutRequest(Config config, byte[] tableName, byte[] row, byte[] family, byte[] qualifier, byte[] value, long timestamp) {
        
        if(config.use_otsdb_timestamp()) 
            if((timestamp & Const.SECOND_MASK) != 0)
                return new PutRequest(tableName, row, family, qualifier, value, timestamp);
            else
                return new PutRequest(tableName, row, family, qualifier, value, timestamp * 1000);
        else
            return new PutRequest(tableName, row, family, qualifier, value);
    }
```

注意这里会调度Compaction，而这里仅仅是把他加入到一个队列里面去，另外一个线程单独调度Compaction

```
final void scheduleForCompaction(final byte[] row, final int base_time) {
  if (config.enable_compactions()) {
    compactionq.add(row);
  }
}
```

## 2.5 时间线元数据的梳理

自己看把，代码也比较简单，和几个配置相关





# 3.查询涉及到的对象以及包含关系



HttpQUery------>^1^TSQuery

## 2.1查询对象

> net.opentsdb.core.TSQuery
>
> 查询对象

```javascript
public final class TSQuery {

  /** User given start date/time, could be relative or absolute */
  private String start;
  
  /** User given end date/time, could be relative, absolute or empty */
  private String end;
  
  /** User's timezone used for converting absolute human readable dates */
  private String timezone;
  
  /** Options for serializers, graphs, etc */
  private HashMap<String, ArrayList<String>> options;
  
  /** 
   * Whether or not to include padding, i.e. data to either side of the start/
   * end dates
   */
  private boolean padding;
  
  /** Whether or not to suppress annotation output */
  private boolean no_annotations;
  
  /** Whether or not to scan for global annotations in the same time range */
  private boolean with_global_annotations;

  /** Whether or not to show TSUIDs when returning data */
  private boolean show_tsuids;
  
  /** A list of parsed sub queries, must have one or more to fetch data */
  private ArrayList<TSSubQuery> queries;

  //下面两个变量是在validateAndSetQuery函数里面计算出来的！
  /** The parsed start time value 
   * <b>Do not set directly</b> */
  private long start_time;
  
  /** The parsed end time value 
   * <b>Do not set directly</b> */
  private long end_time;
  
  /** Whether or not the user wasn't millisecond resolution */
  private boolean ms_resolution;
  
  /** Whether or not to show the sub query with the results */
  private boolean show_query;
  
  /** Whether or not to include stats in the output */
  private boolean show_stats;
  
  /** Whether or not to include stats summary in the output */
  private boolean show_summary;
  
  /** Whether or not to delete the queried data */
  private boolean delete = false;
  
  /** A flag denoting whether or not to align intervals based on the calendar */
  private boolean use_calendar;
```

POST格式的示例数据为：（可以根据上面的增加自己需要的设置项，此处没有写全）

```
{
    "start": 1356998400,
    "end": 1356998460,
    "queries": [
        {
            "aggregator": "sum",
            "metric": "sys.cpu.0",
            "rate": "true",
            "filters": [
                {
                   "type":"wildcard",
                   "tagk":"host",
                   "filter":"*",
                   "groupBy":true
                },
                {
                   "type":"literal_or",
                   "tagk":"dc",
                   "filter":"lga|lga1|lga2",
                   "groupBy":false
                }
            ]
        },
        {
            "aggregator": "sum",
            "tsuids": [
                "000001000002000042",
                "000001000002000043"
            ]
        }
    ]
}
```



## 2.2子查询对象

net.opentsdb.core.TSSubQuery

TSSubQuery

```
public final class TSSubQuery {
  /** User given name of an aggregation function to use */
  private String aggregator;
  
  /** User given name for a metric, e.g. "sys.cpu.0" */
  private String metric;
  
  /** User provided list of timeseries UIDs */
  private List<String> tsuids;

  /** User given downsampler */
  private String downsample;
  
  /** Whether or not the user wants to perform a rate conversion */
  private boolean rate;
  
  /** Rate options for counter rollover/reset */
  private RateOptions rate_options;
  
  /** Parsed aggregation function */
  private Aggregator agg;
  
  /** Parsed downsampling specification. */
  private DownsamplingSpecification downsample_specifier;
  
  /** A list of filters for this query. For now these are pulled out of the
   * tags map. In the future we'll have special JSON objects for them. */
  private List<TagVFilter> filters;
  
  /** Whether or not to match series with ONLY the given tags */
  private boolean explicit_tags;
  
  /** Index of the sub query */
  private int index;
```

## 2.3过滤器

> net.opentsdb.query.filter.TagVFilter
>
> 过滤器对象

### 主要成员变量

可以看出，一个过滤器，只针对一个tagk，但是可以针对多个tagv，比如city=literal_or(guangzhou,shanghai)

group_by是个很重要的变量，标识这个tagk是否参与汇总。在聚合时会使用到。

```
@JsonDeserialize(builder = TagVFilter.Builder.class)
public abstract class TagVFilter implements Comparable<TagVFilter> {
/** The tag key this filter is associated with */
  final protected String tagk;
  
  /** The raw, unparsed filter */
  final protected String filter;
  
  /** The tag key converted into a UID */
  protected byte[] tagk_bytes;
  
  /** An optional list of tag value UIDs if the filter matches on literals. */
  protected List<byte[]> tagv_uids;
  
  /** Whether or not to also group by this filter */
  @JsonProperty
  protected boolean group_by;
  
  /** A flag to indicate whether or not we need to execute a post-scan lookup */
  protected boolean post_scan = true;
  。。。
  
 }
```

### 过滤器的序列化：

其中TagVFilter实现了自己的序列化函数：

注意其中的 buildMethodName = "build", withPrefix = "set"

```
/**
 * Builder class used for deserializing filters from JSON queries via Jackson
 * since we don't want the user to worry about the class name. The type,
 * tagk and filter must be configured or the build will fail.
 */
@JsonIgnoreProperties(ignoreUnknown = true)
@JsonPOJOBuilder(buildMethodName = "build", withPrefix = "set")
public static class Builder {
  private String type;
  private String tagk;
  private String filter;
  @JsonProperty
  private boolean group_by;
  
  /** @param type The type of filter matching a valid filter name */
  public Builder setType(final String type) {
    this.type = type;
    return this;
  }
  
  /** @param tagk The tag key to match on for this filter */
  public Builder setTagk(final String tagk) {
    this.tagk = tagk;
    return this;
  }

  /** @param filter The filter expression to use for matching */
  public Builder setFilter(final String filter) {
    this.filter = filter;
    return this;
  }
  
  /** @param group_by Whether or not the filter should group results */
  public Builder setGroupBy(final boolean group_by) {
    this.group_by = group_by;
    return this;
  }
  
  /**
   * Searches the filter map for the given type and returns an instantiated
   * filter if found. The caller must set the type, tagk and filter values.
   * @return A filter if instantiation was successful
   * @throws IllegalArgumentException if one of the required parameters was
   * not set or the filter couldn't be found.
   * @throws RuntimeException if the filter couldn't be instantiated. Check
   * the implementation if it's a plugin.
   */
  public TagVFilter build() { 
    if (type == null || type.isEmpty()) {
      throw new IllegalArgumentException(
          "The filter type cannot be null or empty");
    }
    if (tagk == null || tagk.isEmpty()) {
      throw new IllegalArgumentException(
          "The tagk cannot be null or empty");
    }
    
    final Pair<Class<?>, Constructor<? extends TagVFilter>> filter_meta = 
        tagv_filter_map.get(type);
    if (filter_meta == null) {
      throw new IllegalArgumentException(
          "Could not find a tag value filter of the type: " + type);
    }
    final Constructor<? extends TagVFilter> ctor = filter_meta.getValue();
    final TagVFilter tagv_filter;
    try {
      tagv_filter = ctor.newInstance(tagk, filter);
    } catch (IllegalArgumentException e) {
      throw e;
    } catch (InstantiationException e) {
      throw new RuntimeException("Failed to instantiate filter: " + type, e);
    } catch (IllegalAccessException e) {
      throw new RuntimeException("Failed to instantiate filter: " + type, e);
    } catch (InvocationTargetException e) {
      if (e.getCause() != null) {
        throw (RuntimeException)e.getCause();
      }
      throw new RuntimeException("Failed to instantiate filter: " + type, e);
    }
    
    tagv_filter.setGroupBy(group_by);
    return tagv_filter;
  }
}
```



### 过滤器的排序：

过滤器是根据tagk的二进制编码进行比较的：

```
@Override
public int compareTo(final TagVFilter filter) {
  return Bytes.memcmpMaybeNull(tagk_bytes, filter.tagk_bytes);
}
```



### 过滤器实现的功能：

1.解析tagk，tagkv，获取各自的二进制编码

2.提供过滤功能，主要是各个子类需要实现虚函数：

```java
public abstract Deferred<Boolean> match(final Map<String, String> tags);
```

## 2.4降采样规格



> 降采样规格
>
> net.opentsdb.core.DownsamplingSpecification

降采样的格式：interval-function[-fill_policy]

```
/**
   * C-tor for string representations.
   * The argument to this c-tor should have the following format:
   * {@code interval-function[-fill_policy]}.
   * This ctor supports the "all" flag to downsample to a single value as well
   * as units suffixed with 'c' to use the calendar for downsample alignment.
   * @param specification String representation of a downsample specifier.
   * @throws IllegalArgumentException if the specification is null or invalid.
   */
  public DownsamplingSpecification(final String specification) {
    if (null == specification) {
      throw new IllegalArgumentException("Downsampling specifier cannot be " +
        "null");
    }

    final String[] parts = specification.split("-");
    if (parts.length < 2) {
      // Too few items.
      throw new IllegalArgumentException("Invalid downsampling specifier '" +
        specification + "': must provide at least interval and function");
    } else if (parts.length > 3) {
      // Too many items.
      throw new IllegalArgumentException("Invalid downsampling specifier '" +
        specification + "': must consist of interval, function, and optional " +
        "fill policy");
    }

    // This porridge is just right.

    // INTERVAL.
    // This will throw if interval is invalid.
    if (parts[0].contains("all")) {
      interval = NO_INTERVAL;
      use_calendar = false;
      string_interval = parts[0];
    } else if (parts[0].charAt(parts[0].length() - 1) == 'c') {
      final String duration = parts[0].substring(0, parts[0].length() - 1);
      interval = DateTime.parseDuration(duration);
      string_interval = duration;
      use_calendar = true;
    } else {
      interval = DateTime.parseDuration(parts[0]);
      use_calendar = false;
      string_interval = parts[0];
    }

    // FUNCTION.
    try {
      function = Aggregators.get(parts[1]);
    } catch (final NoSuchElementException e) {
      throw new IllegalArgumentException("No such downsampling function: " +
        parts[1]);
    }
    if (function == Aggregators.NONE) {
      throw new IllegalArgumentException("cannot use the NONE "
          + "aggregator for downsampling");
    }

    // FILL POLICY.
    if (3 == parts.length) {
      // If the user gave us three parts, then the third must be a fill
      // policy.
      fill_policy = FillPolicy.fromString(parts[2]);
      if (null == fill_policy) {
        final StringBuilder oss = new StringBuilder();
        oss.append("No such fill policy: '").append(parts[2])
           .append("': must be one of:");
        for (final FillPolicy policy : FillPolicy.values()) {
          oss.append(" ").append(policy.getName());
        }

        throw new IllegalArgumentException(oss.toString());
      }
    } else {
      // Default to linear interpolation.
      fill_policy = FillPolicy.NONE;
    }
    timezone = DateTime.timezones.get(DateTime.UTC_ID);
  }
```

# 4.Opentsdb 请求解析

## Opentsdb 查询时间解析



TSQuery.start_time表示查询的开始时间，以==***<u>毫秒</u>***==为单位

> 类：net.opentsdb.core.TSQuery
>
> 函数：validateAndSetQuery

```java
start_time = DateTime.parseDateTimeString(start, timezone);
```

支持的日期格式如下：

> 类：net.opentsdb.utils.DateTime
>
> 函数：parseDateTimeString

```
/**
 * Attempts to parse a timestamp from a given string
 * Formats accepted are:
 * <ul>
 * <li>Relative: {@code 5m-ago}, {@code 1h-ago}, etc. See 
 * {@link #parseDuration}</li>
 * <li>Absolute human readable dates:
 * <ul><li>"yyyy/MM/dd-HH:mm:ss"</li>
 * <li>"yyyy/MM/dd HH:mm:ss"</li>
 * <li>"yyyy/MM/dd-HH:mm"</li>
 * <li>"yyyy/MM/dd HH:mm"</li>
 * <li>"yyyy/MM/dd"</li></ul></li>
 * <li>Unix Timestamp in seconds or milliseconds: 
 * <ul><li>1355961600</li>
 * <li>1355961600000</li>
 * <li>1355961600.000</li></ul></li>
 * </ul>
 * @param datetime The string to parse a value for
 * @return A Unix epoch timestamp in milliseconds
 * @throws NullPointerException if the timestamp is null
 * @throws IllegalArgumentException if the request was malformed 
 */
 
 还支持：
 now,1355961600000ms等
```



解析代码如下:

```
public static final long parseDateTimeString(final String datetime, 
    final String tz) {
  if (datetime == null || datetime.isEmpty())
    return -1;

  //时间戳，毫秒
  if (datetime.matches("^[0-9]+ms$")) {
    return Tags.parseLong(datetime.replaceFirst("^([0-9]+)(ms)$", "$1"));
  }
 //now字符串
  if (datetime.toLowerCase().equals("now")) {
    return System.currentTimeMillis();
  }

  if (datetime.toLowerCase().endsWith("-ago")) {
    long interval = DateTime.parseDuration(
      datetime.substring(0, datetime.length() - 4));
    return System.currentTimeMillis() - interval;
  }
  
  不同格式的日期的支持
  if (datetime.contains("/") || datetime.contains(":")) {
    try {
      SimpleDateFormat fmt = null;
      switch (datetime.length()) {
        // these were pulled from cliQuery but don't work as intended since 
        // they assume a date of 1970/01/01. Can be fixed but may not be worth
        // it
        // case 5:
        //   fmt = new SimpleDateFormat("HH:mm");
        //   break;
        // case 8:
        //   fmt = new SimpleDateFormat("HH:mm:ss");
        //   break;
        case 10:
          fmt = new SimpleDateFormat("yyyy/MM/dd");
          break;
        case 16:
          if (datetime.contains("-"))
            fmt = new SimpleDateFormat("yyyy/MM/dd-HH:mm");
          else
            fmt = new SimpleDateFormat("yyyy/MM/dd HH:mm");
          break;
        case 19:
          if (datetime.contains("-"))
            fmt = new SimpleDateFormat("yyyy/MM/dd-HH:mm:ss");
          else
            fmt = new SimpleDateFormat("yyyy/MM/dd HH:mm:ss");
          break;
        default:
          // todo - deal with internationalization, other time formats
          throw new IllegalArgumentException("Invalid absolute date: " 
              + datetime);
      }
      if (tz != null && !tz.isEmpty())
        setTimeZone(fmt, tz);
      return fmt.parse(datetime).getTime();
    } catch (ParseException e) {
      throw new IllegalArgumentException("Invalid date: " + datetime  
          + ". " + e.getMessage());
    }
  } else {
    try {
      //毫秒字符串的解析。两种格式：秒.毫秒，或者秒
      这里有个坑：如果是秒数，必须位数小于10，因为date +%s就是10位
      	date +%s
		1535555680

      long time;
      final boolean contains_dot = datetime.contains(".");
      // [0-9]{10} ten digits
      // \\. a dot
      // [0-9]{1,3} one to three digits
      final boolean valid_dotted_ms = 
          datetime.matches("^[0-9]{10}\\.[0-9]{1,3}$");
      if (contains_dot) {
        if (!valid_dotted_ms) {
          throw new IllegalArgumentException("Invalid time: " + datetime  
              + ". Millisecond timestamps must be in the format "
              + "<seconds>.<ms> where the milliseconds are limited to 3 digits");
        }
        time = Tags.parseLong(datetime.replace(".", ""));   
      } else {
        time = Tags.parseLong(datetime);
      }
      if (time < 0) {
        throw new IllegalArgumentException("Invalid time: " + datetime  
            + ". Negative timestamps are not supported.");
      }
      // this is a nasty hack to determine if the incoming request is
      // in seconds or milliseconds. This will work until November 2286
      if (datetime.length() <= 10) {
        time *= 1000;
      }
      return time;
    } catch (NumberFormatException e) {
      throw new IllegalArgumentException("Invalid time: " + datetime  
          + ". " + e.getMessage());
    }
  }
}
```





解析相对时间的函数：

> 类：net.opentsdb.utils.DateTime
>
> 函数：parseDuration

```
/**
 * Parses a human-readable duration (e.g, "10m", "3h", "14d") into seconds.
 * <p>
 * Formats supported:<ul>
 * <li>{@code ms}: milliseconds</li>
 * <li>{@code s}: seconds</li>
 * <li>{@code m}: minutes</li>
 * <li>{@code h}: hours</li>
 * <li>{@code d}: days</li>
 * <li>{@code w}: weeks</li> 
 * <li>{@code n}: month (30 days)</li>
 * <li>{@code y}: years (365 days)</li></ul>
 * @param duration The human-readable duration to parse.
 * @return A strictly positive number of milliseconds.
 * @throws IllegalArgumentException if the interval was malformed.
 */
public static final long parseDuration(final String duration) {
  long interval;
  long multiplier;
  double temp;
  int unit = 0;
  
  //提取出所有的数字
  while (Character.isDigit(duration.charAt(unit))) {
    unit++;
    if (unit >= duration.length()) {
      throw new IllegalArgumentException("Invalid duration, must have an "
          + "integer and unit: " + duration);
    }
  }
  try {
    interval = Long.parseLong(duration.substring(0, unit));
  } catch (NumberFormatException e) {
    throw new IllegalArgumentException("Invalid duration (number): " + duration);
  }
  if (interval <= 0) {
    throw new IllegalArgumentException("Zero or negative duration: " + duration);
  }
  
  //判断时间单位，根据单位的力度，换算成毫秒
  
  switch (duration.toLowerCase().charAt(duration.length() - 1)) {
    case 's': 
      if (duration.charAt(duration.length() - 2) == 'm') {
        return interval;
      }
      multiplier = 1; break;                        // seconds
    case 'm': multiplier = 60; break;               // minutes
    case 'h': multiplier = 3600; break;             // hours
    case 'd': multiplier = 3600 * 24; break;        // days
    case 'w': multiplier = 3600 * 24 * 7; break;    // weeks
    case 'n': multiplier = 3600 * 24 * 30; break;   // month (average)
    case 'y': multiplier = 3600 * 24 * 365; break;  // years (screw leap years)
    default: throw new IllegalArgumentException("Invalid duration (suffix): " + duration);
  }
  multiplier *= 1000;
  temp = (double)interval * multiplier;
  if (temp > Long.MAX_VALUE) {
    throw new IllegalArgumentException("Duration must be < Long.MAX_VALUE ms: " + duration);
  }
  return interval * multiplier;
}
```

## Opentsdb 查询解析

> 类：net.opentsdb.tsd.QueryRpc
>
> 函数：handleQuery

```java
final TSQuery data_query;
final List<ExpressionTree> expressions;

//POST：直接反序列化为TSQuery

if (query.method() == HttpMethod.POST) {
  switch (query.apiVersion()) {
  case 0:
  case 1:
    data_query = query.serializer().parseQueryV1();
    break;
  default:
    query_invalid.incrementAndGet();
    throw new BadRequestException(HttpResponseStatus.NOT_IMPLEMENTED, 
        "Requested API version not implemented", "Version " + 
        query.apiVersion() + " is not implemented");
  }
  expressions = null;
} else {

  //Get：需要解析为TSQuery对象
  expressions = new ArrayList<ExpressionTree>();
  data_query = parseQuery(tsdb, query, expressions);
}
```

## 



### 反序列化



#### POST：



```
if (query.method() == HttpMethod.POST) {
  switch (query.apiVersion()) {
  case 0:
  case 1:
    data_query = query.serializer().parseQueryV1();
    break;
  default:
    query_invalid.incrementAndGet();
    throw new BadRequestException(HttpResponseStatus.NOT_IMPLEMENTED, 
        "Requested API version not implemented", "Version " + 
        query.apiVersion() + " is not implemented");
  }
  expressions = null;
} else {
  
}
```

> net.opentsdb.tsd.HttpJsonSerializer#parseQueryV1
>

```
public TSQuery parseQueryV1() {
  final String json = query.getContent();
  if (json == null || json.isEmpty()) {
    throw new BadRequestException(HttpResponseStatus.BAD_REQUEST,
        "Missing message content",
        "Supply valid JSON formatted data in the body of your request");
  }
  try {
    TSQuery data_query =  JSON.parseToObject(json, TSQuery.class);
    // Filter out duplicate queries
    Set<TSSubQuery> query_set = new LinkedHashSet<TSSubQuery>(data_query.getQueries());
    data_query.getQueries().clear();
    data_query.getQueries().addAll(query_set);
    return data_query;
  } catch (IllegalArgumentException iae) {
    throw new BadRequestException("Unable to parse the given JSON", iae);
  }
}
```

#### Get



//http://10.251.44.120:8153/api/query?start=2015/01/01-00:00:00&end=2016/01/01-00:00:00
  // &m=sum:temperature{city=guangzhou,zip_code=*}{value=gt(16)}
  // &m=sum:humidity{city=beijing,zip_code=*}



```java
expressions = new ArrayList<ExpressionTree>();
  //首先把HttpQuery对象转换为TSQuery对象，HttpQuery对象是请求参数，TSQuery对象理解为解析后的参数对象
  data_query = parseQuery(tsdb, query, expressions);
```



```
public static TSQuery parseQuery(final TSDB tsdb, final HttpQuery query,
    final List<ExpressionTree> expressions) {
  final TSQuery data_query = new TSQuery();
  
  data_query.setStart(query.getRequiredQueryStringParam("start"));
  data_query.setEnd(query.getQueryStringParam("end"));
  
  if (query.hasQueryStringParam("padding")) {
    data_query.setPadding(true);
  }
  
  if (query.hasQueryStringParam("no_annotations")) {
    data_query.setNoAnnotations(true);
  }
  
  if (query.hasQueryStringParam("global_annotations")) {
    data_query.setGlobalAnnotations(true);
  }
  
  if (query.hasQueryStringParam("show_tsuids")) {
    data_query.setShowTSUIDs(true);
  }
  
  if (query.hasQueryStringParam("ms")) {
    data_query.setMsResolution(true);
  }
  
  if (query.hasQueryStringParam("show_query")) {
    data_query.setShowQuery(true);
  }  
  
  if (query.hasQueryStringParam("show_stats")) {
    data_query.setShowStats(true);
  }    
  
  if (query.hasQueryStringParam("show_summary")) {
      data_query.setShowSummary(true);
  }
  
  // handle tsuid queries first
  if (query.hasQueryStringParam("tsuid")) {
    final List<String> tsuids = query.getQueryStringParams("tsuid");     
    for (String q : tsuids) {
      parseTsuidTypeSubQuery(q, data_query);
    }
  }

  //参数里面可以有很多m参数，每个m都是一个subquery，然后组装成一个query
  //http://10.251.44.120:8153/api/query?start=2015/01/01-00:00:00&end=2016/01/01-00:00:00
  // &m=sum:temperature{city=guangzhou,zip_code=*}{value=gt(16)}
  // &m=sum:humidity{city=beijing,zip_code=*}
  if (query.hasQueryStringParam("m")) {
    final List<String> legacy_queries = query.getQueryStringParams("m");      
    for (String q : legacy_queries) {
      parseMTypeSubQuery(q, data_query);
    }
  }
  ......


  // Filter out duplicate queries
  Set<TSSubQuery> query_set = new LinkedHashSet<TSSubQuery>(data_query.getQueries());
  data_query.getQueries().clear();
  data_query.getQueries().addAll(query_set);

  LOG.info("====================");
  LOG.info("query parsed,query object={}",data_query.toString());
  LOG.info("====================");
  return data_query;
}
```



##### TSUID的反序列化

> net.opentsdb.tsd.QueryRpc#parseTsuidTypeSubQuery

格式：agg:[interval-agg:][rate:]tsuid[,s]

第一列为聚合函数，最后一列为tsuid的列表，中间为各种其他参数：


```
private static void parseTsuidTypeSubQuery(final String query_string,
  TSQuery data_query) {
    if (query_string == null || query_string.isEmpty()) {
      throw new BadRequestException("The tsuid query string was empty");
    }
  
  // tsuid queries are of the following forms:
  // agg:[interval-agg:][rate:]tsuid[,s]
  // where the parts in square brackets `[' .. `]' are optional.
  final String[] parts = Tags.splitString(query_string, ':');
  int i = parts.length;
  if (i < 2 || i > 5) {
    throw new BadRequestException("Invalid parameter m=" + query_string + " ("
        + (i < 2 ? "not enough" : "too many") + " :-separated parts)");
  }
  
  final TSSubQuery sub_query = new TSSubQuery();

   // the aggregator is first
  sub_query.setAggregator(parts[0]);
  
  i--; // Move to the last part (the metric name).
  final List<String> tsuid_array = Arrays.asList(parts[i].split(","));
  sub_query.setTsuids(tsuid_array);
  
  // parse out the rate and downsampler 
  for (int x = 1; x < parts.length - 1; x++) {
    if (parts[x].toLowerCase().startsWith("rate")) {
      //http://opentsdb.net/docs/build/html/api_http/query/index.html
      //When passing rate options in a query string, the options must be enclosed in curly braces.
      // For example: m=sum:rate{counter,,1000}:if.octets.in. If you wish to use the default counterMax
      // but do want to supply a resetValue, you must add two commas as in the previous example.
      // Additional fields in the rateOptions object include the following:
      //
      //Name   Data Type  Required   Description    Default    Example
      //counter    Boolean    Optional   Whether or not the underlying data is a monotonically increasing counter that may roll over    false  true
      //counterMax Integer    Optional   A positive integer representing the maximum value for the counter. Java Long.MaxValue 65535
      //resetValue Integer    Optional   An optional value that, when exceeded, will cause the aggregator to return a 0 instead of the calculated rate. Useful when data sources are frequently reset to avoid spurious spikes. 0  65000
      //dropResets Boolean    Optional   Whether or not to simply drop rolled-over or reset data points.    false  true

      sub_query.setRate(true);
      if (parts[x].indexOf("{") >= 0) {
        sub_query.setRateOptions(QueryRpc.parseRateOptions(true, parts[x]));
      }
    } else if (Character.isDigit(parts[x].charAt(0))) {
      //downsample spec is:
      // <interval><units>-<aggregator>[c][-<fill policy>]
      //        For example:
      //        1h-sum
      //        30m-avg-nan
      //        24h-max-zero
      //        1dc-sum
      //        0all-sum
      sub_query.setDownsample(parts[x]);
    } else if (parts[x].toLowerCase().startsWith("percentiles")) {

      //Percentiles
      //With OpenTSDB 2.4, the database can store and query histogram or digest data for accurate percentile calculations
      // (as opposed to the built-in percentile aggregators). If one or more percentiles are requested in a query,
      // the TSD will scan storage explicitly for histograms (of any codec type) and regular numeric data will be ignored.
      // More than one percentile can be computed at the same time, for example it may be common to fetch the 99.999th, 99.9th, 99.0th
      // and 95th percentiles in one query via   percentiles[99.999, 99.9, 99.0, 95.0] .
      // NOTE For some plugin implementations (such as the Yahoo Data Sketches implementation) the percentile list must be given
      // in descending sorted order.
      //
      //Results are serialized in the same was as regular data point time series for compatibility with
      // existing graph systems. However the percentile will be appended to the metric name and time series
      // for each group-by and percentile will be returned. For example, if the user asks for percentiles[99.9,75.0]
      // over the sys.cpu.nice metric, the results will have time series sys.cpu.nice_pct_99.9 and sys.cpu.nice_pct_75.0.
      sub_query.setPercentiles(QueryRpc.parsePercentiles(parts[x]));
    } else if (parts[x].toLowerCase().startsWith("show-histogram-buckets")) {
      //这个应该是多余的，最前面聚合函数，最后面tsuid，中间有可能是上面三个？
      sub_query.setShowHistogramBuckets(true);
    }
  }
  
  if (data_query.getQueries() == null) {
    final ArrayList<TSSubQuery> subs = new ArrayList<TSSubQuery>(1);
    data_query.setQueries(subs);
  }
  data_query.getQueries().add(sub_query);
}
```



##### 度量查询的反序列化

> net.opentsdb.tsd.QueryRpc#parseMTypeSubQuery

//http://10.251.44.120:8153/api/query?start=2015/01/01-00:00:00&end=2016/01/01-00:00:00
  // &m=sum:temperature{city=guangzhou,zip_code=*}{value=gt(16)}
  // &m=sum:humidity{city=beijing,zip_code=*}



- 请求按照：拆分，拆分成聚合函数解析（最简单），选项解析，以及过滤器解析

```
private static void parseMTypeSubQuery(final String query_string, 
    TSQuery data_query) {
  if (query_string == null || query_string.isEmpty()) {
    throw new BadRequestException("The query string was empty");
  }

  LOG.info("Parsing Mtype Query_{}",query_string);

  //http://opentsdb.net/docs/build/html/api_http/query/index.html
  // m is of the following forms:
  // agg:[interval-agg:][rate:]metric[{tag=value,...}]
  // where the parts in square brackets `[' .. `]' are optional.
  final String[] parts = Tags.splitString(query_string, ':');
  int i = parts.length;
  if (i < 2 || i > 5) {
    throw new BadRequestException("Invalid parameter m=" + query_string + " ("
        + (i < 2 ? "not enough" : "too many") + " :-separated parts)");
  }
  final TSSubQuery sub_query = new TSSubQuery();
  
  // the aggregator is first
  sub_query.setAggregator(parts[0]);
  
  i--; // Move to the last part (the metric name).
  //    List<TagVFilter> filters = new ArrayList<TagVFilter>();
  //    sub_query.setMetric(Tags.parseWithMetricAndFilters(parts[i], filters));
  //    sub_query.setFilters(filters);
  //    下面是我们为了值过滤增加的逻辑
  List<TagVFilter> all_filters = new ArrayList<TagVFilter>();

  //下面这一步，接触出来的过滤器，放在all_filters中，返回的是metrics，所以是调用setMetric
  sub_query.setMetric(Tags.parseWithMetricAndFilters(parts[i], all_filters));
  List<TagVFilter> filters = new ArrayList<TagVFilter>();
  List<TagVFilter> valueFilters = new ArrayList<TagVFilter>();

  for (TagVFilter item : all_filters) {
    if( item.isFiltValue() ){
      LOG.info("add filter to ValueFilter:{}-{}",item.getTagk(),item.debugInfo());
      valueFilters.add(item);
    }else{
      LOG.info("add filter to tagFilter:{}-{}",item.getTagk(),item.debugInfo());
      filters.add(item);
    }
  }

  sub_query.setFilters(filters);
  sub_query.setValueFilters(valueFilters);

  //前面有>=2<=5的限制，这里有6个选项，加上聚合函数和度量，一共可有8个，矛盾啊
  // parse out the rate and downsampler 
  for (int x = 1; x < parts.length - 1; x++) {
    if (parts[x].toLowerCase().startsWith("rate")) {
      sub_query.setRate(true);

      //http://opentsdb.net/docs/build/html/api_http/query/index.html
      //When passing rate options in a query string, the options must be enclosed in curly braces.
      // For example: m=sum:rate{counter,,1000}:if.octets.in. If you wish to use the default counterMax
      // but do want to supply a resetValue, you must add two commas as in the previous example.
      // Additional fields in the rateOptions object include the following:
      //
      //Name   Data Type  Required   Description    Default    Example
      //counter    Boolean    Optional   Whether or not the underlying data is a monotonically increasing counter that may roll over    false  true
      //counterMax Integer    Optional   A positive integer representing the maximum value for the counter. Java Long.MaxValue 65535
      //resetValue Integer    Optional   An optional value that, when exceeded, will cause the aggregator to return a 0 instead of the calculated rate. Useful when data sources are frequently reset to avoid spurious spikes. 0  65000
      //dropResets Boolean    Optional   Whether or not to simply drop rolled-over or reset data points.    false  true

      if (parts[x].indexOf("{") >= 0) {
        sub_query.setRateOptions(QueryRpc.parseRateOptions(true, parts[x]));
      }
    } else if (Character.isDigit(parts[x].charAt(0))) {
      //downsample spec is:
      // <interval><units>-<aggregator>[c][-<fill policy>]
      //        For example:
      //        1h-sum
      //        30m-avg-nan
      //        24h-max-zero
      //        1dc-sum
      //        0all-sum
      sub_query.setDownsample(parts[x]);
    } else if (parts[x].equalsIgnoreCase("pre-agg")) {
      //他妈的这个文档里面都没有说明啊！
      //http://opentsdb.net/docs/build/html/api_http/query/index.html
      //参见http://opentsdb.net/docs/build/html/user_guide/rollups.html?highlight=pre%20agg
      sub_query.setPreAggregate(true);
    } else if (parts[x].toLowerCase().startsWith("rollup_")) {
      //他妈的这个文档里面都没有说明啊！
      //http://opentsdb.net/docs/build/html/api_http/query/index.html
      //参见http://opentsdb.net/docs/build/html/user_guide/rollups.html?highlight=pre%20agg
      sub_query.setRollupUsage(parts[x]);
    } else if (parts[x].toLowerCase().startsWith("percentiles")) {
      //Percentiles
      //With OpenTSDB 2.4, the database can store and query histogram or digest data for accurate percentile calculations
      // (as opposed to the built-in percentile aggregators). If one or more percentiles are requested in a query,
      // the TSD will scan storage explicitly for histograms (of any codec type) and regular numeric data will be ignored.
      // More than one percentile can be computed at the same time, for example it may be common to fetch the 99.999th, 99.9th, 99.0th
      // and 95th percentiles in one query via   percentiles[99.999, 99.9, 99.0, 95.0] .
      // NOTE For some plugin implementations (such as the Yahoo Data Sketches implementation) the percentile list must be given
      // in descending sorted order.
      //
      //Results are serialized in the same was as regular data point time series for compatibility with
      // existing graph systems. However the percentile will be appended to the metric name and time series
      // for each group-by and percentile will be returned. For example, if the user asks for percentiles[99.9,75.0]
      // over the sys.cpu.nice metric, the results will have time series sys.cpu.nice_pct_99.9 and sys.cpu.nice_pct_75.0.

      sub_query.setPercentiles(QueryRpc.parsePercentiles(parts[x]));
    } else if (parts[x].toLowerCase().startsWith("show-histogram-buckets")) {
      //文档里面都他妈找不到！
      //http://opentsdb.net/docs/build/html/user_guide/rollups.html?highlight=pre%20agg查抄
      sub_query.setShowHistogramBuckets(true);
    } else if (parts[x].toLowerCase().startsWith("explicit_tags")) {
      //Explicit Tags
      //http://10.251.44.120:8153/api/query?start=2015/01/01-00:00:00&end=2016/01/01-00:00:00&m=sum:explicit_tags:temperature{city=*,zip_code=510000,latitude=*,longitude=113.325039}{value=gt(16)}
      //是不是可以这样理解=，就是把所有的tags都写上?这样确实可以减少数据量。tagv可以使用*，但是tagk必须全部出现！？
      //As of 2.3 and later, if you know all of the tag keys for a given metric query latency can be improved greatly
      // by using the explicitTags feature. This flag has two benefits:
      //
      //For metrics that have a high cardinality, the backend can switch to a more efficient query to fetch a smaller subset of data
      // from storage. (Particularly in 2.4)
      //For metrics with varying tags, this can be used to avoid aggregating time series that should not be included in the final result.
      //Explicit tags will craft an underlying storage query that fetches only those rows with the given tag keys.
      // That can allow the database to skip over irrelevant rows and answer in less time.
      //
      sub_query.setExplicitTags(true);
    }
  }
  
  if (data_query.getQueries() == null) {
    final ArrayList<TSSubQuery> subs = new ArrayList<TSSubQuery>(1);
    data_query.setQueries(subs);
  }
  data_query.getQueries().add(sub_query);
}

```

- 过滤器解析

  反序列化过滤器，同时返回度量的名称

> net.opentsdb.core.Tags#parseWithMetricAndFilters

```java
public static String parseWithMetricAndFilters(final String metric, 
    final List<TagVFilter> filters) {
  if (metric == null || metric.isEmpty()) {
    throw new IllegalArgumentException("Metric cannot be null or empty");
  }
  if (filters == null) {
    throw new IllegalArgumentException("Filters cannot be null");
  }
  final int curly = metric.indexOf('{');
  if (curly < 0) {
    return metric;
  }

  //http://opentsdb.net/docs/build/html/api_http/query/index.html
  //Metric Query String Format
  //The full specification for a metric query string sub query is as follows:
  //
  //m=<aggregator>:[rate[{counter[,<counter_max>[,<reset_value>]]]}:][<down_sampler>:][percentiles\[<p1>, <pn>\]:][explicit_tags:]
  // <metric_name>[{<tag_name1>=<grouping filter>[,...<tag_nameN>=<grouping_filter>]}]  -----参与分组的
  // [{<tag_name1>=<non grouping filter>[,...<tag_nameN>=<non_grouping_filter>]}] ----不参与分组的

  final int len = metric.length();
  if (metric.charAt(len - 1) != '}') {  // "foo{"
    throw new IllegalArgumentException("Missing '}' at the end of: " + metric);
  } else if (curly == len - 2) {  // "foo{}"
    //只有度量
    return metric.substring(0, len - 2);
  }
  //注意这里只处理了最后{}，类似于这种m=sum:temperature{city=*,zip_code=*,latitude=*}{city=not_literal_or(shanghai)}{city=literal_or(guangzhou)}{city=literal_or(beijing)}
  //有很多{}的是不会处理的，也就是说，opentsdb只处理{}一种或者{}{}两种情况，也就是说只有两个子过滤器？
  //代码只处理lastIndexOf和indexOf的情况，不处理中间的情况
  final int close = metric.indexOf('}');
  final HashMap<String, String> filter_map = new HashMap<String, String>();
  if (close != metric.length() - 1) { // "foo{...}{tagk=filter}" 
    final int filter_bracket = metric.lastIndexOf('{');
    for (final String filter : splitString(metric.substring(filter_bracket + 1, 
        metric.length() - 1), ',')) {
      if (filter.isEmpty()) {
        break;
      }
      filter_map.clear();
      try {
        //首先处理后面的不参与分组的情况，（因为上面代码是lastIndexOf {,所以就是规格说明里面的最后的不参与分组的过滤器的解析
        parse(filter_map, filter);
        TagVFilter.mapToFilters(filter_map, filters, false);
      } catch (IllegalArgumentException e) {
        throw new IllegalArgumentException("When parsing filter '" + filter
            + "': " + e.getMessage(), e);
      }
    }
  }
  
  // substring the tags out of "foo{a=b,...,x=y}" and parse them.
  //这里处理的是参与分组的那个，因为这里使用的close=indexOf({)，是第一个{出现的位置
  for (final String tag : splitString(metric.substring(curly + 1, close), ',')) {
    try {
      if (tag.isEmpty() && close != metric.length() - 1){
        break;
      }
      filter_map.clear();
      parse(filter_map, tag);
      TagVFilter.tagsToFilters(filter_map, filters);
    } catch (IllegalArgumentException e) {
      throw new IllegalArgumentException("When parsing tag '" + tag
                                         + "': " + e.getMessage(), e);
    }
  }
  // Return the "foo" part of "foo{a=b,...,x=y}"
  //返回的是度量
  return metric.substring(0, curly);
}
```


辅助函数：只有最后一个参数不同，group_by为true表示参与汇总，false表示不参与汇总

```

public static void tagsToFilters(final Map<String, String> tags, 
    final List<TagVFilter> filters) {
  mapToFilters(tags, filters, true);
}

/**
 * Converts the  map to a filter list. If a filter already exists for a
 * tag group by and we're told to process group bys, then the duplicate 
 * is skipped. 
 * @param map A set of tag keys and values. May be null or empty.
 * @param filters A set of filters to add the converted filters to. This may
 * not be null.
 * @param group_by Whether or not to set the group by flag and kick dupes
 */
public static void mapToFilters(final Map<String, String> map, 
    final List<TagVFilter> filters, final boolean group_by) {
  if (map == null || map.isEmpty()) {
    return;
  }

  for (final Map.Entry<String, String> entry : map.entrySet()) {
    TagVFilter filter = getFilter(entry.getKey(), entry.getValue());

    if (filter == null && entry.getValue().equals("*")) {
      filter = new TagVWildcardFilter(entry.getKey(), "*", true);
    } else if (filter == null) {
      //tagk=v的情况，其实就是literal_or的情况，所以这里设置的是TagVLiteralOrFilter
      filter = new TagVLiteralOrFilter(entry.getKey(), entry.getValue());
    }
    
    if (group_by) {
      filter.setGroupBy(true);
      boolean duplicate = false;
      for (final TagVFilter existing : filters) {
        if (filter.equals(existing)) {
          LOG.debug("Skipping duplicate filter: " + existing);
          existing.setGroupBy(true);
          duplicate = true;
          break;
        }
      }
      
      if (!duplicate) {
        filters.add(filter);
      }
    } else {
      filters.add(filter);
    }
  }
}
```



#### 序列化后的校验和后处理

> net.opentsdb.core.TSSubQuery#validateAndSetQuery

aggregator反序列化后为字符串，需要转换为对象，downsample也是

```java
public void validateAndSetQuery() {
  if (aggregator == null || aggregator.isEmpty()) {
    throw new IllegalArgumentException("Missing the aggregation function");
  }
  try {
    agg = Aggregators.get(aggregator);
  } catch (NoSuchElementException nse) {
    throw new IllegalArgumentException(
        "No such aggregation function: " + aggregator);
  }
  
  // we must have at least one TSUID OR a metric
  if ((tsuids == null || tsuids.isEmpty()) && 
      (metric == null || metric.isEmpty())) {
    throw new IllegalArgumentException(
        "Missing the metric or tsuids, provide at least one");
  }
  
  // Make sure we have a filter list
  if (filters == null) {
    filters = new ArrayList<TagVFilter>();
  }

  // parse the downsampler if we have one
  if (downsample != null && !downsample.isEmpty()) {
    // downsampler given, so parse it
    downsample_specifier = new DownsamplingSpecification(downsample);
  } else {
    // no downsampler
    downsample_specifier = DownsamplingSpecification.NO_DOWNSAMPLER;
  }
}
```

# 5.Opentsdb 查询执行

查询的执行顺序：解析请求->异步执行->数据处理->格式化

## 1. 查询数据的处理顺序

- Filtering
- Grouping
- Downsampling
- Interpolation
- Aggregation
- Rate Conversion
- Functions
- Expressions

## 2. 针对异常的处理：

> net.opentsdb.tsd.QueryRpc#handleQuery

```
class ErrorCB implements Callback<Object, Exception> {
  public Object call(final Exception e) throws Exception {
    Throwable ex = e;
    try {
      LOG.error("Query exception: ", e);
      if (ex instanceof DeferredGroupException) {
        ex = e.getCause();
        while (ex != null && ex instanceof DeferredGroupException) {
          ex = ex.getCause();
        }
        if (ex == null) {
          LOG.error("The deferred group exception didn't have a cause???");
        }
      } 

      if (ex instanceof RpcTimedOutException) {
        query_stats.markSerialized(HttpResponseStatus.REQUEST_TIMEOUT, ex);
        query.badRequest(new BadRequestException(
            HttpResponseStatus.REQUEST_TIMEOUT, ex.getMessage()));
        query_exceptions.incrementAndGet();
      } else if (ex instanceof HBaseException) {
        query_stats.markSerialized(HttpResponseStatus.FAILED_DEPENDENCY, ex);
        query.badRequest(new BadRequestException(
            HttpResponseStatus.FAILED_DEPENDENCY, ex.getMessage()));
        query_exceptions.incrementAndGet();
      } else if (ex instanceof QueryException) {
        query_stats.markSerialized(((QueryException)ex).getStatus(), ex);
        query.badRequest(new BadRequestException(
            ((QueryException)ex).getStatus(), ex.getMessage()));
        query_exceptions.incrementAndGet();
      } else if (ex instanceof BadRequestException) {
        query_stats.markSerialized(((BadRequestException)ex).getStatus(), ex);
        query.badRequest((BadRequestException)ex);
        query_invalid.incrementAndGet();
      } else if (ex instanceof NoSuchUniqueName) {
        query_stats.markSerialized(HttpResponseStatus.BAD_REQUEST, ex);
        query.badRequest(new BadRequestException(ex));
        query_invalid.incrementAndGet();
      } else {
        query_stats.markSerialized(HttpResponseStatus.INTERNAL_SERVER_ERROR, ex);
        query.badRequest(new BadRequestException(ex));
        query_exceptions.incrementAndGet();
      }
      
    } catch (RuntimeException ex2) {
      LOG.error("Exception thrown during exception handling", ex2);
      query_stats.markSerialized(HttpResponseStatus.INTERNAL_SERVER_ERROR, ex2);
      query.sendReply(HttpResponseStatus.INTERNAL_SERVER_ERROR, 
          ex2.getMessage().getBytes());
      query_exceptions.incrementAndGet();
    }
    return null;
  }
}
```

## 3.数据结构以及数据组织：

#### DataPoint

```
/**
 * Represents a single data point.
 * <p>
 * Implementations of this interface aren't expected to be synchronized.
 */
public interface DataPoint {

  /**
   * Returns the timestamp (in milliseconds) associated with this data point.
   * @return A strictly positive, 32 bit integer.
   */
  long timestamp();

  /**
   * Tells whether or not the this data point is a value of integer type.
   * @return {@code true} if the {@code i}th value is of integer type,
   * {@code false} if it's of doubleing point type.
   */
  boolean isInteger();

  /**
   * Returns the value of the this data point as a {@code long}.
   * @throws ClassCastException if the {@code isInteger() == false}.
   */
  long longValue();

  /**
   * Returns the value of the this data point as a {@code double}.
   * @throws ClassCastException if the {@code isInteger() == true}.
   */
  double doubleValue();

  /**
   * Returns the value of the this data point as a {@code double}, even if
   * it's a {@code long}.
   * @return When {@code isInteger() == false}, this method returns the same
   * thing as {@link #doubleValue}.  Otherwise, it returns the same thing as
   * {@link #longValue}'s return value casted to a {@code double}.
   */
  double toDouble();

}

```

*注意这只有时间戳字段和值字段，和我们通常了解的时间点是不一样的，我们理解的时间点是包含了度量和时间线的，这里统统没有。*

#### DataPoints

```java
/**
 * Represents a read-only sequence of continuous data points.
 * <p>
 * Implementations of this interface aren't expected to be synchronized.
 */
public interface DataPoints extends Iterable<DataPoint> {

  /**
   * Returns the name of the series.
   */
  String metricName();
  
  /**
   * Returns the name of the series.
   * @since 1.2
   */
  Deferred<String> metricNameAsync();
  
  /**
   * @return the metric UID
   * @since 2.3
   */
  byte[] metricUID();

  /**
   * Returns the tags associated with these data points.
   * @return A non-{@code null} map of tag names (keys), tag values (values).
   */
  Map<String, String> getTags();
  
  /**
   * Returns the tags associated with these data points.
   * @return A non-{@code null} map of tag names (keys), tag values (values).
   * @since 1.2
   */
  Deferred<Map<String, String>> getTagsAsync();
  
  /**
   * Returns a map of tag pairs as UIDs.
   * When used on a span or row, it returns the tag set. When used on a span 
   * group it will return only the tag pairs that are common across all 
   * time series in the group.
   * @return A potentially empty map of tagk to tagv pairs as UIDs
   * @since 2.2
   */
  ByteMap<byte[]> getTagUids();

  /**
   * Returns the tags associated with some but not all of the data points.
   * <p>
   * When this instance represents the aggregation of multiple time series
   * (same metric but different tags), {@link #getTags} returns the tags that
   * are common to all data points (intersection set) whereas this method
   * returns all the tags names that are not common to all data points (union
   * set minus the intersection set, also called the symmetric difference).
   * <p>
   * If this instance does not represent an aggregation of multiple time
   * series, the list returned is empty.
   * @return A non-{@code null} list of tag names.
   */
  List<String> getAggregatedTags();
  
  /**
   * Returns the tags associated with some but not all of the data points.
   * <p>
   * When this instance represents the aggregation of multiple time series
   * (same metric but different tags), {@link #getTags} returns the tags that
   * are common to all data points (intersection set) whereas this method
   * returns all the tags names that are not common to all data points (union
   * set minus the intersection set, also called the symmetric difference).
   * <p>
   * If this instance does not represent an aggregation of multiple time
   * series, the list returned is empty.
   * @return A non-{@code null} list of tag names.
   * @since 1.2
   */
  Deferred<List<String>> getAggregatedTagsAsync();

  /**
   * Returns the tagk UIDs associated with some but not all of the data points. 
   * @return a non-{@code null} list of tagk UIDs.
   * @since 2.3
   */
  List<byte[]> getAggregatedTagUids();
  
  /**
   * Returns a list of unique TSUIDs contained in the results
   * @return an empty list if there were no results, otherwise a list of TSUIDs
   */
  public List<String> getTSUIDs();
  
  /**
   * Compiles the annotations for each span into a new array list
   * @return Null if none of the spans had any annotations, a list if one or
   * more were found
   */
  public List<Annotation> getAnnotations();
  
  /**
   * Returns the number of data points.
   * <p>
   * This method must be implemented in {@code O(1)} or {@code O(n)}
   * where <code>n = {@link #aggregatedSize} &gt; 0</code>.
   * @return A positive integer.
   */
  int size();

  /**
   * Returns the number of data points aggregated in this instance.
   * <p>
   * When this instance represents the aggregation of multiple time series
   * (same metric but different tags), {@link #size} returns the number of data
   * points after aggregation, whereas this method returns the number of data
   * points before aggregation.
   * <p>
   * If this instance does not represent an aggregation of multiple time
   * series, then 0 is returned.
   * @return A positive integer.
   */
  int aggregatedSize();

  /**
   * Returns a <em>zero-copy view</em> to go through {@code size()} data points.
   * <p>
   * The iterator returned must return each {@link DataPoint} in {@code O(1)}.
   * <b>The {@link DataPoint} returned must not be stored</b> and gets
   * invalidated as soon as {@code next} is called on the iterator.  If you
   * want to store individual data points, you need to copy the timestamp
   * and value out of each {@link DataPoint} into your own data structures.
   */
  SeekableView iterator();

  /**
   * Returns the timestamp associated with the {@code i}th data point.
   * The first data point has index 0.
   * <p>
   * This method must be implemented in
   * <code>O({@link #aggregatedSize})</code> or better.
   * <p>
   * It is guaranteed that <pre>timestamp(i) &lt; timestamp(i+1)</pre>
   * @return A strictly positive integer.
   * @throws IndexOutOfBoundsException if {@code i} is not in the range
   * <code>[0, {@link #size} - 1]</code>
   */
  long timestamp(int i);

  /**
   * Tells whether or not the {@code i}th value is of integer type.
   * The first data point has index 0.
   * <p>
   * This method must be implemented in
   * <code>O({@link #aggregatedSize})</code> or better.
   * @return {@code true} if the {@code i}th value is of integer type,
   * {@code false} if it's of floating point type.
   * @throws IndexOutOfBoundsException if {@code i} is not in the range
   * <code>[0, {@link #size} - 1]</code>
   */
  boolean isInteger(int i);

  /**
   * Returns the value of the {@code i}th data point as a long.
   * The first data point has index 0.
   * <p>
   * This method must be implemented in
   * <code>O({@link #aggregatedSize})</code> or better.
   * Use {@link #iterator} to get successive {@code O(1)} accesses.
   * @see #iterator
   * @throws IndexOutOfBoundsException if {@code i} is not in the range
   * <code>[0, {@link #size} - 1]</code>
   * @throws ClassCastException if the
   * <code>{@link #isInteger isInteger(i)} == false</code>.
   */
  long longValue(int i);

  /**
   * Returns the value of the {@code i}th data point as a float.
   * The first data point has index 0.
   * <p>
   * This method must be implemented in
   * <code>O({@link #aggregatedSize})</code> or better.
   * Use {@link #iterator} to get successive {@code O(1)} accesses.
   * @see #iterator
   * @throws IndexOutOfBoundsException if {@code i} is not in the range
   * <code>[0, {@link #size} - 1]</code>
   * @throws ClassCastException if the
   * <code>{@link #isInteger isInteger(i)} == true</code>.
   */
  double doubleValue(int i);

  /**
   * Return the query index that maps this datapoints to the original TSSubQuery.
   * @return index of the query in the TSQuery class
   * @throws UnsupportedOperationException if the implementing class can't map
   * to a sub query.
   * @since 2.2
   */
  int getQueryIndex();
}
```



这个接口才和metrics关联起来，metricName，getTags，getAggregatedTags，timestamp(i)和longValue(i）等,这里获取值和获取时间戳都是带了下标的，由此可见，这个类代表了一组的数据。（）

重点关注一下iterator这个函数，返回的是SeekableView，这是一个迭代器。通过它，实现了ZeroCopy数据的目的。



#### SeekableView

```
/**
 * Provides a <em>zero-copy view</em> to iterate through data points.
 * <p>
 * The iterator returned by classes that implement this interface must return
 * each {@link DataPoint} in {@code O(1)} and does not support {@link #remove}.
 * <p>
 * Because no data is copied during iteration and no new object gets created,
 * <b>the {@link DataPoint} returned must not be stored</b> and gets
 * invalidated as soon as {@link #next} is called on the iterator (actually it
 * doesn't get invalidated but rather its contents changes).  If you want to
 * store individual data points, you need to copy the timestamp and value out
 * of each {@link DataPoint} into your own data structures.
 * <p>
 * In the vast majority of cases, the iterator will be used to go once through
 * all the data points, which is why it's not a problem if the iterator acts
 * just as a transient "view".  Iterating will be very cheap since no memory
 * allocation is required (except to instantiate the actual iterator at the
 * beginning).
 */
public interface SeekableView extends Iterator<DataPoint> {

  /**
   * Returns {@code true} if this view has more elements.
   */
  boolean hasNext();

  /**
   * Returns a <em>view</em> on the next data point.
   * No new object gets created, the referenced returned is always the same
   * and must not be stored since its internal data structure will change the
   * next time {@code next()} is called.
   * @throws NoSuchElementException if there were no more elements to iterate
   * on (in which case {@link #hasNext} would have returned {@code false}.
   */
  DataPoint next();

  /**
   * Unsupported operation.
   * @throws UnsupportedOperationException always.
   */
  void remove();

  /**
   * Advances the iterator to the given point in time.
   * <p>
   * This allows the iterator to skip all the data points that are strictly
   * before the given timestamp.
   * @param timestamp A strictly positive 32 bit UNIX timestamp (in seconds).
   * @throws IllegalArgumentException if the timestamp is zero, or negative,
   * or doesn't fit on 32 bits (think "unsigned int" -- yay Java!).
   */
  void seek(long timestamp);

}

```



#### RowSeq

实现了DataPoints接口，3.0是直接实现了DataPoints接口；

```
public final class RowSeq implements iRowSeq {
}
public interface iRowSeq extends DataPoints {
}
```

##### 比较器RowSeqComparator：

RowSeq代表的是规整到一个小时的数据，所以只要比较BaseTime（小时就够了），这里也隐含了一个条件，就是能够进行比较的RowSeq一定是其他维度相同的！这个在后面的Span实现是挂钩的，因为Span就是相同的时间线来排序的，同一个Span里面，放的是不同的RowSeq，ROwSeq里面放的是相同的RowKey里面按照Qualifer排序的数据！

这样数据就排序起来了！

```
public static final class RowSeqComparator implements Comparator<iRowSeq> {
  public int compare(final iRowSeq a, final iRowSeq b) {
    if (a.baseTime() == b.baseTime()) {
      return 0;
    }
    return a.baseTime() < b.baseTime() ? -1 : 1;
  }
}

```

- ##### 第一层汇聚



首先，有三个成员变量，key，qualifiers，values，有点类似Hbase的一个KeyValue了。

```
/**
 * Represents a read-only sequence of continuous HBase rows.
 * <p>
 * This class stores in memory the data of one or more continuous
 * HBase rows for a given time series. To consolidate memory, the data points
 * are stored in two byte arrays: one for the time offsets/flags and another
 * for the values. Access is granted via pointers.
 */

3.0直接实现了DataPoints接口，2.4RC2实现了iRowSeq，而iRowSeq扩展了DataPoints接口
final class RowSeq implements DataPoints {

  /** The {@link TSDB} instance we belong to. */
  private final TSDB tsdb;

  /** First row key. */
  byte[] key;

  /**
   * Qualifiers for individual data points.
   * <p>
   * Each qualifier is on 2 or 4 bytes.  The last {@link Const#FLAG_BITS} bits 
   * are used to store flags (the type of the data point - integer or floating
   * point - and the size of the data point in bytes).  The remaining MSBs
   * store a delta in seconds from the base timestamp stored in the row key.
   */
  private byte[] qualifiers;

  /** Values in the row.  */
  private byte[] values;

  /**
   * Constructor.
   * @param tsdb The TSDB we belong to.
   */
  RowSeq(final TSDB tsdb) {
    this.tsdb = tsdb;
  }

```



使用时，在new了对象之后，一定要首先调用setRow函数。KeyValue实际上是Hbase 的一个Cell。这里会把上面的三个成员变量初始化。

```java
/**
 * Sets the row this instance holds in RAM using a row from a scanner.
 * @param row The compacted HBase row to set.
 * @throws IllegalStateException if this method was already called.
 */
void setRow(final KeyValue row) {
  if (this.key != null) {
    throw new IllegalStateException("setRow was already called on " + this);
  }

  this.key = row.key();
  this.qualifiers = row.qualifier();
  this.values = row.value();
}
```

这里面比较重要的一个函数是AddRow：

看他的注释：Merges data points for the same HBase row into the local object。主要的作用还是开启Salt的时候，合并数据用。主要是修改一下两个成员变量：qualifiers和values；

1. 挨个比较已有的qualifiers和新加入的qualifiers，按照大小顺序，把数据合并到qualifiers和values
2. 有重复的qualifier，丢弃掉。
3. 参照qualifiers操作小节，是按照offset排序的，就是说按照时间戳的先后排序的。
4. 这里其实应该注意一下写的时候如果时间线相同，唯一不同的是值，看看哪个值生效。（从代码看，看哪个数据先到达，被选取出来，先被选出来的，就生效，有很大的随机性）。



***注意：这里其实是数据汇总的第一层！***，就是把相同的rowkey的不同Qualifer的数据先弄到一起！

注意合并之后，最后面长度延长了1，专门用来保存标志位，标记是否合并后的数据；这个标志位，就是仅仅用来区分qualifer是秒（2）还是毫秒（4字节）还是混合型的！从size函数的实现可以看出来！



##### RowSeq疑问：

读取qualifer和value都是直接读取的字节值，没有判断是不是Compact的。合并之后的也是合并了很多的字节，在这里是不断的增加indx指针来实现对compact的数据的读取的！



这里看不出合并后的数据怎么读取的，需要结合compact的代码！

```
/**
 * Merges data points for the same HBase row into the local object.
 * When executing multiple async queries simultaneously, they may call into 
 * this method with data sets that are out of order. This may ONLY be called 
 * after setRow() has initiated the rowseq. It also allows for rows with 
 * different salt bucket IDs to be merged into the same sequence.
 * @param row The compacted HBase row to merge into this instance.
 * @throws IllegalStateException if {@link #setRow} wasn't called first.
 * @throws IllegalArgumentException if the data points in the argument
 * do not belong to the same row as this RowSeq
 */
@Override
public void addRow(final KeyValue row) {
  if (this.key == null) {
    throw new IllegalStateException("setRow was never called on " + this);
  }

  final byte[] key = row.key();
  if (Bytes.memcmp(this.key, key, Const.SALT_WIDTH(), 
      key.length - Const.SALT_WIDTH()) != 0) {
    throw new IllegalDataException("Attempt to add a different row="
        + row + ", this=" + this);
  }

  final byte[] remote_qual = row.qualifier();
  final byte[] remote_val = row.value();
  final byte[] merged_qualifiers = new byte[qualifiers.length + remote_qual.length];
  final byte[] merged_values = new byte[values.length + remote_val.length]; 

  int remote_q_index = 0;
  int local_q_index = 0;
  int merged_q_index = 0;
  
  int remote_v_index = 0;
  int local_v_index = 0;
  int merged_v_index = 0;
  short v_length;
  short q_length;
  while (remote_q_index < remote_qual.length || 
      local_q_index < qualifiers.length) {
    // if the remote q has finished, we just need to handle left over locals
    if (remote_q_index >= remote_qual.length) {
      v_length = Internal.getValueLengthFromQualifier(qualifiers, 
          local_q_index);
      System.arraycopy(values, local_v_index, merged_values, 
          merged_v_index, v_length);
      local_v_index += v_length;
      merged_v_index += v_length;
      
      q_length = Internal.getQualifierLength(qualifiers, 
          local_q_index);
      System.arraycopy(qualifiers, local_q_index, merged_qualifiers, 
          merged_q_index, q_length);
      local_q_index += q_length;
      merged_q_index += q_length;
      
      continue;
    }
    
    // if the local q has finished, we need to handle the left over remotes
    if (local_q_index >= qualifiers.length) {
      v_length = Internal.getValueLengthFromQualifier(remote_qual, 
          remote_q_index);
      System.arraycopy(remote_val, remote_v_index, merged_values, 
          merged_v_index, v_length);
      remote_v_index += v_length;
      merged_v_index += v_length;
      
      q_length = Internal.getQualifierLength(remote_qual, 
          remote_q_index);
      System.arraycopy(remote_qual, remote_q_index, merged_qualifiers, 
          merged_q_index, q_length);
      remote_q_index += q_length;
      merged_q_index += q_length;
      
      continue;
    }
    
    // for dupes, we just need to skip and continue
    // 有重复的
    final int sort = Internal.compareQualifiers(remote_qual, remote_q_index, 
        qualifiers, local_q_index);
    if (sort == 0) {
      //LOG.debug("Discarding duplicate timestamp: " + 
      //    Internal.getOffsetFromQualifier(remote_qual, remote_q_index));
      v_length = Internal.getValueLengthFromQualifier(remote_qual, 
          remote_q_index);
      remote_v_index += v_length;
      q_length = Internal.getQualifierLength(remote_qual, 
          remote_q_index);
      remote_q_index += q_length;
      continue;
    }
    
    if (sort < 0) {
      v_length = Internal.getValueLengthFromQualifier(remote_qual, 
          remote_q_index);
      System.arraycopy(remote_val, remote_v_index, merged_values, 
          merged_v_index, v_length);
      remote_v_index += v_length;
      merged_v_index += v_length;
      
      q_length = Internal.getQualifierLength(remote_qual, 
          remote_q_index);
      System.arraycopy(remote_qual, remote_q_index, merged_qualifiers, 
          merged_q_index, q_length);
      remote_q_index += q_length;
      merged_q_index += q_length;
    } else {
      v_length = Internal.getValueLengthFromQualifier(qualifiers, 
          local_q_index);
      System.arraycopy(values, local_v_index, merged_values, 
          merged_v_index, v_length);
      local_v_index += v_length;
      merged_v_index += v_length;
      
      q_length = Internal.getQualifierLength(qualifiers, 
          local_q_index);
      System.arraycopy(qualifiers, local_q_index, merged_qualifiers, 
          merged_q_index, q_length);
      local_q_index += q_length;
      merged_q_index += q_length;
    }
  }
  
  // we may have skipped some columns if we were given duplicates. Since we
  // had allocated enough bytes to hold the incoming row, we need to shrink
  // the final results
  if (merged_q_index == merged_qualifiers.length) {
    qualifiers = merged_qualifiers;
  } else {
    qualifiers = Arrays.copyOfRange(merged_qualifiers, 0, merged_q_index);
  }
  
  // set the meta bit based on the local and remote metas
  //这里这段话很重要，因为每个Cell有可能是秒的，也有可能是毫秒的，还有可能是合并以后的，所以就得标记
  //如果是合并以后的数据的读取，其实上面的遍历那么已经做了
  byte meta = 0;
  if ((values[values.length - 1] & Const.MS_MIXED_COMPACT) == 
                                   Const.MS_MIXED_COMPACT || 
      (remote_val[remote_val.length - 1] & Const.MS_MIXED_COMPACT) == 
                                           Const.MS_MIXED_COMPACT) {
    meta = Const.MS_MIXED_COMPACT;
  }
  //注意这里：有一个+1操作，专门用来存放标志位的。
  values = Arrays.copyOfRange(merged_values, 0, merged_v_index + 1);
  values[values.length - 1] = meta;
}

```

前面的SetRow和AddRow是RowSeq写的过程。

既然RowSeq实现了DataPoints接口，那么我们看看他是怎么遍历数据以及获取数据值的：



- 对应的度量和标签（此处仅仅列出度量）

RowSeq对应的是一个rowkey的整行数据了，所以，这里会对应一个metrics；注意，加载的时候tsdb会缓存这些信息的，缓存的地方是tsdb这个实例；

```
public String metricName() {
  try {
    return metricNameAsync().joinUninterruptibly();
  } catch (RuntimeException e) {
    throw e;
  } catch (Exception e) {
    throw new RuntimeException("Should never be here", e);
  }
}

```



- 获取大小；

  这个函数很重要，可以理解数据的组成。MIX类型的，需要遍历，秒或者毫秒类型的，常量时间；所以我们写数据的时候尽量以一种类型去写！

```
public int size() {
  // if we don't have a mix of second and millisecond qualifiers we can run
  // this in O(1), otherwise we have to run O(n)
  if ((values[values.length - 1] & Const.MS_MIXED_COMPACT) == 
    Const.MS_MIXED_COMPACT) {
    int size = 0;
    for (int i = 0; i < qualifiers.length; i += 2) {
      if ((qualifiers[i] & Const.MS_BYTE_FLAG) == Const.MS_BYTE_FLAG) {
        i += 2;
      }
      size++;
    }
    return size;
  } else if ((qualifiers[0] & Const.MS_BYTE_FLAG) == Const.MS_BYTE_FLAG) {
    return qualifiers.length / 4;
  } else {
    return qualifiers.length / 2;
  }
}

```



- 遍历数据：

根据qual_index和value_index作为指针，不断向后移动，同时解析数据。数据的长度存放在qualifier里，同时移动value_index

```
public boolean hasNext() {
  return qual_index < qualifiers.length;
}

public DataPoint next() {
  if (!hasNext()) {
    throw new NoSuchElementException("no more elements");
  }
  
  if (Internal.inMilliseconds(qualifiers[qual_index])) {
    qualifier = Bytes.getInt(qualifiers, qual_index);
    qual_index += 4;
  } else {
    qualifier = Bytes.getUnsignedShort(qualifiers, qual_index);
    qual_index += 2;
  }
  final byte flags = (byte) qualifier;
  value_index += (flags & Const.LENGTH_MASK) + 1;
  //LOG.debug("next -> now=" + toStringSummary());
  return this;
}
```



- Seek到指定的时间戳：和next是一样的逻辑，多了个循环；

```
public void seek(final long timestamp) {
  if ((timestamp & Const.MILLISECOND_MASK) != 0) {  // negative or not 48 bits
    throw new IllegalArgumentException("invalid timestamp: " + timestamp);
  }
  qual_index = 0;
  value_index = 0;
  final int len = qualifiers.length;
  //LOG.debug("Peeking timestamp: " + (peekNextTimestamp() < timestamp));
  while (qual_index < len && peekNextTimestamp() < timestamp) {
    //LOG.debug("Moving to next timestamp: " + peekNextTimestamp());
    if (Internal.inMilliseconds(qualifiers[qual_index])) {
      qualifier = Bytes.getInt(qualifiers, qual_index);
      qual_index += 4;
    } else {
      qualifier = Bytes.getUnsignedShort(qualifiers, qual_index);
      qual_index += 2;
    }
    final byte flags = (byte) qualifier;
    value_index += (flags & Const.LENGTH_MASK) + 1;
  }
  //LOG.debug("seek to " + timestamp + " -> now=" + toStringSummary());
}
```



- 当前数据的时间戳获取：

  注意这里会在Span里面调用到（当把一个Row存放到Span的时候）

```
public long timestamp() {
  assert qual_index > 0: "not initialized: " + this;
  if ((qualifier & Const.MS_FLAG) == Const.MS_FLAG) {
    final long ms = (qualifier & 0x0FFFFFC0) >>> (Const.MS_FLAG_BITS);
    return (base_time * 1000) + ms;            
  } else {
    final long seconds = (qualifier & 0xFFFF) >>> Const.FLAG_BITS;
    return (base_time + seconds) * 1000;
  }
}
```

- 获取值：

  注意：Long或者Float是与FLAG_FLOAT&，FLAG_FLOAT=0x8，就是第四个Bit表示Long或者Float；

```
public boolean isInteger() {
  assert qual_index > 0: "not initialized: " + this;
  return (qualifier & Const.FLAG_FLOAT) == 0x0;
}

public long longValue() {
  if (!isInteger()) {
    throw new ClassCastException("value @"
      + qual_index + " is not a long in " + this);
  }
  final byte flags = (byte) qualifier;
  final byte vlen = (byte) ((flags & Const.LENGTH_MASK) + 1);
  return extractIntegerValue(values, value_index - vlen, flags);
}

public double doubleValue() {
  if (isInteger()) {
    throw new ClassCastException("value @"
      + qual_index + " is not a float in " + this);
  }
  final byte flags = (byte) qualifier;
  final byte vlen = (byte) ((flags & Const.LENGTH_MASK) + 1);
  return extractFloatingPointValue(values, value_index - vlen, flags);
}
```

- getTagUids

  这是一个辅助函数，把所有的tag取出来

```
public static ByteMap<byte[]> getTagUids(final byte[] row) {
  final ByteMap<byte[]> uids = new ByteMap<byte[]>();
  final short name_width = TSDB.tagk_width();
  final short value_width = TSDB.tagv_width();
  final short tag_bytes = (short) (name_width + value_width);
  final short metric_ts_bytes = (short) (TSDB.metrics_width()
                                         + Const.TIMESTAMP_BYTES
                                         + Const.SALT_WIDTH());

  for (short pos = metric_ts_bytes; pos < row.length; pos += tag_bytes) {
    final byte[] tmp_name = new byte[name_width];
    final byte[] tmp_value = new byte[value_width];
    System.arraycopy(row, pos, tmp_name, 0, name_width);
    System.arraycopy(row, pos + name_width, tmp_value, 0, value_width);
    uids.put(tmp_name, tmp_value);
  }
  return uids;
}
```



#### Span

Span封装了RowSeq。参见成员变量rows

```
/**
 * Represents a read-only sequence of continuous data points.
 * <p>
 * This class stores a continuous sequence of {@link RowSeq}s in memory.
 */
public class Span implements DataPoints {

  private static final Logger LOG = LoggerFactory.getLogger(Span.class);

  /** The {@link TSDB} instance we belong to. */
  protected final TSDB tsdb;

  /** All the rows in this span. */
  protected final ArrayList<iRowSeq> rows = new ArrayList<iRowSeq>();
```

Span就是一系列的RowSeq，可以理解为多行数据。但是要求时间线必须相同。

##### SpanCmp

> net.opentsdb.core.TsdbQuery.SpanCmp

可以看出来，这就是在比较时间线

```
private static final class SpanCmp implements Comparator<byte[]> {

  private final short metric_width;

  public SpanCmp(final short metric_width) {
    this.metric_width = metric_width;
  }

  @Override
  public int compare(final byte[] a, final byte[] b) {
    final int length = Math.min(a.length, b.length);
    if (a == b) {  // Do this after accessing a.length and b.length
      return 0;    // in order to NPE if either a or b is null.
    }
    int i;
    // First compare the metric ID.
    for (i = 0; i < metric_width; i++) {
      if (a[i] != b[i]) {
        return (a[i] & 0xFF) - (b[i] & 0xFF);  // "promote" to unsigned.
      }
    }
    // Then skip the timestamp and compare the rest.
    for (i += Const.TIMESTAMP_BYTES; i < length; i++) {
      if (a[i] != b[i]) {
        return (a[i] & 0xFF) - (b[i] & 0xFF);  // "promote" to unsigned.
      }
    }
    return a.length - b.length;
  }

}
```

从下面的函数中可以看出：

- Span写：

```
protected void addRow(final KeyValue row) {
    long last_ts = 0;
    if (rows.size() != 0) {
      // Verify that we have the same metric id and tags.
      final byte[] key = row.key();
      final iRowSeq last = rows.get(rows.size() - 1);
      final short metric_width = tsdb.metrics.width();
      final short tags_offset = 
          (short) (Const.SALT_WIDTH() + metric_width + Const.TIMESTAMP_BYTES);
      final short tags_bytes = (short) (key.length - tags_offset);
      String error = null;
      if (key.length != last.key().length) {
        error = "row key length mismatch";
      } else if (
          Bytes.memcmp(key, last.key(), Const.SALT_WIDTH(), metric_width) != 0) {
        error = "metric ID mismatch";
      } else if (Bytes.memcmp(key, last.key(), tags_offset, tags_bytes) != 0) {
        error = "tags mismatch";
      }
      if (error != null) {
        throw new IllegalArgumentException(error + ". "
            + "This Span's last row key is " + Arrays.toString(last.key())
            + " whereas the row key being added is " + Arrays.toString(key)
            + " and metric_width=" + metric_width);
      }
      last_ts = last.timestamp(last.size() - 1);  // O(n)
    }

    final RowSeq rowseq = new RowSeq(tsdb);
    rowseq.setRow(row);
    sorted = false;
    LOG.info("last_ts={},rowseq.timestamp(0)={}",last_ts,rowseq.timestamp(0));
    if (last_ts >= rowseq.timestamp(0)) {
      // scan to see if we need to merge into an existing row
      for (final iRowSeq rs : rows) {
        if (Bytes.memcmp(rs.key(), row.key(), Const.SALT_WIDTH(), 
            (rs.key().length - Const.SALT_WIDTH())) == 0) {
          rs.addRow(row);
          return;
        }
      }
    }

    LOG.info("Adding row to rows as new");
    rows.add(rowseq);
  }
```

首先看5-25行的判断逻辑，就是比较metric_width的数据，要求metrics必须相等。然后跳过TIMESTAMP_BYTES，比较剩余的tags_bytes，要求必须相等，这就意味着必须是相同的时间线数据。但是这个Span可以包含不同小时的数据。而RowSeq 只能包含相同小时的数据。

当Span为空时，直接插入到ArrayList（第45行），当下个数据到达时（33-39行），先取出已有数据的最后一条RowSeq对应的时间戳，这个时间戳是这条数据的最大的时间戳，如果这个时间戳>需要插入的数据的第一个时间戳，意味着需要插入的数据的小时时间靠前，必须寻找到合适的位置插入，注意，这个是个遍历。

在查找的时候比较的是整个RowKey，这意味着必须属于相同的小时，如果找到，把这个Cell加入到那个小时的RowSeq里面去，如果没有找到，就生成一条新的RowSeq加入到ArrayList里面去（45行）；

疑问：？

注意ArrayList里面没有对RowSeq排序，意味着上面的遍历是对的，也意味着取得数据越多，跨小时数据越多，这里遍历的越多；

添加一个新的RowSeq的两个条件：1.Span为空，2，不为空，但是这个小时的RowSeq没有。插入但是不排序！！！如果我们取一天的数据，也就是24个小时，所以这边遍历应该不会影响很大的性能。

写的时候不排序，那么我们看看读的时候会不会排序？



Span必须是相同时间线的，这里再次强调一次！

- Span读

这里对RowSeq进行了排序！

写的时候排序，肯定会效率低下，读的时候排序，只排序一次！

这里用到了RowSeqComparator！就是比较小时数，谁在前谁在后！

```
/**
 * Checks the sorted flag and sorts the rows if necessary. Should be called
 * by any iteration method.
 * Since 2.0
 */
private void checkRowOrder() {
  if (!sorted) {
    Collections.sort(rows, new RowSeq.RowSeqComparator());
    sorted = true;
  }
}
```

```
/** Package private iterator method to access it as a Span.Iterator. */
Iterator spanIterator() {
  if (!sorted) {
    Collections.sort(rows, new RowSeq.RowSeqComparator());
    sorted = true;
  }
  return new Iterator();
}
```

这个函数在net.opentsdb.core.AggregationIterator#create(java.util.List<net.opentsdb.core.Span>, long, long, net.opentsdb.core.Aggregator, net.opentsdb.core.Aggregators.Interpolation, net.opentsdb.core.Aggregator, long, boolean, net.opentsdb.core.RateOptions, net.opentsdb.core.FillPolicy)里面用到了。

说明这个是用来遍历这个Span的。



```
/** Iterator for {@link Span}s. */
final class Iterator implements SeekableView {

  /** Index of the {@link RowSeq} we're currently at, in {@code rows}. */
  private int row_index;

  /** Iterator on the current row. */
  private iRowSeq.Iterator current_row;

  Iterator() {
    current_row = rows.get(0).internalIterator();
  }

  // ------------------ //
  // Iterator interface //
  // ------------------ //
  
  @Override
  public boolean hasNext() {
    if (current_row.hasNext()) {
      return true;
    }
    // handle situations where a row in the middle may be empty due to some
    // kind of logic kicking out data points
    while (row_index < rows.size() - 1) {
      row_index++;
      current_row = rows.get(row_index).internalIterator();
      if (current_row.hasNext()) {
        return true;
      }
    }
    return false;
  }

  @Override
  public DataPoint next() {
    if (current_row.hasNext()) {
      return current_row.next();
    }
    // handle situations where a row in the middle may be empty due to some
    // kind of logic kicking out data points
    while (row_index < rows.size() - 1) {
      row_index++;
      current_row = rows.get(row_index).internalIterator();
      if (current_row.hasNext()) {
        return current_row.next();
      }
    }
    throw new NoSuchElementException("no more elements");
  }

  @Override
  public void remove() {
    throw new UnsupportedOperationException();
  }

  // ---------------------- //
  // SeekableView interface //
  // ---------------------- //
  
  @Override
  public void seek(final long timestamp) {
    int row_index = seekRow(timestamp);
    if (row_index != this.row_index) {
      this.row_index = row_index;
      current_row = rows.get(row_index).internalIterator();
    }
    current_row.seek(timestamp);
  }

  @Override
  public String toString() {
    return "Span.Iterator(row_index=" + row_index
      + ", current_row=" + current_row + ", span=" + Span.this + ')';
  }

}

private int seekRow(final long timestamp) {
    checkRowOrder();
    int row_index = 0;
    iRowSeq row = null;
    final int nrows = rows.size();
    for (int i = 0; i < nrows; i++) {
      row = rows.get(i);
      final int sz = row.size();
      if (sz < 1) {
        row_index++;
      } else if (row.timestamp(sz - 1) < timestamp) {
        row_index++;  // The last DP in this row is before 'timestamp'.
      } else {
        break;
      }
    }
    if (row_index == nrows) {  // If this timestamp was too large for the
      --row_index;             // last row, return the last row.
    }
    return row_index;
  }
```

上面的移动很简单，就是记住当前rowseq，如果当前row遍历完了，就移动到下一个rowseq。

- 降采样

注意，在Span这个类中，加入了降采样的处理，想想也是合理的，降采样就是要把数据合并和减少，合并肯定要在相同的维度合并，把时间力度扩大，这里岂不就是最好的地方？：）



>  *https://stackoverflow.com/questions/34648064/how-to-group-by-and-aggregate-in-opentsdb-like-rdbms* 
>
> *Q：How to group by and aggregate in openTSDB like RDBMS?*
>
> *A:   Now way for openTSDB to do it. Also if there is requirement like this, then openTSDB may be not your choice. openTSDB is time series db, also for kariosDB. I tried in openTSDB and kariosDB and found they both can not.*
>
> *Because in openTSDB , the group by is one thing, the aggregate is another thing. Not like RDBMS, the agg works on the group by. In openTSDB the agg works on the downsample*

```
**
 * Package private iterator method to access data while downsampling with the
 * option to force interpolation.
 * @param start_time The time in milliseconds at which the data begins.
 * @param end_time The time in milliseconds at which the data ends.
 * @param interval_ms The interval in milli seconds wanted between each data
 * point.
 * @param downsampler The downsampling function to use.
 * @param fill_policy Policy specifying whether to interpolate or to fill
 * missing intervals with special values.
 * @return A new downsampler.
 */
Downsampler downsampler(final long start_time,
                        final long end_time,
                        final long interval_ms,
                        final Aggregator downsampler,
                        final FillPolicy fill_policy) {
  if (FillPolicy.NONE == fill_policy) {
    // The default downsampler simply skips missing intervals, causing the
    // span group to linearly interpolate.
    return new Downsampler(spanIterator(), interval_ms, downsampler);
  } else {
    // Otherwise, we need to instantiate a downsampler that can fill missing
    // intervals with special values.
    return new FillingDownsampler(spanIterator(), start_time, end_time,
      interval_ms, downsampler, fill_policy);
  }
}
```

这里生成了一个Downsampler对象，最主要的是把spanIterator传过去了，这里要记住，降采样是对相同时间线不同小时的数据进行遍历进行插值计算而已！





- #### 辅助函数：getTagUids

获取tags，前面已经讲过，span都是同一个时间线的，所以只要获取第一个就够了！



```
@Override
public ByteMap<byte[]> getTagUids() {
  checkNotEmpty();
  return rows.get(0).getTagUids();
}
```

#### SpanGroup

Span把相同时间线的不同小时的数据，都组织好了，那么SpanGroup是干什么的呢？

SpanGroup代表了不同的时间线的数据！

主要的目的是针对查询提供的汇总维度进行数据组织！这里就减少了汇总的维度；Span可以理解为按照时间线通过降采样进行GroupBy统计，SpanGroup可以理解为减少时间线里面的维度，将数据按照减少后的维度进行GroupBy统计。



##### 疑问：

这里的spans是不是排序的还不知道！（看后面mergeDataPoints是不排序的）

```
SpanGroup(final TSDB tsdb,
          final long start_time, 
          final long end_time,
          final Iterable<Span> spans,
          final boolean rate, 
          final RateOptions rate_options,
          final Aggregator aggregator,
          final DownsamplingSpecification downsampler, 
          final long query_start,
          final long query_end,
          final int query_index,
          final RollupQuery rollup_query) {
   annotations = new ArrayList<Annotation>();
   this.start_time = (start_time & Const.SECOND_MASK) == 0 ? 
       start_time * 1000 : start_time;
   this.end_time = (end_time & Const.SECOND_MASK) == 0 ? 
       end_time * 1000 : end_time;
   if (spans != null) {
     for (final Span span : spans) {
       add(span);
     }
   }
   this.rate = rate;
   this.rate_options = rate_options;
   this.aggregator = aggregator;
   this.downsampler = downsampler;
   this.query_start = query_start;
   this.query_end = query_end;
   this.query_index = query_index;
   this.rollup_query = rollup_query;
   this.tsdb = tsdb;
}
```



- 写数据：



```
/**
 * Adds a span to this group, provided that it's in the right time range.
 * <b>Must not</b> be called once {@link #getTags} or
 * {@link #getAggregatedTags} has been called on this instance.
 * @param span The span to add to this group.  If none of the data points
 * fall within our time range, this method will silently ignore that span.
 */
void add(final Span span) {
  if (tags != null) {
    throw new AssertionError("The set of tags has already been computed"
                             + ", you can't add more Spans to " + this);
  }

  // normalize timestamps to milliseconds for proper comparison
  final long start = (start_time & Const.SECOND_MASK) == 0 ? 
      start_time * 1000 : start_time;
  final long end = (end_time & Const.SECOND_MASK) == 0 ? 
      end_time * 1000 : end_time;

  if (span.size() == 0) {
    // copy annotations that are in the time range
    for (Annotation annot : span.getAnnotations()) {
      long annot_start = annot.getStartTime();
      if ((annot_start & Const.SECOND_MASK) == 0) {
        annot_start *= 1000;
      }
      long annot_end = annot.getStartTime();
      if ((annot_end & Const.SECOND_MASK) == 0) {
        annot_end *= 1000;
      }
      if (annot_end >= start && annot_start <= end) {
        annotations.add(annot);
      }
    }
  } else {
    long first_dp = span.timestamp(0);
    if ((first_dp & Const.SECOND_MASK) == 0) {
      first_dp *= 1000;
    }
    // The following call to timestamp() will throw an
    // IndexOutOfBoundsException if size == 0, which is OK since it would
    // be a programming error.
    long last_dp = span.timestamp(span.size() - 1);
    if ((last_dp & Const.SECOND_MASK) == 0) {
      last_dp *= 1000;
    }
    if (first_dp <= end && last_dp >= start) {
      this.spans.add(span);
      annotations.addAll(span.getAnnotations());
    }
  }
}
```

注意上面的add函数，里面，有一个关于时间的判断逻辑：

first_dp是这个Span里面时间最小的一个时间点，last_dp是这个Span里面最大的一个时间点。

如果这个Span的开始结束时间与查询的开始结束时间有交集，就把这个Span加入到SpanGroup里面去。

如果Span的最大时间都小于开始时间（ ！ last_dp >= start）或者最小时间都大于开始时间（！first_dp <= end ），那么很显然，这里的数据是不需要的！

***<u>==这里实现了第一层的数据过滤！==</u>***





##### computeTags



```
/**
 * Computes the intersection set + symmetric difference of tags in all spans.
 * This method loads the UID aggregated list and tag pair maps with byte arrays
 * but does not actually resolve the UIDs to strings. 
 * On the first run, it will initialize the UID collections (which may be empty)
 * and subsequent calls will skip processing.
 */
private void computeTags() {
  if (tag_uids != null && aggregated_tag_uids != null) {
    return;
  }
  if (spans.isEmpty()) {
    tag_uids = new ByteMap<byte[]>();
    aggregated_tag_uids = new HashSet<byte[]>();
    return;
  }
  
  // local tag uids
  final ByteMap<byte[]> tag_set = new ByteMap<byte[]>();
  
  // value is always null, we just want the set of unique keys
  final ByteMap<byte[]> discards = new ByteMap<byte[]>();
  final Iterator<Span> it = spans.iterator();
  while (it.hasNext()) {
    final Span span = it.next();
    final ByteMap<byte[]> uids = span.getTagUids();
    
    for (final Map.Entry<byte[], byte[]> tag_pair : uids.entrySet()) {
      // we already know it's an aggregated tag
      if (discards.containsKey(tag_pair.getKey())) {
        continue;
      }
      
      final byte[] tag_value = tag_set.get(tag_pair.getKey());
      if (tag_value == null) {
        tag_set.put(tag_pair.getKey(), tag_pair.getValue());
      } else if (Bytes.memcmp(tag_value, tag_pair.getValue()) != 0) {
        // bump to aggregated tags
        discards.put(tag_pair.getKey(), null);
        tag_set.remove(tag_pair.getKey());
      }
    }
  }
  
  aggregated_tag_uids = discards.keySet();
  tag_uids = tag_set;
}

```

这里遍历所有的Span的RowKey取出来的tags，如果出现了tagk相同，但是tagv不相同的，那么就把这些tagk认为是被聚合掉的，这里正好印证了SpanGroup的作用：就是查询里需要汇总的相同维度的Span会放到一个SpanGroup里面去！不同的维度当然就是被汇总掉的维度，哈哈。正好也验证了百度的数据，city和zip_code是一一对应的，但是如果我们只按照city汇总，但是zip_code也会出现在汇总维度里面！因为他都是一样的。

这样如果一个tagk出现了多个值，一定是属于被聚合掉的！



##### 读取数据：

```java
public SeekableView iterator() {
  return AggregationIterator.create(spans, start_time, end_time, aggregator,
                                aggregator.interpolationMethod(),
                                downsampler, query_start, query_end,
                                rate, rate_options, rollup_query);
}
```



```
public static AggregationIterator create(final List<Span> spans,
    final long start_time,
    final long end_time,
    final Aggregator aggregator,
    final Interpolation method,
    final DownsamplingSpecification downsampler,
    final long query_start,
    final long query_end,
    final boolean rate,
    final RateOptions rate_options,
    final RollupQuery rollup_query) {
  final int size = spans.size();
  final SeekableView[] iterators = new SeekableView[size];
  for (int i = 0; i < size; i++) {
    SeekableView it;
    if (downsampler == null || 
        downsampler == DownsamplingSpecification.NO_DOWNSAMPLER) {
      it = spans.get(i).spanIterator();
    } else {
      it = spans.get(i).downsampler(start_time, end_time, downsampler, 
          query_start, query_end, rollup_query);
    }
    if (rate) {
      it = new RateSpan(it, rate_options);
    }
    iterators[i] = it;
  }
  return new AggregationIterator(iterators, start_time, end_time, aggregator,
      method, rate);  
}
```

上面代码有个逻辑：如果不需要降采样，很显然，我们就应该去遍历每一个Span，然后Span在遍历每个RowSeq，所以地17行就是这么处理的，直接返回spanIterator();如果需要降采样，很显然，应该是把Span里面的数据按照interval进行汇总之后，再把数据返回来；注意，这里是根据Span进行降采样，很显然，还没有进行查询里面要求的汇总。

##### AggregationIterator

```
/**
 * Iterator that aggregates multiple spans or time series data and does linear
 * interpolation (lerp) for missing data points.
 聚合多个span或者时间线数据的迭代器，对缺失的数据进行插值操作。
 * <p>
 * This where the real business of {@link SpanGroup} is.  This iterator
 * provides a merged, aggregated view of multiple {@link Span}s.  The data
 * points in all the Spans are returned in chronological order.  Each time
 * we return a data point from a span, we aggregate it with the current
 * value from all the other Spans.  If other Spans don't have a value at
 * that specific timestamp, we do a linear interpolation in order to
 * estimate what the value of that Span should be at that time.
 这个迭代器为多个spans提供了一个合并、聚合的视图；所有的span里面的数据是按照时间排序的(chronological：按时间的前后顺序排列的; 编年的;)。当我们从span返回一个数据点时
 * <p>
 * All this merging, linear interpolation and aggregation happens in
 * {@code O(1)} space and {@code O(N)} time.  All we need is to keep an
 * iterator on each Span, and {@code 4*k} {@code long}s in memory, where
 * {@code k} is the number of Spans in the group.  When computing a rate,
 * we need an extra {@code 2*k} {@code long}s in memory (see below).
 * <p>
 * In order to do linear interpolation, we need to know two data points:
 * the current one and the next one.  So for each Span in the group, we need
 * 4 longs: the current value, the current timestamp, the next value and the
 * next timestamp.  We maintain two arrays for timestamps and values.  Those
 * arrays have {@code 2 * iterators.length} elements.  The first half
 * contains the current values and second half the next values.  When a Span
 * gets used, its next data point becomes the current one (so its value and
 * timestamp are moved from the 2nd half of their respective array to the
 * first half) and the new-next data point is fetched from the underlying
 * iterator of that Span.
 * <p>
 * Here is an example when the SpanGroup contains 2 Spans:
 * <pre>              current    |     next
 *               +-------+-------+-------+-------+
 *   timestamps: |  T1   |  T2   |  T3   |  T4   |
 *               +-------+-------+-------+-------+
 *                    current    |     next
 *               +-------+-------+-------+-------+
 *       values: |  V1   |  V2   |  V3   |  V4   |
 *               +-------+-------+-------+-------+
 *                               |
 *   current: 0
 *   pos: 0
 *   iterators: [ it0, it1 ]
 * </pre>
 * Since {@code current == 0}, the current data point has the value V1
 * and time T1.  Let's note that (V1, T1).  Now this group has 2 Spans,
 * which means we're trying to aggregate 2 different series (same metric ID
 * but different tags).  So The next value that this iterator returns needs
 * to be a combination of V1 and V2 (assuming that T2 is less than T1).
 * If our aggregation function is "sum", we sort of want to sum up V1 and
 * V2.  But those two data points may not necessarily be at the same time.
 * T2 can be less than or equal to T1.  If T2 is greater than T1, we ignore
 * V2 and return just V1, since we haven't reached the time yet where V2
 * exist, so it's essentially as if it wasn't there.
 * Say T2 is less than T1.  Summing up V1 and V2 doesn't make sense, since
 * they represent two measurements made at different times.  So instead,
 * we need to find what the value V2 would have been, had it been measured
 * at time T1 instead of T2.  We do this using linear interpolation between
 * the data point (V2, T2) and the following one for that series, (V4, T4).
 * The result is thus the sum of V1 and the interpolated value between V2
 * and V4.
 * <p>
 * Now let's move onto the next data point.  Assuming that T3 is less than
 * T4, it means we need to advance to the next point on the 1st series.  To
 * do this we use the iterator it0 to get the next data point for that
 * series and we end up with the following state:
 * <pre>              current    |     next
 *               +-------+-------+-------+-------+
 *   timestamps: |  T3   |  T2   |  T5   |  T4   |
 *               +-------+-------+-------+-------+
 *                    current    |     next
 *               +-------+-------+-------+-------+
 *       values: |  V3   |  V2   |  V5   |  V4   |
 *               +-------+-------+-------+-------+
 *                               |
 *   current: 0
 *   pos: 0
 *   iterators: [ it0, it1 ]
 * </pre>
 * Then all you need is to "rinse(冲洗，漂洗) and repeat".
 * <p>
 * More details: Since each value above can be either an integer or a
 * floating point, we have to keep track of the type of each value.  Values
 * are always stored in a {@code long}.  When a value is a floating point
 * value, the bits of the longs just need to be interpreted to get back the
 * floating point value.  The way we keep track of the type is by using the
 * most significant bit of the timestamp (to avoid an extra array).  This is
 * fine since our timestamps only really use 32 of the 64 bits of the long
 * in which they're stored.  When there is no "current" value (1st half of
 * the arrays depicted(描绘) above), the timestamp will be set to 0.  When there
 * is no "next" value (2nd half of the arrays), the timestamp will be set
 * to a special, really large value (too large to be a valid timestamp).
 * <p>
 */
```



上述代码中，先把每个Span的逻辑iterator计算出来，然后把所有的iterator汇总起来，所以叫AggregationIterator！

看他的注释也是这个意思：

> *Creates an aggregation iterator for a group of data point iterators.*

那么我们就看看他的构造函数和next是怎么处理的：

这里缩小了时间范围，精确的匹配了查询的时间对应的数据点！这里的算法比较复杂，稍后分析。

16-17行：首先定位到开始的时间戳，这里只是调用了这个函数，但是不一定能够找到合适的时间戳，因为span的seek再找不到的时候会把指针知道最后一个row

29行：找到合适的数据，保存到对应的数组中。

```
public AggregationIterator(final SeekableView[] iterators,
                            final long start_time,
                            final long end_time,
                            final Aggregator aggregator,
                            final Interpolation method,
                            final boolean rate) {
  .....成员变量赋值......
  final int size = iterators.length;
  timestamps = new long[size * 2];
  values = new long[size * 2];
  // Initialize every Iterator, fetch their first values that fall
  // within our time range.
  int num_empty_spans = 0;
  for (int i = 0; i < size; i++) {
    SeekableView it = iterators[i];
    //这里不一定就是合适的值：
    //如果span里面找不到就会返回最后一个row的！所以才会有后面的时间戳的判断！
    it.seek(start_time);
    DataPoint dp;
    if (!it.hasNext()) {
      ++num_empty_spans;
      endReached(i);
      continue;
    }
    dp = it.next();
 
    if (dp.timestamp() >= start_time) {
      //如果数据点匹配查询要求的时间点，保存起来
      putDataPoint(size + i, dp);
    } else {
      //这里的条件是什么？
      // if there is data, advance to the start time if applicable.
      while (dp != null && dp.timestamp() < start_time) {
        if (it.hasNext()) {
          dp = it.next();
        } else {
          dp = null;
        }
      }
      if (dp == null) {
        if (LOG.isDebugEnabled()) {
          LOG.debug(String.format("No DP in range for #%d: start time %d", i,
                                  start_time));
        }
        endReached(i);
        continue;
      }
      putDataPoint(size + i, dp);
    }
    if (rate) {
      // The first rate against the time zero should be populated
      // for the backward compatibility that uses the previous rate
      // instead of interpolating for aggregation when a data point is
      // missing for the current timestamp.
      // TODO: Use the next rate that contains the current timestamp.
      if (it.hasNext()) {
        moveToNext(i);
      } else {
        endReached(i);
      }
    }
  }
  if (num_empty_spans > 0) {
    LOG.debug(String.format("%d out of %d spans are empty!",
                            num_empty_spans, size));
  }
}
```

hasNext：查看是否有元素满足时间条件

```
public boolean hasNext() {
  final int size = iterators.length;
  for (int i = 0; i < size; i++) {
    // As long as any of the iterators has a data point with a timestamp
    // that falls within our interval, we know we have at least one next.
    if ((timestamps[size + i] & TIME_MASK) <= end_time) {
      //LOG.debug("hasNext #" + (size + i));
      return true;
    }
  }
  //LOG.debug("No hasNext (return false)");
  return false;
}
```
next：

23-34行：首先要找出哪个时间戳最小，然后定位到对应的的iterator

40行：找到对应的元素用来填充timestamps和values数组

45-49行：把所有的最小时间戳的数据都读出来，填充到timestamps和values数组



这里有一点要注意的是：

聚合函数，需要读取所有的值，进行聚合，这里遍历不是hasNext，而是hasNextValue

```
public DataPoint next() {
  final int size = iterators.length;
  long min_ts = Long.MAX_VALUE;

  // In case we reached the end of one or more Spans, we need to make sure
  // we mark them as such by zeroing their current timestamp.  There may
  // be multiple Spans that reached their end at once, so check them all.
  for (int i = current; i < size; i++) {
    if (timestamps[i + size] == TIME_MASK) {
      //LOG.debug("Expiring last DP for #" + current);
      timestamps[i] = 0;
    }
  }

  // Now we need to find which Span we'll consume next.  We'll pick the
  // one that has the data point with the smallest timestamp since we want to
  // return them in chronological order.
  current = -1;
  // If there's more than one Span with the same smallest timestamp, we'll
  // set this to true so we can fetch the next data point in all of them at
  // the same time.
  boolean multiple = false;
  for (int i = 0; i < size; i++) {
    final long timestamp = timestamps[size + i] & TIME_MASK;
    if (timestamp <= end_time) {
      if (timestamp < min_ts) {
        min_ts = timestamp;
        current = i;
        // We just found a new minimum so right now we can't possibly have
        // multiple Spans with the same minimum.
        multiple = false;
      } else if (timestamp == min_ts) {
        multiple = true;
      }
    }
  }
  if (current < 0) {
    throw new NoSuchElementException("no more elements");
  }
  moveToNext(current);
  if (multiple) {
    //LOG.debug("Moving multiple DPs at time " + min_ts);
    // We know we saw at least one other data point with the same minimum
    // timestamp after `current', so let's move those ones too.
    for (int i = current + 1; i < size; i++) {
      final long timestamp = timestamps[size + i] & TIME_MASK;
      if (timestamp == min_ts) {
        moveToNext(i);
      }
    }
  }

  return this;
}

/**
 * Makes iterator number {@code i} move forward to the next data point.
 * @param i The index in {@link #iterators} of the iterator.
 */
private void moveToNext(final int i) {
  final int next = iterators.length + i;
  timestamps[i] = timestamps[next];
  values[i] = values[next];

  final SeekableView it = iterators[i];
  if (it.hasNext()) {
    putDataPoint(next, it.next());
  } else {
    endReached(i);
  }
}

public void remove() {
  throw new UnsupportedOperationException();
}

// ---------------------- //
// SeekableView interface //
// ---------------------- //

public void seek(final long timestamp) {
  for (final SeekableView it : iterators) {
    it.seek(timestamp);
  }
}

// ------------------- //
// DataPoint interface //
// ------------------- //

public long timestamp() {
  return timestamps[current] & TIME_MASK;
}

public boolean isInteger() {
  if (rate) {
    // An rate can never be precisely represented without floating point.
    return false;
  }
  // If at least one of the values we're going to aggregate or interpolate
  // with is a float, we have to convert everything to a float.
  for (int i = timestamps.length - 1; i >= 0; i--) {
    if ((timestamps[i] & FLAG_FLOAT) == FLAG_FLOAT) {
      return false;
    }
  }
  return true;
}

public long longValue() {
  if (isInteger()) {
    pos = -1;
    return aggregator.runLong(this);
  }
  throw new ClassCastException("current value is a double: " + this);
}

public double doubleValue() {
  if (!isInteger()) {
    pos = -1;
    final double value = aggregator.runDouble(this);
    //LOG.debug("aggregator returned " + value);
    if (Double.isInfinite(value)) {
      throw new IllegalStateException("Got Infinity: "
         + value + " in this " + this);
    }
    return value;
  }
  throw new ClassCastException("current value is a long: " + this);
}

public double toDouble() {
  return isInteger() ? longValue() : doubleValue();
}

// -------------------------- //
// Aggregator.Longs interface //
// -------------------------- //

public boolean hasNextValue() {
  return hasNextValue(false);
}

/**
 * Returns whether or not there are more values to aggregate.
 * @param update_pos Whether or not to also move the internal pointer
 * {@link #pos} to the index of the next value to aggregate.
 * @return true if there are more values to aggregate, false otherwise.
 */
private boolean hasNextValue(boolean update_pos) {
  final int size = iterators.length;
  for (int i = pos + 1; i < size; i++) {
    if (timestamps[i] != 0) {
      //LOG.debug("hasNextValue -> true #" + i);
      if (update_pos) {
        pos = i;
      }
      return true;
    }
  }
  //LOG.debug("hasNextValue -> false (ran out)");
  return false;
}

public long nextLongValue() {
  if (hasNextValue(true)) {
    final long y0 = values[pos];
    if (rate) {
      throw new AssertionError("Should not be here, impossible! " + this);
    }
    if (current == pos) {
      return y0;
    }
    final long x = timestamps[current] & TIME_MASK;
    final long x0 = timestamps[pos] & TIME_MASK;
    if (x == x0) {
      return y0;
    }
    final long y1 = values[pos + iterators.length];
    final long x1 = timestamps[pos + iterators.length] & TIME_MASK;
    if (x == x1) {
      return y1;
    }
    if ((x1 & Const.MILLISECOND_MASK) != 0) {
      throw new AssertionError("x1=" + x1 + " in " + this);
    }
    final long r;
    switch (method) {
      case LERP:
        r = y0 + (x - x0) * (y1 - y0) / (x1 - x0);
        //LOG.debug("Lerping to time " + x + ": " + y0 + " @ " + x0
        //          + " -> " + y1 + " @ " + x1 + " => " + r);
        break;
      case ZIM:
        r = 0;
        break;
      case MAX:
        r = Long.MAX_VALUE;
        break;
      case MIN:
        r = Long.MIN_VALUE;
        break;
      default:
        throw new IllegalDataException("Invalid interpolation somehow??");
    }
    return r;
  }
  throw new NoSuchElementException("no more longs in " + this);
}

// ---------------------------- //
// Aggregator.Doubles interface //
// ---------------------------- //

public double nextDoubleValue() {
  if (hasNextValue(true)) {
    final double y0 = ((timestamps[pos] & FLAG_FLOAT) == FLAG_FLOAT
                       ? Double.longBitsToDouble(values[pos])
                       : values[pos]);
    if (current == pos) {
      //LOG.debug("Exact match, no lerp needed");
      return y0;
    }
    if (rate) {
      // No LERP for the rate. Just uses the rate of any previous timestamp.
      // If x0 is smaller than the current time stamp 'x', we just use
      // y0 as a current rate of the 'pos' span. If x0 is bigger than the
      // current timestamp 'x', we don't go back further and just use y0
      // instead. It happens only at the beginning of iteration.
      // TODO: Use the next rate the time range of which includes the current
      // timestamp 'x'.
      return y0;
    }
    final long x = timestamps[current] & TIME_MASK;
    final long x0 = timestamps[pos] & TIME_MASK;
    if (x == x0) {
      //LOG.debug("No lerp needed x == x0 (" + x + " == "+x0+") => " + y0);
      return y0;
    }
    final int next = pos + iterators.length;
    final double y1 = ((timestamps[next] & FLAG_FLOAT) == FLAG_FLOAT
                       ? Double.longBitsToDouble(values[next])
                       : values[next]);
    final long x1 = timestamps[next] & TIME_MASK;
    if (x == x1) {
      //LOG.debug("No lerp needed x == x1 (" + x + " == "+x1+") => " + y1);
      return y1;
    }
    if ((x1 & Const.MILLISECOND_MASK) != 0) {
      throw new AssertionError("x1=" + x1 + " in " + this);
    }
    final double r;
    switch (method) {
    case LERP:
      r = y0 + (x - x0) * (y1 - y0) / (x1 - x0);
      //LOG.debug("Lerping to time " + x + ": " + y0 + " @ " + x0
      //          + " -> " + y1 + " @ " + x1 + " => " + r);
      break;
    case ZIM:
      r = 0;
      break;
    case MAX:
      r = Double.MAX_VALUE;
      break;
    case MIN:
      r = Double.MIN_VALUE;
      break;
    default:
      throw new IllegalDataException("Invalid interploation somehow??");
  }
    return r;
  }
  throw new NoSuchElementException("no more doubles in " + this);
}
```



## 代码执行顺序：

Annotation.getGlobalAnnotations->==GlobalCB==->net.opentsdb.core.TSQuery#buildQueriesAsync->==BuildCB==->query.runAsync->==QueriesCB==->SendIt,所有的操作都是使用ErrorCB作为错误处理机制。

net.opentsdb.core.TSQuery#buildQueriesAsync

		configureFromQuery->==GroupFinished==这里完成了所有的子查询的GroupBy的配置工作,返回的是子查询列表



BuildCB：net.opentsdb.core.TsdbQuery#runAsync -> GroupByAndAggregateCB->返回处理后的数据

子查询：net.opentsdb.core.TsdbQuery#runAsync ->  net.opentsdb.core.SaltScanner#scan -> net.opentsdb.core.SaltScanner#scan（CallBack）-> 处理数据

## 第一步：加载Annotation

> net.opentsdb.meta.Annotation#getGlobalAnnotations



可选。执行条件为：

```
 !data_query.getNoAnnotations() && data_query.getGlobalAnnotations()
```

这里可以学习到Hbase最简单的扫描逻辑：new一个scanner，添加一个callback，积攒数据，数据全部返回后，调用CallBack，CallBack把数据保存到成员变量里面。这是比较典型的异步数据处理套路。



*一般来讲，一个CallBack都是数据已经获取到了，然后再被回调；但是当一个CallBack需要返回一个Deferred对象时，需要调用addCallbackDeferring。看注释是为了保持类型信息。（个人理解）*



```
/**
 * Registers a callback.
 * <p>
 * This has the exact same effect as {@link #addCallback}, but keeps the type
 * information "correct" when the callback to add returns a {@code Deferred}.
 * @param cb The callback to register.
 * @return {@code this} with an "updated" type.
 */
@SuppressWarnings("unchecked")
public <R, D extends Deferred<R>>
  Deferred<R> addCallbackDeferring(final Callback<D, T> cb) {
  return addCallbacks((Callback<R, T>) ((Object) cb), Callback.PASSTHROUGH);
}
```

```
scanner.nextRows().addCallbackDeferring(this);
```



```
/**
 * Scans through the global annotation storage rows and returns a list of 
 * parsed annotation objects. If no annotations were found for the given
 * timespan, the resulting list will be empty.
 * @param tsdb The TSDB to use for storage access
 * @param start_time Start time to scan from. May be 0
 * @param end_time End time to scan to. Must be greater than 0
 * @return A list with detected annotations. May be empty.
 * @throws IllegalArgumentException if the end timestamp has not been set or 
 * the end time is less than the start time
 */
public static Deferred<List<Annotation>> getGlobalAnnotations(final TSDB tsdb, 
    final long start_time, final long end_time) {
  if (end_time < 1) {
    throw new IllegalArgumentException("The end timestamp has not been set");
  }
  if (end_time < start_time) {
    throw new IllegalArgumentException(
        "The end timestamp cannot be less than the start timestamp");
  }
  
  /**
   * Scanner that loops through the [0, 0, 0, timestamp] rows looking for
   * global annotations. Returns a list of parsed annotation objects.
   * The list may be empty.
   */
  final class ScannerCB implements Callback<Deferred<List<Annotation>>, 
    ArrayList<ArrayList<KeyValue>>> {
    final Scanner scanner;
    final ArrayList<Annotation> annotations = new ArrayList<Annotation>();
    
    /**
     * Initializes the scanner
     */
    public ScannerCB() {
      final byte[] start = new byte[Const.SALT_WIDTH() + 
                                    TSDB.metrics_width() + 
                                    Const.TIMESTAMP_BYTES];
      final byte[] end = new byte[Const.SALT_WIDTH() + 
                                  TSDB.metrics_width() + 
                                  Const.TIMESTAMP_BYTES];
      
      final long normalized_start = (start_time - 
          (start_time % Const.MAX_TIMESPAN));
      final long normalized_end = (end_time - 
          (end_time % Const.MAX_TIMESPAN) + Const.MAX_TIMESPAN);
      
      Bytes.setInt(start, (int) normalized_start, 
          Const.SALT_WIDTH() + TSDB.metrics_width());
      Bytes.setInt(end, (int) normalized_end, 
          Const.SALT_WIDTH() + TSDB.metrics_width());

      scanner = tsdb.getClient().newScanner(tsdb.dataTable());
      scanner.setStartKey(start);
      scanner.setStopKey(end);
      scanner.setFamily(FAMILY);
    }
    
    public Deferred<List<Annotation>> scan() {
      return scanner.nextRows().addCallbackDeferring(this);
    }
    
    @Override
    public Deferred<List<Annotation>> call (
        final ArrayList<ArrayList<KeyValue>> rows) throws Exception {
      if (rows == null || rows.isEmpty()) {
        return Deferred.fromResult((List<Annotation>)annotations);
      }
      
      for (final ArrayList<KeyValue> row : rows) {
        for (KeyValue column : row) {
          if ((column.qualifier().length == 3 || column.qualifier().length == 5) 
              && column.qualifier()[0] == PREFIX()) {
            Annotation note = JSON.parseToObject(column.value(),
                Annotation.class);
            if (note.start_time < start_time || note.end_time > end_time) {
              continue;
            }
            annotations.add(note);
          }
        }
      }
      
      return scan();
    }
    
  }

  return new ScannerCB().scan();
}
```

当Annotation加载完毕后，调用回调函数：

```
/** Handles storing the global annotations after fetching them */
class GlobalCB implements Callback<Object, List<Annotation>> {
  public Object call(final List<Annotation> annotations) throws Exception {
    globals.addAll(annotations);
    return data_query.buildQueriesAsync(tsdb).addCallback(new BuildCB());
  }
}
```

这里就是保存到成员变量globals，然后真正执行查询。



## 第二步：查询解析

### 总体流程

遍历每个子查询，启动查询解析工作，把TsSubquery转换为TsdbQuery，把解析完毕后的子查询数组作为返回对象：

```
public Deferred<Query[]> buildQueriesAsync(final TSDB tsdb) {
  final Query[] tsdb_queries = new Query[queries.size()];
  
  final List<Deferred<Object>> deferreds =
      new ArrayList<Deferred<Object>>(queries.size());
  for (int i = 0; i < queries.size(); i++) {
    final Query query = tsdb.newQuery();
    //
    deferreds.add(query.configureFromQuery(this, i));
    tsdb_queries[i] = query;
  }
  
  class GroupFinished implements Callback<Query[], ArrayList<Object>> {
    @Override
    public Query[] call(final ArrayList<Object> deferreds) {
      return tsdb_queries;
    }
    @Override
    public String toString() {
      return "Query compile group callback";
    }
  }
  
  return Deferred.group(deferreds).addCallback(new GroupFinished());
}
```

从上面的CallBack可以看出，每一个configureFromQuery执行完毕后就执行GroupFinished把组装的tsdb_queries返回了。

### 子查询的解析构建

> net.opentsdb.core.TsdbQuery#configureFromQuery

#### 度量的编码查询

主要是度量和过滤器名称和值到二进制编码的查询：

```
final UniqueId metrics;
 
@Override
public Deferred<Object> configureFromQuery(final TSQuery query, 
    final int index) {
    
     // fire off the callback chain by resolving the metric first
      return tsdb.metrics.getIdAsync(sub_query.getMetric())
          .addCallbackDeferring(new MetricCB());
    }
```



查询完了Metrics名称到编码的转换之后，查询过滤器的编码：

getIdAsync->==MetricCB==->resolveTagFilters->->==FilterCB==

	resolveTagFilters->loop filters:resolveTagkName->ResolvedCB->保存到成员变量net.opentsdb.query.filter.TagVFilter.tagk_bytes

```java
class MetricCB implements Callback<Deferred<Object>, byte[]> {
  @Override
  public Deferred<Object> call(final byte[] uid) throws Exception {
    //这里获取到了metrics的uid
    metric = uid;
    if (filters != null) {
      return Deferred.group(resolveTagFilters()).addCallback(new FilterCB());
    } else {
      return Deferred.fromResult(null);
    }
  }
  
  private List<Deferred<byte[]>> resolveTagFilters() {
    final List<Deferred<byte[]>> deferreds = 
        new ArrayList<Deferred<byte[]>>(filters.size());
    for (final TagVFilter filter : filters) {
      // determine if the user is asking for pre-agg data
      if (filter instanceof TagVLiteralOrFilter && tsdb.getAggTagKey() != null) {
        if (filter.getTagk().equals(tsdb.getAggTagKey())) {
          if (tsdb.getRawTagValue() != null && 
              !filter.getFilter().equals(tsdb.getRawTagValue())) {
            pre_aggregate = true;
          }
        }
      }
      
      deferreds.add(filter.resolveTagkName(tsdb));
    }
    return deferreds;
  }
}	
```



#### Tagk的编码查询

- 普通的解析：

```
public Deferred<byte[]> resolveTagkName(final TSDB tsdb) {
  class ResolvedCB implements Callback<byte[], byte[]> {
    @Override
    public byte[] call(final byte[] uid) throws Exception {
      tagk_bytes = uid;
      return uid;
    }
  }
  
  return tsdb.getUIDAsync(UniqueIdType.TAGK, tagk)
      .addCallback(new ResolvedCB());
}
```

- TagVLiteralOrFilter的解析，这个过滤器很常见，所以需要关注一下：

net.opentsdb.query.filter.TagVLiteralOrFilter#resolveTagkName

TagVLiteralOrFilter改写了父类的resolveTagkName，对其中的值也进行了编码解析

解析值得两个条件：大小写敏感（不是大小写不敏感，没有超过配置的参数tsd.query.filter.expansion_limit

```
@Override
public Deferred<byte[]> resolveTagkName(final TSDB tsdb) {
  final Config config = tsdb.getConfig();
  
  // resolve tag values if the filter is NOT case insensitive and there are 
  // fewer literals than the expansion limit
  if (!case_insensitive && 
      literals.size() <= config.getInt("tsd.query.filter.expansion_limit")) {
    return resolveTags(tsdb, literals);
  } else {
    return super.resolveTagkName(tsdb);
  }
}
```

#### 分组的设置



> How to group by and aggregate in openTSDB like RDBMS?
>
> https://stackoverflow.com/questions/34648064/how-to-group-by-and-aggregate-in-opentsdb-like-rdbms
>
> Now way for openTSDB to do it. Also if there is requirement like this, then openTSDB may be not your choice. openTSDB is time series db, also for kariosDB. I tried in openTSDB and kariosDB and found they both can not.
>
> Because in openTSDB , the group by is one thing, the aggregate is another thing. Not like RDBMS, the agg works on the group by. In openTSDB the agg works on the downsample
>



> 感觉这里有个bUG，因为tagk在下一个循环没有重置为下一个的值。
>
> 后来google了一下，在如下链接找到关于这点的讨论：
>
> https://github.com/OpenTSDB/opentsdb/issues/973
>
> 不过看官方的答复，不建议有相同的tagk过滤，所以do循环体里面基本是不生效的，只看while就够了



主要的思路很简单：就是找出groupBy属性为true的tagk的列表，如果这个过滤器里面能够明确确定是哪些tagv需要过滤，那么就把tagv的一串二进制字符串串起来，放到一个map里面去，用tagk的二进制编码做Key。

这里会设置过滤器的setPostScan属性，因为后面数据加载进来后，会根据过滤器进行过滤，这里如果setPostScan设置为false表示，数据已经根据tagk和tagv进行了过滤，所以后续就没必要进行这一步了。因为选择出来的数据，没有符合这个过滤器的条件的了。

```java
以下是涉及到的成员变量的声明：

 /**
   * Tag key and values to use in the row key filter, all pre-sorted
   */
  private ByteMap<byte[][]> row_key_literals;
  private List<ByteMap<byte[][]>> row_key_literals_list;
    /**
   * Tags by which we must group the results.
   * Each element is a tag ID.
   * Invariant: an element cannot be both in this array and in {@code tags}.
   */
  private ArrayList<byte[]> group_bys;
  

class FilterCB implements Callback<Object, ArrayList<byte[]>> {
  @Override
  public Object call(final ArrayList<byte[]> results) throws Exception {
    findGroupBys();
    return null;
  }
}

ByteMap就是个二进制memcmp的TreeMap
  public static final class ByteMap<V> extends TreeMap<byte[], V>
    implements Iterable<Map.Entry<byte[], V>> {

    public ByteMap() {
      super(MEMCMP);
    }
 }
    
private void findGroupBys() {
    row_key_literals = new ByteMap<byte[][]>();

    //filters是按照tagk的uid二进制字节序来排序的！
    Collections.sort(filters);

    final Iterator<TagVFilter> current_iterator = filters.iterator();
    final Iterator<TagVFilter> look_ahead = filters.iterator();
    byte[] tagk = null;
    TagVFilter next = look_ahead.hasNext() ? look_ahead.next() : null;
    int row_key_literals_count = 0;

    while (current_iterator.hasNext()) {
      next = look_ahead.hasNext() ? look_ahead.next() : null;
      int gbs = 0;
      // sorted!
      final ByteMap<Void> literals = new ByteMap<Void>();
      final List<TagVFilter> literal_filters = new ArrayList<TagVFilter>();
      TagVFilter current = null;
      //这里主要是为了解决一个问题：同一个subquery不同的过滤器之间，
      // 有可能出现相同的tagk，这时候会把
      //第二个的过滤器里面的groupby也设置为true
      do { // yeah, I'm breakin out the do!!!
        current = current_iterator.next();
          
          这里的tagk值用来作为循环结束的判断
        if (tagk == null) {
          tagk = new byte[TSDB.tagk_width()];
          System.arraycopy(current.getTagkBytes(), 0, tagk, 0, TSDB.tagk_width());
        }
        
        if (current.isGroupBy()) {
          gbs++;
        }
        if (!current.getTagVUids().isEmpty()) {
          for (final byte[] uid : current.getTagVUids()) {
            literals.put(uid, null);
          }
          literal_filters.add(current);
        }

        if (next != null && Bytes.memcmp(tagk, next.getTagkBytes()) != 0) {
          break;
        }
        next = look_ahead.hasNext() ? look_ahead.next() : null;

      } while (current_iterator.hasNext() && 
          Bytes.memcmp(tagk, current.getTagkBytes()) == 0);

      if (gbs > 0) {
        if (group_bys == null) {
          group_bys = new ArrayList<byte[]>();
        }
        group_bys.add(current.getTagkBytes());
      }
      
      if (literals.size() > 0) {
        if (literals.size() + row_key_literals_count > expansion_limit) {
          LOG.debug("Skipping literals for " + current.getTagk() + 
              " as it exceedes the limit");
          //has_filter_cannot_use_get = true;
        } else {
          final byte[][] values = new byte[literals.size()][];
          literals.keySet().toArray(values);
          row_key_literals.put(current.getTagkBytes(), values);
          row_key_literals_count += values.length;
          
          for (final TagVFilter filter : literal_filters) {
            // 看注释：post_scan Whether or not this filter should be executed against
            //  scan results ，就是读取到结果以后还要执行scan。
            filter.setPostScan(false);
          }
        }
      } else {
        row_key_literals.put(current.getTagkBytes(), null);
        // no literal values, just keys, so we can't multi-get
        if (search_query_failure) {
          use_multi_gets = false;
        }
      }
      
      // make sure the multi-get cardinality doesn't exceed our limit (or disable
      // multi-gets)
      if ((use_multi_gets && override_multi_get)) {
        int multi_get_limit = tsdb.getConfig().getInt("tsd.query.multi_get.limit");
        int cardinality = filters.size() * row_key_literals_count;
        if (cardinality > multi_get_limit) {
          use_multi_gets = false;
        } else if (search_query_failure) {
          row_key_literals_list.add(row_key_literals);
        }
        // TODO - account for time as well
      }
    }
  }
```

## 

截止到目前为止，实际上已经完成了如下步骤：

1.Annotation的加载

2.tagk和tagv的名称到id的转换

3.groupby的查找完成。



这时候buildQueriesAsync就算完成了，build完成之后，会调用GroupFinished这个回调，从这个回调的名称也可以判断出，分组已经完成，GroupFinished返回的是已经转配好的TsdbQuery对象数组。

接下来应该就是真正的执行这些查询了，因为该准备的都准备好了。

这时候继续回到net.opentsdb.tsd.QueryRpc的handleQuery里面，可以看到，查询对象都准备好了，这时候会执行BuildCB，意思就是每个子查询都Build好了，可以执行了。

## 第三步：查询执行

```java
class BuildCB implements Callback<Deferred<Object>, Query[]> {
  @Override
  public Deferred<Object> call(final Query[] queries) {
    final ArrayList<Deferred<DataPoints[]>> deferreds = 
        new ArrayList<Deferred<DataPoints[]>>(queries.length);
    for (final Query query : queries) {
      deferreds.add(query.runAsync());
    }
    return Deferred.groupInOrder(deferreds).addCallback(new QueriesCB());
  }
}
```

所以我们继续看，runAsync执行那些动作，这些动作执行完之后，会调用QueriesCB



> net.opentsdb.core.TsdbQuery#runAsync

```java
@Override
public Deferred<DataPoints[]> runAsync() throws HBaseException {
  return findSpans().addCallback(new GroupByAndAggregateCB());
}
```

#### 数据的处理逻辑：

这里涉及到真正的数据处理逻辑了，需要理清楚数据之间的处理关系。

1.数据是由时间点组成的，时间点是时间线+时间戳

2.数据首先按照时间线进行分组，这个分组成为一个Span

3.因为参与查询要求的GroupBy的维度一定是时间线的维度的一个子集，因此将Span按照查询的要求进行GroupBy的维度再进行一次汇总，每个分组称为SpanGroup

4.降采样处理逻辑是在Span里面做的

5.数据过滤处理逻辑？

6.汇总处理逻辑？

5.数据不保存计算结果，而是采用了懒加载的方式，在用到时才进行计算。这是通过各种iterator来实现的，而每个iterator都实现了datapoint接口，datapoint结果包含数据最基本的元素：时间戳，值。



#### 数据的加载



##### Salt的计算：

Salt Width最大为8个字节，可配置。

```
static void setSaltWidth(final int width) {
  if (width < 0 || width > 8) {
    throw new IllegalArgumentException("Salt width must be between 0 and 8");
  }
  SALT_WIDTH = width;
}
```

根据时间线计算Hash，模上 SALT_BUCKETS就是应该落到哪个桶，假设为i，根据i进行一系列的唯一运算，得到结果。

```java
private static int SALT_BUCKETS = 20;
```

```

public static void prefixKeyWithSalt(final byte[] row_key) {
  if (Const.SALT_WIDTH() > 0) {
    if (row_key.length < (Const.SALT_WIDTH() + TSDB.metrics_width()) || 
      (Bytes.memcmp(row_key, new byte[Const.SALT_WIDTH() + TSDB.metrics_width()], 
          Const.SALT_WIDTH(), TSDB.metrics_width()) == 0)) {
      // ^ Don't salt the global annotation row, leave it at zero
      return;
    }
    final int tags_start = Const.SALT_WIDTH() + TSDB.metrics_width() + 
        Const.TIMESTAMP_BYTES;
    
    // we want the metric and tags, not the timestamp
    final byte[] salt_base = 
        new byte[row_key.length - Const.SALT_WIDTH() - Const.TIMESTAMP_BYTES];
    System.arraycopy(row_key, Const.SALT_WIDTH(), salt_base, 0, TSDB.metrics_width());
    System.arraycopy(row_key, tags_start,salt_base, TSDB.metrics_width(), 
        row_key.length - tags_start);
    int modulo = Arrays.hashCode(salt_base) % Const.SALT_BUCKETS();
    if (modulo < 0) {
      // make sure we return a positive salt.
      modulo = modulo * -1;
    }
  
    final byte[] salt = getSaltBytes(modulo);
    System.arraycopy(salt, 0, row_key, 0, Const.SALT_WIDTH());
  } // else salting is disabled so it's a no-op
}

  public static byte[] getSaltBytes(final int bucket) {
    final byte[] bytes = new byte[Const.SALT_WIDTH()];
    int shift = 0;
    for (int i = 1;i <= Const.SALT_WIDTH(); i++) {
      bytes[Const.SALT_WIDTH() - i] = (byte) (bucket >>> shift);
      shift += 8;
    }
    return bytes;
  }
```



##### 生成Scanner

```
public static Scanner getMetricScanner(final TSDB tsdb, final int salt_bucket, 
    final byte[] metric, final int start, final int stop, 
    final byte[] table, final byte[] family) {
  final short metric_width = TSDB.metrics_width();
  final int metric_salt_width = metric_width + Const.SALT_WIDTH();
  final byte[] start_row = new byte[metric_salt_width + Const.TIMESTAMP_BYTES];
  final byte[] end_row = new byte[metric_salt_width + Const.TIMESTAMP_BYTES];
  
  if (Const.SALT_WIDTH() > 0) {
    final byte[] salt = RowKey.getSaltBytes(salt_bucket);
    System.arraycopy(salt, 0, start_row, 0, Const.SALT_WIDTH());
    System.arraycopy(salt, 0, end_row, 0, Const.SALT_WIDTH());
  }
  
  Bytes.setInt(start_row, start, metric_salt_width);
  Bytes.setInt(end_row, stop, metric_salt_width);
  
  System.arraycopy(metric, 0, start_row, Const.SALT_WIDTH(), metric_width);
  System.arraycopy(metric, 0, end_row, Const.SALT_WIDTH(), metric_width);
  
  final Scanner scanner = tsdb.getClient().newScanner(table);
  scanner.setMaxNumRows(tsdb.getConfig().scanner_maxNumRows());
  scanner.setStartKey(start_row);
  scanner.setStopKey(end_row);
  scanner.setFamily(family);
  return scanner;
}
```

这里是生成Scanner的最基本步骤：

1.设置salt 前缀值 ,if salt enabled

2.设置metrics 值

3.根据开始时间和结束时间设置扫描startKey和stopKey

4.设置cf（固定值 t ）

##### 设置Scanner过滤器：

- TSUID的过滤设置

  TSUID都是可见字符，需要首先变为原来的二进制，

  8-13行 把所有的tagk和tagv的uid变为二进制；

  10行这里忽略了metrics，因为这个在前面设置rowkey的start和end的时候已经设置好了。



  主要看看正则表达式：

  1.(?s)表示DOTALL，

  `(?:)` creates a *non-capturing group*. It groups things together without creating a backreference. 



| [Positive lookahead](https://www.regular-expressions.info/lookaround.html) | `(?=regex)` | Matches at a position where the pattern inside the lookahead can be matched. Matches only the position. It does not consume any characters or expand the match. In a pattern like `one(?=two)three`, both `two` and `three` have to match at the position where the match of `one` ends. | `t(?=s)` matches the second `t`in `streets`. | YES  | YES  |
| ------------------------------------------------------------ | ----------- | ------------------------------------------------------------ | -------------------------------------------- | ---- | ---- |
| [Negative lookahead](https://www.regular-expressions.info/lookaround.html) | `(?!regex)` | Similar to positive lookahead, except that negative lookahead only succeeds if the regex inside the lookahead fails to match. | `t(?!s)` matches the first `t` in `streets`. | YES  | YES  |

  1.DOTALL语义：这里使用了(?s),主要是因为我们的数据是二进制的，不能当成文本处理

  > java.util.regex.Pattern

      /**
       * Enables dotall mode.
       *
       * <p> In dotall mode, the expression <tt>.</tt> matches any character,
       * including a line terminator.  By default this expression does not match
       * line terminators.
       *
       * <p> Dotall mode can also be enabled via the embedded flag
       * expression&nbsp;<tt>(?s)</tt>.  (The <tt>s</tt> is a mnemonic for
       * "single-line" mode, which is what this is called in Perl.)  </p>
       */
  \\Q\000\000\001\000\000\002\\E

  \Q和\E之间的内容表示引用QUote，引用就是不代表特殊含义，里面的仅仅是数据而已。

```
public static String getRowKeyTSUIDRegex(final List<String> tsuids) {
  Collections.sort(tsuids);
  
  // first, convert the tags to byte arrays and count up the total length
  // so we can allocate the string builder
  final short metric_width = TSDB.metrics_width();
  int tags_length = 0;
  final ArrayList<byte[]> uids = new ArrayList<byte[]>(tsuids.size());
  
  //先把明文转换为二进制字符
  for (final String tsuid : tsuids) {
    final String tags = tsuid.substring(metric_width * 2);
    final byte[] tag_bytes = UniqueId.stringToUid(tags);
    tags_length += tag_bytes.length;
    uids.add(tag_bytes);
  }
  
  // Generate a regexp for our tags based on any metric and timestamp (since
  // those are handled by the row start/stop) and the list of TSUID tagk/v
  // pairs. The generated regex will look like: ^.{7}(tags|tags|tags)$
  // where each "tags" is similar to \\Q\000\000\001\000\000\002\\E
  final StringBuilder buf = new StringBuilder(
      13  // "(?s)^.{N}(" + ")$"
      + (tsuids.size() * 11) // "\\Q" + "\\E|"
      + tags_length); // total # of bytes in tsuids tagk/v pairs
  
  // Alright, let's build this regexp.  From the beginning...
  buf.append("(?s)"  // Ensure we use the DOTALL flag.
             + "^.{")
     // ... start by skipping the metric ID and timestamp.
     .append(Const.SALT_WIDTH() + metric_width + Const.TIMESTAMP_BYTES)
     .append("}(");
  
  for (final byte[] tags : uids) {
     // quote the bytes
    buf.append("\\Q");
    addId(buf, tags, true);
    buf.append('|');
  }
  
  // Replace the pipe of the last iteration, close and set
  buf.setCharAt(buf.length() - 1, ')');
  buf.append("$");
  return buf.toString();
}

这里主要是针对\E进行了特殊处理。因为数据里面有可能是\E，必须转换为\\\E
public static void addId(final StringBuilder buf, final byte[] id, 
      final boolean close) {
    boolean backslash = false;
    for (final byte b : id) {
      buf.append((char) (b & 0xFF));
      if (b == 'E' && backslash) {  // If we saw a `\' and now we have a `E'.
        // So we just terminated the quoted section because we just added \E
        // to `buf'.  So let's put a litteral \E now and start quoting again.
        buf.append("\\\\E\\Q");
      } else {
        backslash = b == '\\';
      }
    }
    if (close) {
      buf.append("\\E");
    }
  }
```

这个正则表达式的意思是：

1. DOTALL，作为单行对待
2. 以Const.SALT_WIDTH() + metric_width + Const.TIMESTAMP_BYTES个任意字符开头
3. 后面跟着多个tagk(3字节)tagv三字节（如注释所示： \\Q\000\000\001\000\000\002\\E）
4. 添加结尾符号$

实际就是根据确定的tagk tagv增加一种过滤，因为tagk，tagv都已经是很明确的了，都包含在tsuiud这个时间线里面了。

- 非时间线的过滤

  如果explicit_tags那么使用FuzzyRowFilter，否则使用KeyRegexpFilter

> net.opentsdb.core.TsdbQuery#createAndSetTSUIDFilter

```
private void createAndSetFilter(final Scanner scanner) {
  QueryUtil.setDataTableScanFilter(scanner, group_bys, row_key_literals, 
      explicit_tags, enable_fuzzy_filter, 
      (end_time == UNSET
      ? -1  // Will scan until the end (0xFFF...).
      : (int) getScanEndTimeSeconds()));
}

public static void setDataTableScanFilter(
      final Scanner scanner, 
      final List<byte[]> group_bys, 
      final ByteMap<byte[][]> row_key_literals,
      final boolean explicit_tags,
      final boolean enable_fuzzy_filter,
      final int end_time) {
    
    // no-op
    if ((group_bys == null || group_bys.isEmpty()) 
        && (row_key_literals == null || row_key_literals.isEmpty())) {
      return;
    }
    
    final int prefix_width = Const.SALT_WIDTH() + TSDB.metrics_width() + 
        Const.TIMESTAMP_BYTES;
    final short name_width = TSDB.tagk_width();
    final short value_width = TSDB.tagv_width();
    final byte[] fuzzy_key;
    final byte[] fuzzy_mask;
    //如果明确所有的tag已经提供了，那么只需要使用Hbase的模糊匹配，需要提供掩码
    if (explicit_tags && enable_fuzzy_filter) {
      fuzzy_key = new byte[prefix_width + (row_key_literals.size() * 
          (name_width + value_width))];
      fuzzy_mask = new byte[prefix_width + (row_key_literals.size() *
          (name_width + value_width))];
      System.arraycopy(scanner.getCurrentKey(), 0, fuzzy_key, 0, 
          scanner.getCurrentKey().length);
    } else {
      fuzzy_key = fuzzy_mask = null;
    }
    
    final String regex = getRowKeyUIDRegex(group_bys, row_key_literals, 
        explicit_tags, fuzzy_key, fuzzy_mask);
    final KeyRegexpFilter regex_filter = new KeyRegexpFilter(
        regex.toString(), Const.ASCII_CHARSET);
    if (LOG.isDebugEnabled()) {
      LOG.debug("Regex for scanner: " + scanner + ": " + 
          byteRegexToString(regex));
    }
    
    //普通查询，只使用正则表达式
    if (!(explicit_tags && enable_fuzzy_filter)) {
      scanner.setFilter(regex_filter);
      return;
    }
    
    //explicit_tags的话，通过前面的掩码，设置FuzzyRowFilter
    scanner.setStartKey(fuzzy_key);
    final byte[] stop_key = Arrays.copyOf(fuzzy_key, fuzzy_key.length);
    Internal.setBaseTime(stop_key, end_time);
    int idx = Const.SALT_WIDTH() + TSDB.metrics_width() + 
        Const.TIMESTAMP_BYTES + TSDB.tagk_width();
    // max out the tag values
    while (idx < stop_key.length) {
      for (int i = 0; i < TSDB.tagv_width(); i++) {
        stop_key[idx++] = (byte) 0xFF;
      }
      idx += TSDB.tagk_width();
    }
    scanner.setStopKey(stop_key);
    final List<ScanFilter> filters = new ArrayList<ScanFilter>(2);
    filters.add(
        new FuzzyRowFilter(
            new FuzzyRowFilter.FuzzyFilterPair(fuzzy_key, fuzzy_mask)));
    filters.add(regex_filter);
    scanner.setFilter(new FilterList(filters));
  }
  
  
```



注意：

1. findGroupBys里面设置了group_bys和row_key_literals； 

   row_key_literals = new ByteMap<byte[][]>();

2. tsd.query.enable_fuzzy_filter配置项对应enable_fuzzy_filter

3. explicit_tags是查询带过来的参数

下面我们主要看看row_key_literals和group_bys是怎么用的？





主要看这个函数：

73-89行：针对groupby的，正则表达式形式为 首先是key，然后是value|value，如果value为空，那么就是一个通配符${N}，这样就只匹配了tagk；

97-98行，匹配任意的tagk/tagv对

上面两处，构造了完成的正则表达式；

> net.opentsdb.query.QueryUtil#getRowKeyUIDRegex(java.util.List<byte[]>, org.hbase.async.Bytes.ByteMap<byte[][]>, boolean, byte[], byte[])

```
public static String getRowKeyUIDRegex(
    final List<byte[]> group_bys, 
    final ByteMap<byte[][]> row_key_literals, 
    final boolean explicit_tags,
    final byte[] fuzzy_key, 
    final byte[] fuzzy_mask) {
  if (group_bys != null) {
    Collections.sort(group_bys, Bytes.MEMCMP);
  }
  final int prefix_width = Const.SALT_WIDTH() + TSDB.metrics_width() + 
      Const.TIMESTAMP_BYTES;
  final short name_width = TSDB.tagk_width();
  final short value_width = TSDB.tagv_width();
  //tagk tagv对，所以这里是6
  final short tagsize = (short) (name_width + value_width);
  // Generate a regexp for our tags.  Say we have 2 tags: { 0 0 1 0 0 2 }
  // and { 4 5 6 9 8 7 }, the regexp will be:
  // "^.{7}(?:.{6})*\\Q\000\000\001\000\000\002\\E(?:.{6})*\\Q\004\005\006\011\010\007\\E(?:.{6})*$"
  final StringBuilder buf = new StringBuilder(
      15  // "^.{N}" + "(?:.{M})*" + "$"
      + ((13 + tagsize) // "(?:.{M})*\\Q" + tagsize bytes + "\\E"
         * ((row_key_literals == null ? 0 : row_key_literals.size()) + 
             (group_bys == null ? 0 : group_bys.size() * 3))));
  // In order to avoid re-allocations, reserve a bit more w/ groups ^^^

  // Alright, let's build this regexp.  From the beginning...
  //确保DOTALL语义，然后跳过SALT和metrics和时间戳，
  buf.append("(?s)"  // Ensure we use the DOTALL flag.
             + "^.{")
     // ... start by skipping the salt, metric ID and timestamp.
     .append(Const.SALT_WIDTH() + TSDB.metrics_width() + Const.TIMESTAMP_BYTES)
     .append("}");

  final Iterator<Entry<byte[], byte[][]>> it = row_key_literals == null ? 
      new ByteMap<byte[][]>().iterator() : row_key_literals.iterator();
  int fuzzy_offset = Const.SALT_WIDTH() + TSDB.metrics_width();
  if (fuzzy_mask != null) {
    // make sure to skip the timestamp when scanning
    while (fuzzy_offset < prefix_width) {
      fuzzy_mask[fuzzy_offset++] = 1;
    }
  }
  
  while(it.hasNext()) {
    Entry<byte[], byte[][]> entry = it.hasNext() ? it.next() : null;
    // TODO - This look ahead may be expensive. We need to get some data around
    // whether it's faster for HBase to scan with a look ahead or simply pass
    // the rows back to the TSD for filtering.
    final boolean not_key = 
        entry.getValue() != null && entry.getValue().length == 0;
    
    // Skip any number of tags.
    if (!explicit_tags) {
      buf.append("(?:.{").append(tagsize).append("})*");
    } else if (fuzzy_mask != null) {
      // TODO - see if we can figure out how to improve the fuzzy filter by
      // setting explicit tag values whenever we can. In testing there was
      // a conflict between the row key regex and fuzzy filter that prevented
      // results from returning properly.
      System.arraycopy(entry.getKey(), 0, fuzzy_key, fuzzy_offset, name_width);
      fuzzy_offset += name_width;
      for (int i = 0; i < value_width; i++) {
        fuzzy_mask[fuzzy_offset++] = 1;
      }
    }
    if (not_key) {
      // start the lookahead as we have a key we explicitly do not want in the
      // results
      buf.append("(?!");
    }
    buf.append("\\Q");
    
    addId(buf, entry.getKey(), true);
    if (entry.getValue() != null && entry.getValue().length > 0) {  // Add a group_by.
      // We want specific IDs.  List them: /(AAA|BBB|CCC|..)/
      buf.append("(?:");
      for (final byte[] value_id : entry.getValue()) {
        if (value_id == null) {
          continue;
        }
        buf.append("\\Q");
        addId(buf, value_id, true);
        buf.append('|');
      }
      // Replace the pipe of the last iteration.
      buf.setCharAt(buf.length() - 1, ')');
    } else {
      buf.append(".{").append(value_width).append('}');  // Any value ID.
    }
    
    if (not_key) {
      // be sure to close off the look ahead
      buf.append(")");
    }
  }
  // Skip any number of tags before the end.
  if (!explicit_tags) {
    buf.append("(?:.{").append(tagsize).append("})*");
  }
  buf.append("$");
  return buf.toString();
}
```



如果启用了Salt，就会生成20个Scanner，否则就是1个Scanner，Tsdb用SaltScanner封装了这种差异，统一用这类来表示：

```
public SaltScanner(final TSDB tsdb, final byte[] metric,
    final List<Scanner> scanners,
    final TreeMap<byte[], Span> spans,
    final List<TagVFilter> filters,
    final boolean delete,
    final RollupQuery rollup_query,
    final QueryStats query_stats,
    final int query_index,
    final TreeMap<byte[], HistogramSpan> histogramSpans,
    final long max_bytes,
    final long max_data_points,
    final List<TagVFilter> value_filters) {
    .......
    
    countdown = new CountDownLatch(scanners.size());
 }   
    
```

每个Scanner执行完毕后，都会将countdown减一，当countdown计数为0的时候表示查询都已经返回了。



```
void close(final boolean ok) {
scanner.close();
...
if (ok && exception == null) {
        validateAndTriggerCallback(kvs, annotations, histograms);
      } else {
        countdown.countDown();
      }
}


private void validateAndTriggerCallback(
          final List<KeyValue> kvs,
          final Map<byte[], List<Annotation>> annotations,
          final List<SimpleEntry<byte[], List<HistogramDataPoint>>> histograms) {

    //减少计数
    countdown.countDown();
    final long count = countdown.getCount();
    if (kvs.size() > 0) {
      //kv_map保存的是每个scanner对应的数据
      kv_map.put((int) count, kvs);
    }
    ......
    //所有的都返回了
    if (countdown.getCount() <= 0) {
      try {
        mergeAndReturnResults();
      } catch (final Exception ex) {
        results.callback(ex);
      }
    }
  }
```



最后tsdb调用

net.opentsdb.core.SaltScanner#scan

启动数据异步加载逻辑；

```
public Object scan() {
  if (scanner_start < 0) {
    scanner_start = DateTime.nanoTime();
  }
  fetch_start = DateTime.nanoTime();
  return scanner.nextRows().addCallback(this).addErrback(new ErrorCb());
}
```



数据的接收：



从scan可以看出，接收到数据之后的回调是this，因此我们看net.opentsdb.core.SaltScanner#call函数：



```
public Object call2(final ArrayList<ArrayList<KeyValue>> rows) throws Exception {
  try {
    //rows == null表示这个Scanner已经扫描到最后了，所以需要关闭Scanner进行后续处理
    if (rows == null) {
      close(true);
      return null;
    }
    .....

    // used for UID resolution if a filter is involved
    //存放的是ID对name的反解析
    final List<Deferred<Object>> lookups =
            filters != null && !filters.isEmpty() ?
                    new ArrayList<Deferred<Object>>(rows.size()) : null;

    ...一些限流计算，比如加载到的数据点数，字节数等等

      // If any filters have made it this far then we need to resolve
      // the row key UIDs to their names for string comparison. We'll
      // try to avoid the resolution with some sets but we may dupe
      // resolve a few times.
      // TODO - more efficient resolution
      // TODO - byte set instead of a string for the uid may be faster
      if (filters != null && !filters.isEmpty()) {
        lookups.clear();
        //执行查询的数据过滤，如果时间线已经被过滤了，那么就直接跳过，这里skips记录的就是那些被过滤的数据
        final String tsuid =
                UniqueId.uidToString(UniqueId.getTSUIDFromKey(key,
                        TSDB.metrics_width(), Const.TIMESTAMP_BYTES));
        if (skips.contains(tsuid)) {
          continue;
        }
        //keepers记录的是保留的数据。因为我们的查询输入的是tagk的name，不是二进制的编码，所以在这里需要把tagk的二进制编码重新解析为名字，然后用过滤器去判断是不是符合要求的数据，所以首先GetTagsCB，然后再调用MatchCB
        if (!keepers.contains(tsuid)) {
          final long uid_start = DateTime.nanoTime();

          /** CB to called after all of the UIDs have been resolved */
          class MatchCB implements Callback<Object, ArrayList<Boolean>> {
            @Override
            public Object call(final ArrayList<Boolean> matches)
                    throws Exception {
              for (final boolean matched : matches) {
                if (!matched) {
                  skips.add(tsuid);
                  return null;
                }
              }
              // matched all, good data
              keepers.add(tsuid);
              processRow(key, row);
              return null;
            }
          }

          /** Resolves all of the row key UIDs to their strings for filtering */
          class GetTagsCB implements
                  Callback<Deferred<ArrayList<Boolean>>, Map<String, String>> {
            @Override
            public Deferred<ArrayList<Boolean>> call(
                    final Map<String, String> tags) throws Exception {
              uid_resolve_time += (DateTime.nanoTime() - uid_start);
              uids_resolved += tags.size();
              final List<Deferred<Boolean>> matches =
                      new ArrayList<Deferred<Boolean>>(filters.size());

              for (final TagVFilter filter : filters) {
                //注意：这里一旦解析成功，就会调用filter进行判断，是否符合过滤条件，把true或者false保存到matches这个数组中。
                matches.add(filter.match(tags));
              }

              return Deferred.group(matches);
            }
          }

          lookups.add(Tags.getTagsAsync(tsdb, key)
                  .addCallbackDeferring(new GetTagsCB())
                  .addBoth(new MatchCB()));
        } else {
          processRow(key, row);
        }
      } else {
        processRow(key, row);
      }
    }

    // either we need to wait on the UID resolutions or we can go ahead
    // if we don't have filters.
    if (lookups != null && lookups.size() > 0) {
      class GroupCB implements Callback<Object, ArrayList<Object>> {
        @Override
        public Object call(final ArrayList<Object> group) throws Exception {
          return scan();
        }
      }
      return Deferred.group(lookups).addCallback(new GroupCB());
    } else {
      return scan();
    }
  } catch (final RuntimeException e) {
    LOG.error("Unexpected exception on scanner " + this, e);
    close(false);
    handleException(e);
    return null;
  }
}

void processRow(final byte[] key, final ArrayList<KeyValue> row) {
      ++rows_post_filter;
      if (delete) {
        final DeleteRequest del = new DeleteRequest(tsdb.dataTable(), key);
        tsdb.getClient().delete(del);
      }

        final KeyValue compacted;
        try {
          final List<Annotation> notes = Lists.newArrayList();
          compacted = tsdb.compact(row, notes, hists);
        if (compacted != null) { // Can be null if we ignored all KVs.
          kvs.add(compacted);
        }
      }
    }
```



从上面的call可以看出，数据的第一步处理逻辑：

1.针对scanner的数据，根据rowkey，调用Tags.getTagsAsync对每一个tagk/tagv进行uid到name的解析，保存到一个Map里面返回。

2.针对Map里面的每个tagk，调用过滤器的match函数，判断是否需要过滤

3.针对需要保留的数据，调用函数processRow，processRow主要执行两个逻辑：a：如果查询需要delete数据，那么就生成一个DeleteRequest，b：保存到kvs成员变量中；

4.如此一直处理，一直等到所有的scanner加载完毕。

#### 数据的合并

数据加载完毕后，会调用close函数.

25-27行：每个scanner返回的结果，都会存放到kv_map，key是countdown的返回值，这个只是一个临时标记，用来区分不同的scanner返回的数据，

```
void close(final boolean ok) {
    scanner.close();

    ...一些统计歇息...
    if (ok && exception == null) {
      validateAndTriggerCallback(kvs, annotations, histograms);
    } else {
      countdown.countDown();
    }
  }
}

/**
 * Called each time a scanner completes with valid or empty data.
 * @param kvs The compacted columns fetched by the scanner
 * @param annotations The annotations fetched by the scanners
 */
private void validateAndTriggerCallback(
        final List<KeyValue> kvs,
        final Map<byte[], List<Annotation>> annotations,
        final List<SimpleEntry<byte[], List<HistogramDataPoint>>> histograms) {

  countdown.countDown();
  final long count = countdown.getCount();
  if (kvs.size() > 0) {
    kv_map.put((int) count, kvs);
  }
  
  if (countdown.getCount() <= 0) {
    try {
      mergeAndReturnResults();
    } catch (final Exception ex) {
      results.callback(ex);
    }
  }
}
```

> net.opentsdb.core.SaltScanner#mergeAndReturnResults

```
private void mergeAndReturnResults() {
 ......
  //Merge sorted spans together
  if (!isHistogramScan()) {
    mergeDataPoints();
  } else {
    // Merge histogram data points
    mergeHistogramDataPoints();
  }
  .....
  if (!isHistogramScan()) {
    results.callback(spans);
  } else {
    histogramResults.callback(histSpans);
  }
}
```

上面最主要的代码有以下功能：

1。mergeDataPoints合并数据

2.调用CallBack。这里是和函数net.opentsdb.core.SaltScanner#scan前后呼应的。因为下面函数直接启动了scan之后就返回result了，这时候result啥都没有，知道数据处理完毕后，才调用callback，通知回调函数进行处理。

```
public Deferred<TreeMap<byte[], Span>> scan() {
  start_time = System.currentTimeMillis();
  int i = 0;
  for (final Scanner scanner: scanners) {
    new ScannerCB(scanner, i++).scan();
  }
  return results;
}
```

所以最主要的函数就是mergeDataPoints

注意如下代码：spans是一个TreeMap，使用了SpanCmp作为比较器。

```
private Deferred<TreeMap<byte[], Span>> findSpans() throws HBaseException {
  final short metric_width = tsdb.metrics.width();
  final TreeMap<byte[], Span> spans = // The key is a row key from HBase.
    new TreeMap<byte[], Span>(new SpanCmp(
        (short)(Const.SALT_WIDTH() + metric_width)));
  
    .......
    return new SaltScanner(tsdb, metric, scanners, spans, scanner_filters,
        delete, rollup_query, query_stats, query_index, null, 
        max_bytes, max_data_points,this.valueFilters).scan();
  
}
```



```
private void mergeDataPoints() {
    // Merge sorted spans together

      for (final KeyValue kv : kvs) {
        ...
        Span datapoints = spans.get(kv.key());
        if (datapoints == null) {
          datapoints = RollupQuery.isValidQuery(rollup_query) ?
                  new RollupSpan(tsdb, this.rollup_query) : new Span(tsdb);
          spans.put(kv.key(), datapoints);
        }

        try {
          datapoints.addRow(kv);
        } catch (RuntimeException e) {
          LOG.error("Exception adding row to span", e);
          throw e;
        }
    }
  }
  
```

其实这里很简单，就是把所有的KeyValue数据组装成一个一个的Span。Span会按照数据结构小节讨论的进行数据组装。注意spans的key是rowkey，就是spans里面存放了所有的rowkey，很多不同的rowkey（时间戳不一样），即使有不同的时间戳的数据，也只有第一条能加进去，剩下的都加入到Span里面去了！。Spans的keys是所有的时间线，而不是所有的rowkey，因为相同的时间线后续的时间点的数据是不能作为key加入进去的，只能加入到相应的作为value的span里面去！



至此，数据已经加载完成，并且完成了数据的逻辑上的组装。

所以数据流会回到findSpans，findSpans执行完毕后，results.callback(spans);，利用spans通知后续的回调；

而runAsync的代码里面n:result = findSpans().addCallback(new GroupByAndAggregateCB());

所以我们就得看GroupByAndAggregateCB怎么对数据进行GroupByAndAggregate：

#### 数据的汇总

> *GroupByAndAggregateCB*

```
/**
 * Callback that should be attached the the output of
 * {@link TsdbQuery#findSpans} to group and sort the results.
 */
private class GroupByAndAggregateCB implements 
  Callback<DataPoints[], TreeMap<byte[], Span>>{
  
  /**
  * Creates the {@link SpanGroup}s to form the final results of this query.
  * @param spans The {@link Span}s found for this query ({@link #findSpans}).
  * Can be {@code null}, in which case the array returned will be empty.
  * @return A possibly empty array of {@link SpanGroup}s built according to
  * any 'GROUP BY' formulated in this query.
  */
  @Override
  public DataPoints[] call(final TreeMap<byte[], Span> spans) throws Exception {
   ......
    情况1：聚合函数为none的情况；
    情况2：没有分组的情况;
    情况3：有groupby，根据grouby的tagv进行分组
}
```

汇总时，分了三种情况：

1.聚合函数为none

```
// The raw aggregator skips group bys and ignores downsampling
    //这里的注释是错误的，没有忽略降采样，即使aggregator为none，降采样也会带过去，降采样也会按照自己的汇总函数汇总的
    if (aggregator == Aggregators.NONE) {
      final SpanGroup[] groups = new SpanGroup[spans.size()];
      int i = 0;
      for (final Span span : spans.values()) {
        final SpanGroup group = new SpanGroup(
            tsdb, 
            getScanStartTimeSeconds(),
            getScanEndTimeSeconds(),
            null, //注意这里传的是NULL，如果不是null，会把所有的spans都加入到一个group里面去，而这里需要的是每个span一个group。
            rate, 
            rate_options,
            aggregator,
            downsampler,
            getStartTime(), 
            getEndTime(),
            query_index,
            rollup_query);
        group.add(span);
        groups[i++] = group;
      }
      return groups;
    }
```

2.没有分组，就是{}都是空的，这时候默认只有一个分组

```
if (group_bys == null) {
      // We haven't been asked to find groups, so let's put all the spans
      // together in the same group.
      final SpanGroup group = new SpanGroup(tsdb,
                                            getScanStartTimeSeconds(),
                                            getScanEndTimeSeconds(),
                                            spans.values(),
                                            rate, rate_options,
                                            aggregator,
                                            downsampler,
                                            getStartTime(), 
                                            getEndTime(),
                                            query_index,
                                            rollup_query);
      if (query_stats != null) {
        query_stats.addStat(query_index, QueryStat.GROUP_BY_TIME, 0);
      }
      return new SpanGroup[] { group };
    }
```

 情况3：有groupby，根据grouby的tagv进行分组

```
// Maps group value IDs to the SpanGroup for those values. Say we've
    // been asked to group by two things: foo=* bar=* Then the keys in this
    // map will contain all the value IDs combinations we've seen. If the
    // name IDs for `foo' and `bar' are respectively [0, 0, 7] and [0, 0, 2]
    // then we'll have group_bys=[[0, 0, 2], [0, 0, 7]] (notice it's sorted
    // by ID, so bar is first) and say we find foo=LOL bar=OMG as well as
    // foo=LOL bar=WTF and that the IDs of the tag values are:
    // LOL=[0, 0, 1] OMG=[0, 0, 4] WTF=[0, 0, 3]
    // then the map will have two keys:
    // - one for the LOL-OMG combination: [0, 0, 1, 0, 0, 4] and,
    // - one for the LOL-WTF combination: [0, 0, 1, 0, 0, 3].
    final ByteMap<SpanGroup> groups = new ByteMap<SpanGroup>();
    final short value_width = tsdb.tag_values.width();
    final byte[] group = new byte[group_bys.size() * value_width];
    for (final Map.Entry<byte[], Span> entry : spans.entrySet()) {
      //这里是每一行的rowkey
      final byte[] row = entry.getKey();
      byte[] value_id = null;
      int i = 0;
      // TODO(tsuna): The following loop has a quadratic behavior. We can
      // make it much better since both the row key and group_bys are sorted.
      for (final byte[] tag_id : group_bys) {
        value_id = Tags.getValueId(tsdb, row, tag_id);
        if (value_id == null) {
          break;
        }
        System.arraycopy(value_id, 0, group, i, value_width);
        i += value_width;
      }
      //经过上述的循环后，group里面已经存放的是只有参与汇总的tagv的uid了，利用这个group做key，
      //保存到groups里面去
      if (value_id == null) {
        LOG.error("WTF? Dropping span for row " + Arrays.toString(row)
                 + " as it had no matching tag from the requested groups,"
                 + " which is unexpected. Query=" + this);
        continue;
      }

  SpanGroup thegroup = groups.get(group);
  if (thegroup == null) {
    thegroup = new SpanGroup(tsdb, getScanStartTimeSeconds(),
                             getScanEndTimeSeconds(),
                             null, rate, rate_options, aggregator,
                             downsampler,
                             getStartTime(), 
                             getEndTime(),
                             query_index,
                             rollup_query);
    // Copy the array because we're going to keep `group' and overwrite
    // its contents. So we want the collection to have an immutable copy.
    final byte[] group_copy = new byte[group.length];
    System.arraycopy(group, 0, group_copy, 0, group.length);
    groups.put(group_copy, thegroup);
  }
  thegroup.add(entry.getValue());
}
if (query_stats != null) {
  query_stats.addStat(query_index, QueryStat.GROUP_BY_TIME, 0);
}
return groups.values().toArray(new SpanGroup[groups.size()]);
}
```

主要逻辑：

遍历spans（spans的keys是所有的符合条件的时间线！），实际上就是遍历所有的时间线，单独挑出时间线中属于groupby的tagv的值，把这些值弄到一个group变量里面去，利用这个group作为key，把属于一组的span加入到SpanGroup里面去！



这时候数据就组装完成了！

#### 数据的读取

最后一行，返回的DataPoints[]数组，实际上就是SpanGroup数组，这里会利用iterator进行遍历，所以就又回到前面的分析去了，分析SpanGroup,Span,RowSeq的iterator的实现，通过即时计算来获取需要的时间戳和值，

# 6.Compact的实现

## 压缩相关配置默认值;

```


default_map.put("tsd.storage.enable_compaction", "true");
default_map.put("tsd.storage.compaction.flush_interval", "10");
default_map.put("tsd.storage.compaction.min_flush_threshold", "100");
default_map.put("tsd.storage.compaction.max_concurrent_flushes", "10000");
default_map.put("tsd.storage.compaction.flush_speed", "2");
```




> CompactionQueue
>



## 基本流程：

CompactionQueue开启单独的线程进行压缩：

net.opentsdb.utils.Config





1.构造函数里面：Cmp(tsdb),比较器;根据timestamp比较元素

2.compcat只压缩过时的数据（至少是上个小时的）

3.出现OOM的时候queue会清空自己：）

4.每次compact之后会sleep flush_interval*1000毫秒，所以flush_speed至少为2，否则有可能compact不玩：）。我艹！

5.到底一次Flush多少元素？一般来讲，以时间为比例，假设我们10秒compact一次，那么一个小时的数据一共3600秒，那么我们应该flush 1/360*size的总数据，但是为了加快compact的速度，可以设置flush_speed，如果设置为2，那么就一次flush 2\*1/360=1/180\*size个。同时在这个数与min_flush_threshold取最大者，也就是每次至少100条。

6.这个线程是常驻的，如果发生了异常，会尽可能处理，否则重启一个线程，然后退出，

7.遍历rowkey，get出来，调用compact

```
final class CompactionQueue extends ConcurrentSkipListMap<byte[], Boolean> {


public CompactionQueue(final TSDB tsdb) {
    super(new Cmp(tsdb));
    this.tsdb = tsdb;
    metric_width = tsdb.metrics.width();
    flush_interval = tsdb.config.getInt("tsd.storage.compaction.flush_interval");
    min_flush_threshold = tsdb.config.getInt("tsd.storage.compaction.min_flush_threshold");
    max_concurrent_flushes = tsdb.config.getInt("tsd.storage.compaction.max_concurrent_flushes");
    flush_speed = tsdb.config.getInt("tsd.storage.compaction.flush_speed");

    //开启单独的线程
    if (tsdb.config.enable_compactions()) {
      startCompactionThread();
    }
  }  


  private void startCompactionThread() {
    final Thrd thread = new Thrd();
    thread.setDaemon(true);
    thread.start();
  }

final class Thrd extends Thread {
    public Thrd() {
      super("CompactionThread");
    }

    @Override
    public void run() {
      while (true) {
          ...
          final int size = size();
          if (size > min_flush_threshold) {
            //计算需要flush的数量
            final int maxflushes = Math.max(min_flush_threshold,
              size * flush_interval * flush_speed / Const.MAX_TIMESPAN);

            flush(now / 1000 - Const.MAX_TIMESPAN - 1, maxflushes);
       
          }
        ....
        try {
          Thread.sleep(flush_interval * 1000);
        } catch (InterruptedException e) {
          LOG.error("Compaction thread interrupted, doing one last flush", e);
          flush();
          return;
        }
      }
    }
  }
```



```
private Deferred<ArrayList<Object>> flush(final long cut_off, int maxflushes) {
	
  ......
  int seed = (int) (System.nanoTime() % 3);
	...
    final long base_time = Bytes.getUnsignedInt(row,
        Const.SALT_WIDTH() + metric_width);
    if (base_time > cut_off) {
     //当前小时的数据不compact
      break;
    }else{
        ......
    } 
    ds.add(tsdb.get(row).addCallbacks(compactcb, handle_read_error));
  }
  
  final Deferred<ArrayList<Object>> group = Deferred.group(ds);
  
  //循环Flush.....
  if (nflushes == max_concurrent_flushes && maxflushes > 0) {
    // We're not done yet.  Once this group of flushes completes, we need
    // to kick off more.
    tsdb.getClient().flush();  // Speed up this batch by telling the client to flush.
    final int maxflushez = maxflushes;  // Make it final for closure.
    final class FlushMoreCB implements Callback<Deferred<ArrayList<Object>>,
                                                ArrayList<Object>> {
      @Override
      public Deferred<ArrayList<Object>> call(final ArrayList<Object> arg) {
        return flush(cut_off, maxflushez);
      }
      @Override
      public String toString() {
        return "Continue flushing with cut_off=" + cut_off
          + ", maxflushes=" + maxflushez;
      }
    }
    group.addCallbackDeferring(new FlushMoreCB());
  }
  return group;
}
```

```
private final class CompactCB implements Callback<Object, ArrayList<KeyValue>> {
  @Override
  public Object call(final ArrayList<KeyValue> row) {
    return compact(row, null, null, null);
  }
  @Override
  public String toString() {
    return "compact";
  }
}
```

compact函数：

```
Deferred<Object> compact(final ArrayList<KeyValue> row,
    final KeyValue[] compacted,
    List<Annotation> annotations,
    List<HistogramDataPoint> histograms) {
  return new Compaction(row, compacted, annotations, histograms).compact();
}

```

## 数据压缩：





简单理解为：同一个小时的KeyValue，按照时间戳排好序，放到一个有衔接队列里面，不断的取出数据，合并到一个Qualifier和Vaule里面去。最后把Qualifier组装成Vaule写回到Hbase里面去。如果是秒和毫秒混合，也会设置一个flag。

net.opentsdb.core.CompactionQueue.Compaction#compact

qualifier_offset作为指针，遍历KeyValue里面的数据

> ColumnDatapointIterator

```
public ColumnDatapointIterator(final KeyValue kv) {
  this.column_timestamp = kv.timestamp();
  this.qualifier = kv.qualifier();
  this.value = kv.value();
  qualifier_offset = 0;
  value_offset = 0;
  checkForFixup();
  update();
}


private boolean update() {
    if (qualifier_offset >= qualifier.length || value_offset >= value.length) {
      return false;
    }
    if (Internal.inMilliseconds(qualifier[qualifier_offset])) {
      current_qual_length = 4;
      is_ms = true;
    } else {
      current_qual_length = 2;
      is_ms = false;
    }
    current_timestamp_offset = Internal.getOffsetFromQualifier(qualifier, qualifier_offset);
    current_val_length = Internal.getValueLengthFromQualifier(qualifier, qualifier_offset);
    return true;
  }
  
  比较器：比较Qualfier时间戳，如果时间戳相同，比较Cell的时间戳：）
    @Override
  public int compareTo(ColumnDatapointIterator o) {
    int c = current_timestamp_offset - o.current_timestamp_offset;
    if (c == 0) {
      // note inverse order of comparison!
      c = Long.signum(o.column_timestamp - column_timestamp);
    }
    return c;
  }
```

buildHeapProcessAnnotations

就是把一个个的KeyValue封装成ColumnDatapointIterator，放到一个优先级队列里面去；提供便利的数据访问功能和排序功能；

这里注意一点：compactedKVTimestamp，这里是把KeyValue里面最大的那个时间戳作为合并后的时间戳。

合并的逻辑：不断的从ColumnDatapointIterator取出数据，如果存在时间戳重复的，根据tsdb.config.use_max_value()来选择一个最大值或者最小值；

最主要的时候函数writeToBuffersFromOffset，一旦判断某个DataPoint需要合并，那么就把qual和value写到一个总的byte数组里面去；每个合适的都封装成一个BufferSegment，然后统一写到byte数组里面去。

最后生成一个KeyValue，写入到Hbase，并且删除已有的数据。



```
public Deferred<Object> compact() {
  // no columns in row, nothing to do
  if (nkvs == 0) {
    return null;
  }

  //1.数据排序
  compactedKVTimestamp = Long.MIN_VALUE;
  // go through all the columns, process annotations, and
  heap = new PriorityQueue<ColumnDatapointIterator>(nkvs);
  int tot_values = buildHeapProcessAnnotations();

  // if there are no datapoints or only one that needs no fixup, we are done
  if (noMergesOrFixups()) {
    // return the single non-annotation entry if requested
    if (compacted != null && heap.size() == 1) {
      compacted[0] = findFirstDatapointColumn();
    }
    return null;
  }

  // merge the datapoints, ordered by timestamp and removing duplicates
  final ByteBufferList compacted_qual = new ByteBufferList(tot_values);
  final ByteBufferList compacted_val = new ByteBufferList(tot_values);
  compaction_count.incrementAndGet();
  
  2.合并数据，合并为一个qual和value
  mergeDatapoints(compacted_qual, compacted_val);

  // if we wound up with no data in the compacted column, we are done
  if (compacted_qual.segmentCount() == 0) {
    return null;
  }

  // build the compacted columns
  final KeyValue compact = buildCompactedColumn(compacted_qual, compacted_val);

  final boolean write = updateDeletesCheckForWrite(compact);

  if (compacted != null) {  // Caller is interested in the compacted form.
    compacted[0] = compact;
    final long base_time = Bytes.getUnsignedInt(compact.key(),
        Const.SALT_WIDTH() + metric_width);
    final long cut_off = System.currentTimeMillis() / 1000
        - Const.MAX_TIMESPAN - 1;
    if (base_time > cut_off) {  // If row is too recent...
      return null;              // ... Don't write back compacted.
    }
  }
  // if compactions aren't enabled or there is nothing to write, we're done
  if (!tsdb.config.enable_compactions() || (!write && to_delete.isEmpty())) {
    return null;
  }

  final byte[] key = compact.key();
  //LOG.debug("Compacting row " + Arrays.toString(key));
  deleted_cells.addAndGet(to_delete.size());  // We're going to delete this.
  if (write) {
    written_cells.incrementAndGet();
    Deferred<Object> deferred = tsdb.put(key, compact.qualifier(), compact.value(), compactedKVTimestamp);
    if (!to_delete.isEmpty()) {
      deferred = deferred.addCallbacks(new DeleteCompactedCB(to_delete), handle_write_error);
    }
    return deferred;
  } else if (last_append_column == null) {
    // We had nothing to write, because one of the cells is already the
    // correctly compacted version, so we can go ahead and delete the
    // individual cells directly.
    new DeleteCompactedCB(to_delete).call(null);
    return null;
  } else {
    return null;
  }
}



private int buildHeapProcessAnnotations() {
  int tot_values = 0;

  for (final KeyValue kv : row) {
    byte[] qual = kv.qualifier();
    int len = qual.length;
    ......其他类型的KeyValue的处理
    // estimate number of points based on the size of the first entry
    // in the column; if ms/sec datapoints are mixed, this will be
    // incorrect, which will cost a reallocation/copy
    final int entry_size = Internal.inMilliseconds(qual) ? 4 : 2;
    tot_values += (len + entry_size - 1) / entry_size;
    if (longest == null || longest.qualifier().length < kv.qualifier().length) {
      longest = kv;
    }
    //封装KeyValue
    ColumnDatapointIterator col = new ColumnDatapointIterator(kv);
    compactedKVTimestamp = Math.max(compactedKVTimestamp, kv.timestamp());
    if (col.hasMoreData()) {
      heap.add(col);
    }
    to_delete.add(kv);
  }
  return tot_values;
}
```

```
dtcsMergeDataPoints和defaultMergeDataPoints两个处理的数据不同，一个是处理使用dp的时间戳作为hbase的时间戳，一个是使用系统时间作为时间戳。两者处理重复数据的逻辑不同，前者是根据配置使用最大值或者最小者；后者是根据是否修复配置，如果修复，则默认使用第一个读取到的值，应该也是随机的！

//dtsc:Data Transmission and Control System
private void dtcsMergeDataPoints(ByteBufferList compacted_qual, ByteBufferList compacted_val) {
  // Compare timestamps for two KeyValues at the same time,
  // if they are same compare their values
  // Return maximum or minimum value depending upon tsd.storage.use_max_value parameter
  // This function is called once for every RowKey,
  // so we only care about comparing offsets, which
  // are a part of column qualifier
  ColumnDatapointIterator col1 = null;
  ColumnDatapointIterator col2 = null;
  while (!heap.isEmpty()) {
    col1 = heap.remove();
    Pair<Integer, Integer> offsets = col1.getOffsets();
    Pair<Integer, Integer> offsetLengths = col1.getOffsetLengths();
    int ts1 = col1.getTimestampOffsetMs();
    double val1 = col1.getCellValueAsDouble();
    if (col1.advance()) {
      heap.add(col1);
    }
    int ts2 = ts1;
    while (ts1 == ts2) {
      col2 = heap.peek();
      ts2 = col2 != null ? col2.getTimestampOffsetMs() : ts2;
      if (col2 == null || ts1 != ts2)
        break;
      double val2 = col2.getCellValueAsDouble();
      if ((tsdb.config.use_max_value() && val2 > val1) || (!tsdb.config.use_max_value() && val1 > val2)) {
        // Reduce copying of byte arrays by just using col1 variable to reference to either max or min KeyValue
        col1 = col2;
        val1 = val2;
        offsets = col2.getOffsets();
        offsetLengths = col2.getOffsetLengths();
      }
      heap.remove();
      if (col2.advance()) {
        heap.add(col2);
      }
    }
    //？？我怎么觉得这里和writeToBuffers是一样的呢？
    col1.writeToBuffersFromOffset(compacted_qual, compacted_val, offsets, offsetLengths);
    ms_in_row |= col1.isMilliseconds();
    s_in_row |= !col1.isMilliseconds();
  }
}

分析：
-----------------------------
因为并不是col1对象使用col2的offset，而是col1=col2，再使用col2的offsets，所以感觉两个结果是一样的；
  public Pair<Integer, Integer> getOffsets() {
	return new Pair<Integer, Integer>(qualifier_offset, value_offset);
  }

  public Pair<Integer, Integer> getOffsetLengths() {
	return new Pair<Integer, Integer>(current_qual_length, current_val_length);
  }
    public void writeToBuffers(ByteBufferList compQualifier, ByteBufferList compValue) {
    compQualifier.add(qualifier, qualifier_offset, current_qual_length);
    compValue.add(value, value_offset, current_val_length);
  }

  public void writeToBuffersFromOffset(ByteBufferList compQualifier, ByteBufferList compValue, Pair<Integer, Integer> offsets, Pair<Integer, Integer> offsetLengths) {
    compQualifier.add(qualifier, offsets.getKey(), offsetLengths.getKey());
	compValue.add(value, offsets.getValue(), offsetLengths.getValue());
  }
  ------------------------------------------

private void defaultMergeDataPoints(ByteBufferList compacted_qual,
        ByteBufferList compacted_val) {
  int prevTs = -1;
  while (!heap.isEmpty()) {
    final ColumnDatapointIterator col = heap.remove();
    final int ts = col.getTimestampOffsetMs();
    if (ts == prevTs) {
      // check to see if it is a complete duplicate, or if the value changed
      final byte[] existingVal = compacted_val.getLastSegment();
      final byte[] discardedVal = col.getCopyOfCurrentValue();
      if (!Arrays.equals(existingVal, discardedVal)) {
        duplicates_different.incrementAndGet();
        if (!tsdb.config.fix_duplicates()) {
          throw new IllegalDataException("Duplicate timestamp for key="
              + Arrays.toString(row.get(0).key()) + ", ms_offset=" + ts + ", older="
              + Arrays.toString(existingVal) + ", newer=" + Arrays.toString(discardedVal)
              + "; set tsd.storage.fix_duplicates=true to fix automatically or run Fsck");
        }
        LOG.warn("Duplicate timestamp for key=" + Arrays.toString(row.get(0).key())
            + ", ms_offset=" + ts + ", kept=" + Arrays.toString(existingVal) + ", discarded="
            + Arrays.toString(discardedVal));
      } else {
        duplicates_same.incrementAndGet();
      }
    } else {
      prevTs = ts;
      col.writeToBuffers(compacted_qual, compacted_val);
      ms_in_row |= col.isMilliseconds();
      s_in_row |= !col.isMilliseconds();
    }
    if (col.advance()) {
      // there is still more data in this column, so add it back to the heap
      heap.add(col);
    }
  }
}


private KeyValue buildCompactedColumn(ByteBufferList compacted_qual,
        ByteBufferList compacted_val) {
      // metadata is a single byte for a multi-value column, otherwise nothing
      final int metadata_length = compacted_val.segmentCount() > 1 ? 1 : 0;
      final byte[] cq = compacted_qual.toBytes(0);
      final byte[] cv = compacted_val.toBytes(metadata_length);

      // add the metadata flag, which right now only includes whether we mix s/ms datapoints
      if (metadata_length > 0) {
        byte metadata_flag = 0;
        if (ms_in_row && s_in_row) {
          metadata_flag |= Const.MS_MIXED_COMPACT;
        }
        cv[cv.length - 1] = metadata_flag;
      }

      final KeyValue first = row.get(0);
      if(tsdb.getConfig().getBoolean("tsd.storage.use_otsdb_timestamp")) {
        return new KeyValue(first.key(), first.family(), cq, compactedKVTimestamp, cv);
      } else {
        return new KeyValue(first.key(), first.family(), cq, cv);
      }
    }
```



# 7.批量数据的导入

和put类似，但是

1.没有写meta

2.禁用了WAL，

所以这里如果使用HbaseReplicaiton作备份的话，还得注意一下这一点

# 8.Annonation:

## qualifier:

net.opentsdb.meta.Annotation#getGlobalAnnotations

```
if ((column.qualifier().length == 3 || column.qualifier().length == 5)
    && column.qualifier()[0] == PREFIX()) {
```



```
/** Byte used for the qualifier prefix to indicate this is an annotation */
private static final byte PREFIX = 0x01;
```

qualifier为3，或者为5，或者=0x01都是Annotation如果为秒，就是3个字节，如果为毫秒，就是5个字节。

net.opentsdb.meta.Annotation#getQualifier，秒一般是2个字节标识，毫秒是4个字节表示，第一个字节加上0x01，表示是Annotation

```
private static byte[] getQualifier(final long start_time) {
  if (start_time < 1) {
    throw new IllegalArgumentException("The start timestamp has not been set");
  }
  
  final long base_time;
  final byte[] qualifier;
  long timestamp = start_time;
  // downsample to seconds to save space AND prevent duplicates if the time
  // is on a second boundary (e.g. if someone posts at 1328140800 with value A 
  // and 1328140800000L with value B)
  if (timestamp % 1000 == 0) {
    timestamp = timestamp / 1000;
  }
  
  if ((timestamp & Const.SECOND_MASK) != 0) {
    // drop the ms timestamp to seconds to calculate the base timestamp
    base_time = ((timestamp / 1000) - 
        ((timestamp / 1000) % Const.MAX_TIMESPAN));
    qualifier = new byte[5];
    final int offset = (int) (timestamp - (base_time * 1000));
    System.arraycopy(Bytes.fromInt(offset), 0, qualifier, 1, 4);
  } else {
    base_time = (timestamp - (timestamp % Const.MAX_TIMESPAN));
    qualifier = new byte[3];
    final short offset = (short) (timestamp - base_time);
    System.arraycopy(Bytes.fromShort(offset), 0, qualifier, 1, 2);
  }
  qualifier[0] = PREFIX;
  return qualifier;
}
```



HTTP请求里面，可以是：

- /api/annotation/bulk

- /api/annotation

- /api/annotation

  第二个只根据开始时间获取 \0x000000 timestamp qualifier=offset的Annotation，new一个Get请求

  第三个如果没有结束时间，就把结束时间取系统时间，setStartKey和StopKey，取出所有的qualfier，在判断qualifier是不是注释，这样很浪费流量啊！（批量加载不能只加载某个qualifier？）

  ## 单条读取：

  ```
  public static Deferred<Annotation> getAnnotation(final TSDB tsdb, 
      final byte[] tsuid, final long start_time) {
    
    /**
     * Called after executing the GetRequest to parse the meta data.
     */
    final class GetCB implements Callback<Deferred<Annotation>, 
      ArrayList<KeyValue>> {
  
      /**
       * @return Null if the meta did not exist or a valid Annotation object if 
       * it did.
       */
      @Override
      public Deferred<Annotation> call(final ArrayList<KeyValue> row) 
        throws Exception {
        if (row == null || row.isEmpty()) {
          return Deferred.fromResult(null);
        }
        
        Annotation note = JSON.parseToObject(row.get(0).value(),
            Annotation.class);
        return Deferred.fromResult(note);
      }
      
    }
  
    final GetRequest get = new GetRequest(tsdb.dataTable(), 
        getRowKey(start_time, tsuid));
    get.family(FAMILY);
    get.qualifier(getQualifier(start_time));
    return tsdb.getClient().get(get).addCallbackDeferring(new GetCB());    
  }
  ```


## 批量读取

```
/**
   * Scanner that loops through the [0, 0, 0, timestamp] rows looking for
   * global annotations. Returns a list of parsed annotation objects.
   * The list may be empty.
   */
  final class ScannerCB implements Callback<Deferred<List<Annotation>>, 
    ArrayList<ArrayList<KeyValue>>> {
    final Scanner scanner;
    final ArrayList<Annotation> annotations = new ArrayList<Annotation>();
    
    /**
     * Initializes the scanner
     */
    public ScannerCB() {
      final byte[] start = new byte[Const.SALT_WIDTH() + 
                                    TSDB.metrics_width() + 
                                    Const.TIMESTAMP_BYTES];
      final byte[] end = new byte[Const.SALT_WIDTH() + 
                                  TSDB.metrics_width() + 
                                  Const.TIMESTAMP_BYTES];
      
      final long normalized_start = (start_time - 
          (start_time % Const.MAX_TIMESPAN));
      final long normalized_end = (end_time - 
          (end_time % Const.MAX_TIMESPAN) + Const.MAX_TIMESPAN);
      
      Bytes.setInt(start, (int) normalized_start, 
          Const.SALT_WIDTH() + TSDB.metrics_width());
      Bytes.setInt(end, (int) normalized_end, 
          Const.SALT_WIDTH() + TSDB.metrics_width());

      scanner = tsdb.getClient().newScanner(tsdb.dataTable());
      scanner.setStartKey(start);
      scanner.setStopKey(end);
      scanner.setFamily(FAMILY);
    }
    
    public Deferred<List<Annotation>> scan() {
      return scanner.nextRows().addCallbackDeferring(this);
    }
    
    @Override
    public Deferred<List<Annotation>> call (
        final ArrayList<ArrayList<KeyValue>> rows) throws Exception {
      if (rows == null || rows.isEmpty()) {
        return Deferred.fromResult((List<Annotation>)annotations);
      }
      
      for (final ArrayList<KeyValue> row : rows) {
        for (KeyValue column : row) {
          if ((column.qualifier().length == 3 || column.qualifier().length == 5) 
              && column.qualifier()[0] == PREFIX()) {
            Annotation note = JSON.parseToObject(column.value(),
                Annotation.class);
            if (note.start_time < start_time || note.end_time > end_time) {
              continue;
            }
            annotations.add(note);
          }
        }
      }
      
      return scan();
    }
    
  }

  return new ScannerCB().scan();
}
```

## 删除：

直接一个请求

```
public Deferred<Object> delete(final TSDB tsdb) {
  if (start_time < 1) {
    throw new IllegalArgumentException("The start timestamp has not been set");
  }
  
  final byte[] tsuid_byte = tsuid != null && !tsuid.isEmpty() ? 
      UniqueId.stringToUid(tsuid) : null;
  final DeleteRequest delete = new DeleteRequest(tsdb.dataTable(), 
      getRowKey(start_time, tsuid_byte), FAMILY, 
      getQualifier(start_time));
  return tsdb.getClient().delete(delete);
}
```

## 批量删除：

tsdb的批量操作都是先scan再一条一条delete

```
/**
   * Iterates through the scanner results in an asynchronous manner, returning
   * once the scanner returns a null result set.
   */
  final class ScannerCB implements Callback<Deferred<List<Deferred<Object>>>, 
      ArrayList<ArrayList<KeyValue>>> {
    final Scanner scanner;

    public ScannerCB() {
      scanner = tsdb.getClient().newScanner(tsdb.dataTable());
      scanner.setStartKey(start_row);
      scanner.setStopKey(end_row);
      scanner.setFamily(FAMILY);
      if (tsuid != null) {
        final List<String> tsuids = new ArrayList<String>(1);
        tsuids.add(UniqueId.uidToString(tsuid));
        Internal.createAndSetTSUIDFilter(scanner, tsuids);
      }
    }
    
    public Deferred<List<Deferred<Object>>> scan() {
      return scanner.nextRows().addCallbackDeferring(this);
    }
    
    @Override
    public Deferred<List<Deferred<Object>>> call (
        final ArrayList<ArrayList<KeyValue>> rows) throws Exception {
      if (rows == null || rows.isEmpty()) {
        return Deferred.fromResult(delete_requests);
      }
      
      for (final ArrayList<KeyValue> row : rows) {
        final long base_time = Internal.baseTime(tsdb, row.get(0).key());
        for (KeyValue column : row) {
          if ((column.qualifier().length == 3 || column.qualifier().length == 5)
              && column.qualifier()[0] == PREFIX()) {
            final long timestamp = timeFromQualifier(column.qualifier(), 
                base_time);
            if (timestamp < start_time || timestamp > end_time) {
              continue;
            }
            final DeleteRequest delete = new DeleteRequest(tsdb.dataTable(), 
                column.key(), FAMILY, column.qualifier());
            delete_requests.add(tsdb.getClient().delete(delete));
          }
        }
      }
      return scan();
    }
  }

  /** Called when the scanner is done. Delete requests may still be pending */
  final class ScannerDoneCB implements Callback<Deferred<ArrayList<Object>>, 
    List<Deferred<Object>>> {
    @Override
    public Deferred<ArrayList<Object>> call(final List<Deferred<Object>> deletes)
        throws Exception {
      return Deferred.group(delete_requests);
    }
  }
  
  /** Waits on the group of deferreds to complete before returning the count */
  final class GroupCB implements Callback<Deferred<Integer>, ArrayList<Object>> {
    @Override
    public Deferred<Integer> call(final ArrayList<Object> deletes)
        throws Exception {
      return Deferred.fromResult(deletes.size());
    }
  }
  
  Deferred<ArrayList<Object>> scanner_done = new ScannerCB().scan()
      .addCallbackDeferring(new ScannerDoneCB());
  return scanner_done.addCallbackDeferring(new GroupCB());
}
```



# 8.其他功能：

## 插件的编写：

2.4RC2 UniqueIdWhitelistFilter可以作为参考

Tsdb3.0 有很多插件，可以参考

## 限流

net.opentsdb.core.TSDB#TSDB(org.hbase.async.HBaseClient, net.opentsdb.utils.Config)

```
query_limits = new QueryLimitOverride(this);
```

待看。

其实Tsdb的限流还有很多细节，什么最大返回dps啊，最大字节数啊等



