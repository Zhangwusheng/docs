## 查询的执行顺序

- Filtering
- Grouping
- Downsampling
- Interpolation
- Aggregation
- Rate Conversion
- Functions
- Expressions



## 查询涉及到的对象以及包含关系

HttpQUery------>^1^TSQuery

## 查询对象

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



## 子查询对象

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

## 过滤器

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

## 降采样规格



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





### 序列化后的校验

> net.opentsdb.core.TSSubQuery#validateAndSetQuery

aggregator反序列化后为字符串，需要转换为对象，downsample也是

```
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



### 查询第一步：名称到编码



### 查询第二步：



### 查询第三步：



