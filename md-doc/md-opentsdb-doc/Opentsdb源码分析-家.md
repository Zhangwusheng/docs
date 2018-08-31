# 1.写流程



# 2.Qualifer的相关操作：

1. Qualifer的生成：
2. 比较

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

3. 获取数据长度（数据的长度保存在Qualifier里面）

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

3.提供HTTPGet请求的解析

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



TSQuery.start_time表示查询的开始时间，以毫秒为单位

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

4.降采样处理逻辑？数据过滤处理逻辑？汇总处理逻辑？

5.数据不保存计算结果，而是采用了懒加载的方式，在用到时才进行计算。这是通过各种iterator来实现的，而每个iterator都实现了datapoint接口，datapoint结果包含数据最基本的元素：时间戳，值。

#### 计算涉及到的数据结构：

##### DataPoint

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

##### DataPoints

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



##### SeekableView

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



##### RowSeq

实现了DataPoints接口，3.0是直接实现了DataPoints接口；

```
public final class RowSeq implements iRowSeq {
}
public interface iRowSeq extends DataPoints {
}
```

- 比较器：


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



- ##### RowSeq疑问：

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

##### Span

#### Span的查找





# 6.Compact的实现

# 7.元数据的写入

# 8.批量数据的导入

# 9.其他功能：

## 限流

net.opentsdb.core.TSDB#TSDB(org.hbase.async.HBaseClient, net.opentsdb.utils.Config)

```
query_limits = new QueryLimitOverride(this);
```

## 

