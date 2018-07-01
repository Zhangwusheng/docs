[TOC]



# ZooKeeper相关操作

本文基于HBASE-1.3。Hbase-2.0已经改成了基于Curator。不过学习这段代码还是有好处的

Hbase使用zookeeper来进行各种操作，因此首先学习HBASE针对Zookeeper的封装。

Hbase针对Zookeeper提供了基本的操作封装，然后提供了目录监听功能，后面还实现了基于ZK的进程锁功能。



### **1.工具-延迟实例化**



#### **InstancePending**

------

包名：org.apache.hadoop.hbase.zookeeper

类名：InstancePending

首先查看这个类的说明：

```
/**
 * Placeholder of an instance which will be accessed by other threads
 * but is not yet created. Thread safe.
 */
```

这个类代表了某个对象的实例，这个实例可以在其他线程中进行访问，亦即这个类是线程安全的。在使用这个类的使用，这个类代表的对象可能还没有被创建。如果需要使用它所代表的对象，那么必须等待它所代表的对象创建成功后才能进行访问。后续我们会看到，如果我们在创建Zookpper的连接时，需要传入一个Watcher，有可能这个Watcher还没有准备好，但是Zookeeper已经建立连接需要进行回调了，这时候在回调函数里面必须等待这个对象初始化完成之后才能进行事件回调。这种思路值得借鉴。***（这里可以把不同的线程之间的异步回调变得同步！）***

下面我们学习这个对象是怎么实现这个功能的。

**首先**使用了一个Holder类来保存真正的对象，这个是句柄模式：

```
private static class InstanceHolder<T> {
  // The JLS ensures the visibility of a final field and its contents
  // unless they are exposed to another thread while the construction.
  final T instance;

  InstanceHolder(T instance) {
    this.instance = instance;
  }
}
```

```
/**
 * Returns the instance given by the method {@link #prepare}.
 * This is an interruptible blocking method
 * and the interruption flag will be set just before returning if any.
 */
T get() {
  InstanceHolder<T> instanceHolder;
  boolean interrupted = false;

  while ((instanceHolder = this.instanceHolder) == null) {
    try {
      pendingLatch.await();
    } catch (InterruptedException e) {
      interrupted = true;
    }
  }

  if (interrupted) {
    Thread.currentThread().interrupt();
  }
  return instanceHolder.instance;
}
```

当我们需要创建对象时，需要调用get方法：

```
/**
 * Returns the instance given by the method {@link #prepare}.
 * This is an interruptible blocking method
 * and the interruption flag will be set just before returning if any.
 */
T get() {
  InstanceHolder<T> instanceHolder;
  boolean interrupted = false;

  //注意这里对InterruptedException的处理。
  while ((instanceHolder = this.instanceHolder) == null) {
    try {
      pendingLatch.await();
    } catch (InterruptedException e) {
      interrupted = true;
    }
  }

  if (interrupted) {
    Thread.currentThread().interrupt();
  }
  return instanceHolder.instance;
}
```

调用get时，如果对象还没有初始化（没有调用prepare方法），这时候会阻塞在pendingLatch.await方法上，这里可以学习一下线程被打断的处理方法：即使被打断，也会继续判断是否为空。等待对象被填充后，才返回重置打断标志位。

当对象准备好后，调用prepare方法，此时会触发锁释放：

```
 * Associates the given instance for the method {@link #get}.
 * This method should be called once, and {@code instance} should be non-null.
 * This method is expected to call as soon as possible
 * because the method {@code get} is uninterruptibly blocked until this method is called.
 */
void prepare(T instance) {
  assert instance != null;
  instanceHolder = new InstanceHolder<T>(instance);
  pendingLatch.countDown();
}
```



### **2.重试机制**



#### **RetryConfig**

------

一般来讲，如果我们重试某种操作，会设置重试次数，每次重试之间的时间间隔，如果每次间隔的时间不等，那么我们还应该设置允许的最大睡眠时间等。

```
public static class RetryConfig {
  //最大重试次数
  private int maxAttempts;
  //睡眠时间
  private long sleepInterval;
  //最大睡眠时间（和后续睡眠策略有关）
  private long maxSleepTime;
  //睡眠时间单位
  private TimeUnit timeUnit;
  //睡眠策略
  private BackoffPolicy backoffPolicy;
    
}
```

Hbase提供的策略有三种：均时睡眠，指数睡眠（不限时长），指数睡眠（限制时长）

均时睡眠的策略是每次重试睡眠sleepInterval时间，代码很简单不详述：

```
public static class BackoffPolicy {
  public long getBackoffTime(RetryConfig config, int attempts) {
    return config.getSleepInterval();
  }
}

public static class ExponentialBackoffPolicy extends BackoffPolicy {
    @Override
    public long getBackoffTime(RetryConfig config, int attempts) {
      long backoffTime = (long) (config.getSleepInterval() * Math.pow(2, attempts));
      return backoffTime;
    }
  }

public static class ExponentialBackoffPolicyWithLimit extends ExponentialBackoffPolicy {
    @Override
    public long getBackoffTime(RetryConfig config, int attempts) {
      long backoffTime = super.getBackoffTime(config, attempts);
      //如果传入-1，则代表不限制睡眠时间
      return config.getMaxSleepTime() > 0 ? Math.min(backoffTime, config.getMaxSleepTime()) : backoffTime;
    }
  }
```

#### **RetryCounter**

------

如果只指定最大次数和时间间隔，那么默认使用***<u>指数睡眠</u>***策略

```
public RetryCounter(int maxAttempts, long sleepInterval, TimeUnit timeUnit) {
  this(new RetryConfig(maxAttempts, sleepInterval, -1, timeUnit, new ExponentialBackoffPolicy()));
}

public RetryCounter(RetryConfig retryConfig) {
  this.attempts = 0;
  this.retryConfig = retryConfig;
}
```

最主要是下面这个函数：

```
/**
 * Sleep for a back off time as supplied by the backoff policy, and increases the attempts
 * @throws InterruptedException
 */
public void sleepUntilNextRetry() throws InterruptedException {
  int attempts = getAttemptTimes();
  long sleepTime = retryConfig.backoffPolicy.getBackoffTime(retryConfig, attempts);
  if (LOG.isTraceEnabled()) {
    LOG.trace("Sleeping " + sleepTime + "ms before retry #" + attempts + "...");
  }
  retryConfig.getTimeUnit().sleep(sleepTime);
  useRetry();
}
```

逻辑很简单，得到曾经重试的次数，然后计算出应该睡眠的时间，然后将重试次数加1。



#### **RetryCounterFactory**

------

```
public RetryCounterFactory(int maxAttempts, int sleepIntervalMillis) {
  //-1代表不限制重试时间
  this(maxAttempts, sleepIntervalMillis, -1);
}

//此处可以加上最大重试
public RetryCounterFactory(int maxAttempts, int sleepIntervalMillis, int maxSleepTime) {
  this(new RetryConfig(
    maxAttempts,
    sleepIntervalMillis,
    maxSleepTime,
    TimeUnit.MILLISECONDS,
    new ExponentialBackoffPolicyWithLimit()));
}

public RetryCounterFactory(RetryConfig retryConfig) {
  this.retryConfig = retryConfig;
}

public RetryCounter create() {
  return new RetryCounter(retryConfig);
}
```

默认工厂返回的是不限制睡眠时间的重试器。

### 3**.ZooKeeper操作**

#### **RecoverableZooKeeper**

------

参见附件针对Zookeeper如何区分可恢复错误和不可恢复错误。会导致ZK句柄失效的错误都属于严重错误，和连接相关的（断开，失去连接，超时），属于可恢复错误。

这个类最主要的是三个变量

```
private ZooKeeper zk;   ----ZK句柄
private final RetryCounterFactory retryCounterFactory;  ----重试工厂，用于创建重试策略
// An identifier of this process in the cluster
private final String identifier;
private final byte[] id;
private Watcher watcher;       ----ZK Watcher
private int sessionTimeout;
private String quorumServers;
private final Random salter;
```

主要提供了对ZK节点的create,delete,getChildren,setData,getData等操作，在这些操作里，针对Recoverable errors提供了处理。所有的函数里面，基本都是调用zk接口进行处理，在异常处理里面对Recoverable 进行了重试，下面以delete为例：



```
public void delete(String path, int version)
throws InterruptedException, KeeperException {
  //分布式跟踪
  TraceScope traceScope = null;
  try {
    traceScope = Trace.startSpan("RecoverableZookeeper.delete");
    //获取重试器
    RetryCounter retryCounter = retryCounterFactory.create();
    boolean isRetry = false; // False for first attempt, true for all retries.
    while (true) {
      try {
        checkZk().delete(path, version);
        return;
      } catch (KeeperException e) {
        switch (e.code()) {
          //这里是每个动作进行的自己独有的错误码的处理逻辑
          case NONODE:
            if (isRetry) {
              LOG.debug("Node " + path + " already deleted. Assuming a " +
                  "previous attempt succeeded.");
              return;
            }
            LOG.debug("Node " + path + " already deleted, retry=" + isRetry);
            throw e;

          case CONNECTIONLOSS:
          case OPERATIONTIMEOUT:
            //这里是公共处理逻辑：都是针对失去连接和超时进行的
            retryOrThrow(retryCounter, e, "delete");
            break;

          default:
            throw e;
        }
      }
      retryCounter.sleepUntilNextRetry();
      isRetry = true;
    }
  } finally {
    if (traceScope != null) traceScope.close();
  }
}

  private void retryOrThrow(RetryCounter retryCounter, KeeperException e,
      String opName) throws KeeperException {
    LOG.debug("Possibly transient ZooKeeper, quorum=" + quorumServers + ", exception=" + e);
    if (!retryCounter.shouldRetry()) {
      LOG.error("ZooKeeper " + opName + " failed after "
        + retryCounter.getMaxAttempts() + " attempts");
      throw e;
    }
  }
```



| 函数                | 转有返回码                                             |
| ------------------- | ------------------------------------------------------ |
| exists              | 无                                                     |
| delete              | NONODE                                                 |
| getChildren         | 无                                                     |
| getData             | 无                                                     |
| setData             | BADVERSION                                             |
| getAcl/setAcl       | 无                                                     |
| createNonSequential | NODEEXISTS                                             |
| createSequential    | 会记录是否第一次创建，如果不是，则会查找自己的创建记录 |
| multi               | 无                                                     |

如上所述，createSequential的特殊处理里面，会在重试时判断是否是第一次创建，如果不是第一次，而是重试阶段，那么会在父目录下面查找，自己的是否创建成功了。查找的依据就是自己创建的路径的prefix

```
private String findPreviousSequentialNode(String path)
  throws KeeperException, InterruptedException {
  int lastSlashIdx = path.lastIndexOf('/');
  assert(lastSlashIdx != -1);
  String parent = path.substring(0, lastSlashIdx);
  String nodePrefix = path.substring(lastSlashIdx+1);

  List<String> nodes = checkZk().getChildren(parent, false);
  List<String> matching = filterByPrefix(nodes, nodePrefix);
  for (String node : matching) {
    String nodePath = parent + "/" + node;
    Stat stat = checkZk().exists(nodePath, false);
    if (stat != null) {
      return nodePath;
    }
  }
  return null;
}
```



#### **ZooKeeperWatcher**

------

可以理解为ZK的句柄类，此类主要提供外部与ZK的交互，理论上讲ZooKeeperWatcher只有一个，由ZooKeeperWatcher来维护与zk的连接，任何对zk事件感兴趣的外部用户，必须实现ZooKeeperListener来实现回调。（此方法值得学习）。

另外一个重要的功能，就是提供了获取HBASE各个组件的ZK节点的接口函数！

这个类主要维护了几个变量：

- [ ] ZK句柄，RecoverableZooKeeper的一个实例，以及与zk连接相关的参数，比如quorum，identifier，这些参数都是从Configuration类读取的
- [ ] listeners，事件回调集合
- [ ] Hbase各个节点的设置和维护。（从Configuration读取，有默认值）
- [ ] Abortable，可以理解为各类Application，当发生Unrecoverable errors（ZK句柄无效）时，调用Abortable的abort方法，后续可以注意到，HBase的主要的类，都实现了Abortable接口，从而对ZK的事件进行处理。

主要进行了如下逻辑：

- [ ] 连接zk

- [ ] 处理zk事件，通知监听器

下面分析如何建立连接：

```
public ZooKeeperWatcher(Configuration conf, String identifier,
    Abortable abortable, boolean canCreateBaseZNode)
throws IOException, ZooKeeperConnectionException {
  this.conf = conf;
  this.quorum = ZKConfig.getZKQuorumServersString(conf);
  this.prefix = identifier;
  // Identifier will get the sessionid appended later below down when we
  // handle the syncconnect event.
  this.identifier = identifier + "0x0";
  this.abortable = abortable;
  setNodeNames(conf);
  PendingWatcher pendingWatcher = new PendingWatcher();
  this.recoverableZooKeeper = ZKUtil.connect(conf, quorum, pendingWatcher, identifier);
  pendingWatcher.prepare(this);
  if (canCreateBaseZNode) {
    createBaseZNodes();
  }
}
```

建立连接时，首先建立recoverableZooKeeper，这个构造函数里面会建立zk的会话。同时注意这里使用了InstancePending，并且把this作为对象传进去。**这里把两个异步的事件通过get同步起来了**！高！

```
class PendingWatcher implements Watcher {
  private final InstancePending<Watcher> pending = new InstancePending<Watcher>();

  @Override
  public void process(WatchedEvent event) {
    pending.get().process(event);
  }

  /**
   * Associates the substantial watcher of processing events.
   * This method should be called once, and {@code watcher} should be non-null.
   * This method is expected to call as soon as possible
   * because the event processing, being invoked by the ZooKeeper event thread,
   * is uninterruptibly blocked until this method is called.
   */
  void prepare(Watcher watcher) {
    pending.prepare(watcher);
  }
}
```

事件处理：

```
@Override
public void process(WatchedEvent event) {
  LOG.debug(prefix("Received ZooKeeper Event, " +
      "type=" + event.getType() + ", " +
      "state=" + event.getState() + ", " +
      "path=" + event.getPath()));

  switch(event.getType()) {

    // If event type is NONE, this is a connection status change
    // None代表的是连接事件！
    case None: {
      connectionEvent(event);
      break;
    }

    // Otherwise pass along to the listeners

    case NodeCreated: {
      for(ZooKeeperListener listener : listeners) {
        listener.nodeCreated(event.getPath());
      }
      break;
    }

    case NodeDeleted: {
      for(ZooKeeperListener listener : listeners) {
        listener.nodeDeleted(event.getPath());
      }
      break;
    }

    case NodeDataChanged: {
      for(ZooKeeperListener listener : listeners) {
        listener.nodeDataChanged(event.getPath());
      }
      break;
    }

    case NodeChildrenChanged: {
      for(ZooKeeperListener listener : listeners) {
        listener.nodeChildrenChanged(event.getPath());
      }
      break;
    }
  }
}

private void connectionEvent(WatchedEvent event) {
    switch(event.getState()) {
      //连接建立的时候，根据会话ID生成一个标识符
      case SyncConnected:
        this.identifier = this.prefix + "-0x" +
          Long.toHexString(this.recoverableZooKeeper.getSessionId());
        // Update our identifier.  Otherwise ignore.
        LOG.debug(this.identifier + " connected");
        break;
	
	  这里不对？断掉连接的时候应该abort？
      // Abort the server if Disconnected or Expired
      case Disconnected:
        LOG.debug(prefix("Received Disconnected from ZooKeeper, ignoring"));
        break;
      
      //会话过期，应该Abort！
      case Expired:
        String msg = prefix(this.identifier + " received expired from " +
          "ZooKeeper, aborting");
        // TODO: One thought is to add call to ZooKeeperListener so say,
        // ZooKeeperNodeTracker can zero out its data values.
        if (this.abortable != null) {
          this.abortable.abort(msg, new KeeperException.SessionExpiredException());
        }
        break;

      case ConnectedReadOnly:
      case SaslAuthenticated:
      case AuthFailed:
        break;

      default:
        throw new IllegalStateException("Received event is not valid: " + event.getState());
    }
  }
```



总结一下：ZooKeeperWatcher代表了对ZK的连接，当ZK连接时，把自己包装在PendingWatcher里面，等待ZK回调，（如果还没有运行到下一行，ZK会阻塞调用，因为InstancePending 还没有设置成this，太巧妙了！）。把自己作为ZK的Watcher，进行事件回调，对可以Reconver的进行重试，***对Expired的进行Abort***！后续的每个APP都应该重点关注abort事件是怎么处理的！

#### **ZooKeeperListener**

```
/**
 * Base class for internal listeners of ZooKeeper events.
 *
 * The {@link ZooKeeperWatcher} for a process will execute the appropriate
 * methods of implementations of this class.  In order to receive events from
 * the watcher, every listener must register itself via {@link ZooKeeperWatcher#registerListener}.
 *
 * Subclasses need only override those methods in which they are interested.
 *
 * Note that the watcher will be blocked when invoking methods in listeners so
 * they must not be long-running.
 */
@InterfaceAudience.Private
public abstract class ZooKeeperListener {

  // Reference to the zk watcher which also contains configuration and constants
  protected ZooKeeperWatcher watcher;

  /**
   * Construct a ZooKeeper event listener.
   */
  public ZooKeeperListener(ZooKeeperWatcher watcher) {
    this.watcher = watcher;
  }

  /**
   * Called when a new node has been created.
   * @param path full path of the new node
   */
  public void nodeCreated(String path) {
    // no-op
  }

  /**
   * Called when a node has been deleted
   * @param path full path of the deleted node
   */
  public void nodeDeleted(String path) {
    // no-op
  }

  /**
   * Called when an existing node has changed data.
   * @param path full path of the updated node
   */
  public void nodeDataChanged(String path) {
    // no-op
  }

  /**
   * Called when an existing node has a child node added or removed.
   * @param path full path of the node whose children have changed
   */
  public void nodeChildrenChanged(String path) {
    // no-op
  }

  /**
   * @return The watcher associated with this listener
   */
  public ZooKeeperWatcher getWatcher() {
    return this.watcher;
  }
}
```

#### **ZooKeeperNodeTracker**

顾名思义，这个是对某个节点的内容和生命周期进行跟踪的，很显然，他应该是一个**ZooKeeperListener**，Watch某个节点，响应响应事件。

***This is the base class used by trackers in both the Master and RegionServers.***

```
@Override
public synchronized void nodeCreated(String path) {
  if (!path.equals(node)) return;
  try {
    byte [] data = ZKUtil.getDataAndWatch(watcher, node);
    if (data != null) {
      this.data = data;
      notifyAll();
    } else {
      nodeDeleted(path);
    }
  } catch(KeeperException e) {
    abortable.abort("Unexpected exception handling nodeCreated event", e);
  }
}

@Override
public synchronized void nodeDeleted(String path) {
  if(path.equals(node)) {
    try {
      if(ZKUtil.watchAndCheckExists(watcher, node)) {
        nodeCreated(path);
      } else {
        this.data = null;
      }
    } catch(KeeperException e) {
      abortable.abort("Unexpected exception handling nodeDeleted event", e);
    }
  }
}
```

上面两个函数比较有意思，删除的监听那里，判断一下是否被创建了，创建的那里判断是否被删除了！

可以借鉴一下，如何处理ZK的目录！



总结：***ZooKeeperNodeTracker只要出现了KeeperException，就会调用abortable.abort!***

结合**ZooKeeperWatcher**，可以看出，Zookeeper对Hbase影响有多严重！只要有异常，就会abort！





#### **DeletionListener**

DeletionListener使用了一个CountDownLatch类作为通知机制。这个在后面的进程锁的时候用到了，用来监听锁的释放，释放的时候CountDownLatch-1

```
public class DeletionListener extends ZooKeeperListener {

  private static final Log LOG = LogFactory.getLog(DeletionListener.class);

  private final String pathToWatch;
  private final CountDownLatch deletedLatch;

  private volatile Throwable exception;

  /**
   * Create a new instance of the deletion watcher.
   * @param zkWatcher ZookeeperWatcher instance
   * @param pathToWatch (Fully qualified) ZNode path that we are waiting to
   *                    be deleted.
   * @param deletedLatch Count down on this latch when deletion has occured.
   */
  public DeletionListener(ZooKeeperWatcher zkWatcher, String pathToWatch,
      CountDownLatch deletedLatch) {
    super(zkWatcher);
    this.pathToWatch = pathToWatch;
    this.deletedLatch = deletedLatch;
    exception = null;
  }

  /**
   * Check if an exception has occurred when re-setting the watch.
   * @return True if we were unable to re-set a watch on a ZNode due to
   *         an exception.
   */
  public boolean hasException() {
    return exception != null;
  }

  /**
   * Get the last exception which has occurred when re-setting the watch.
   * Use hasException() to check whether or not an exception has occurred.
   * @return The last exception observed when re-setting the watch.
   */
  public Throwable getException() {
    return exception;
  }

  //注意，这里也加入了目录是否被删除的判断
  @Override
  public void nodeDataChanged(String path) {
    if (!path.equals(pathToWatch)) {
      return;
    }
    try {
      //监听不成功，说明目录不存在
      if (!(ZKUtil.setWatchIfNodeExists(watcher, pathToWatch))) {
        deletedLatch.countDown();
      }
    } catch (KeeperException ex) {
      exception = ex;
      deletedLatch.countDown();
      LOG.error("Error when re-setting the watch on " + pathToWatch, ex);
    }
  }

  @Override
  public void nodeDeleted(String path) {
    if (!path.equals(pathToWatch)) {
      return;
    }
    if (LOG.isDebugEnabled()) {
      LOG.debug("Processing delete on " + pathToWatch);
    }
    deletedLatch.countDown();
  }
}
```

### 4.跨进程锁

------

Hbase提供了两个锁，一个是进程间的锁，一个是读写锁。接口如下：

#### **InterProcessLock**

**An interface for an application-specific lock**

```
public interface InterProcessLock {

  /**
   * Acquire the lock, waiting indefinitely until the lock is released or
   * the thread is interrupted.
   * @throws IOException If there is an unrecoverable error releasing the lock
   * @throws InterruptedException If current thread is interrupted while
   *                              waiting for the lock
   */
  void acquire() throws IOException, InterruptedException;

  /**
   * Acquire the lock within a wait time.
   * @param timeoutMs The maximum time (in milliseconds) to wait for the lock,
   *                  -1 to wait indefinitely
   * @return True if the lock was acquired, false if waiting time elapsed
   *         before the lock was acquired
   * @throws IOException If there is an unrecoverable error talking talking
   *                     (e.g., when talking to a lock service) when acquiring
   *                     the lock
   * @throws InterruptedException If the thread is interrupted while waiting to
   *                              acquire the lock
   */
  boolean tryAcquire(long timeoutMs)
  throws IOException, InterruptedException;

  /**
   * Release the lock.
   * @throws IOException If there is an unrecoverable error releasing the lock
   * @throws InterruptedException If the thread is interrupted while releasing
   *                              the lock
   */
  void release() throws IOException, InterruptedException;

  /**
   * If supported, attempts to reap all the locks of this type by forcefully
   * deleting the locks (both held and attempted) that have expired according
   * to the given timeout. Lock reaping is different than coordinated lock revocation
   * in that, there is no coordination, and the behavior is undefined if the
   * lock holder is still alive.
   * @throws IOException If there is an unrecoverable error reaping the locks
   */
  void reapExpiredLocks(long expireTimeoutMs) throws IOException;

  /**
   * If supported, attempts to reap all the locks of this type by forcefully
   * deleting the locks (both held and attempted). Lock reaping is different
   * than coordinated lock revocation in that, there is no coordination, and
   * the behavior is undefined if the lock holder is still alive.
   * Calling this should have the same affect as calling {@link #reapExpiredLocks(long)}
   * with timeout=0.
   * @throws IOException If there is an unrecoverable error reaping the locks
   */
  void reapAllLocks() throws IOException;

  /**
   * An interface for objects that process lock metadata.
   */
  interface MetadataHandler {

    /**
     * Called after lock metadata is successfully read from a distributed
     * lock service. This method may contain any procedures for, e.g.,
     * printing the metadata in a humanly-readable format.
     * @param metadata The metadata
     */
    void handleMetadata(byte[] metadata);
  }

  /**
   * Visits the locks (both held and attempted) of this type with the given
   * {@link MetadataHandler}.
   * @throws IOException If there is an unrecoverable error
   */
  void visitLocks(MetadataHandler handler) throws IOException;
}
```



#### **InterProcessReadWriteLock**

**An interface for a distributed reader-writer lock.**

```
/**
 * An interface for a distributed reader-writer lock.
 */
@InterfaceAudience.Private
public interface InterProcessReadWriteLock {

  /**
   * Obtain a read lock containing given metadata.
   * @param metadata Serialized lock metadata (this may contain information
   *                 such as the process owning the lock or the purpose for
   *                 which the lock was acquired).
   * @return An instantiated InterProcessLock instance
   */
  InterProcessLock readLock(byte[] metadata);

  /**
   * Obtain a write lock containing given metadata.
   * @param metadata Serialized lock metadata (this may contain information
   *                 such as the process owning the lock or the purpose for
   *                 which the lock was acquired).
   * @return An instantiated InterProcessLock instance
   */
  InterProcessLock writeLock(byte[] metadata);
}
```



#### **ZKInterProcessLockBase**

主要是利用了ZK的EPHEMERAL_SEQUENTIAL节点特性，选取ID最小的作为获取锁的进程。ZK自己生成的临时顺序目录的最后10位为顺序号，所以ZNodeComparator根据后10位进行比较。

- [ ] ZNodeComparator

```
protected static class ZNodeComparator implements Comparator<String> {

  public static final ZNodeComparator COMPARATOR = new ZNodeComparator();

  private ZNodeComparator() {
  }

  /** Parses sequenceId from the znode name. Zookeeper documentation
   * states: The sequence number is always fixed length of 10 digits, 0 padded
   */
  public static long getChildSequenceId(String childZNode) {
    Preconditions.checkNotNull(childZNode);
    assert childZNode.length() >= 10;
    String sequenceIdStr = childZNode.substring(childZNode.length() - 10);
    return Long.parseLong(sequenceIdStr);
  }

  @Override
  public int compare(String zNode1, String zNode2) {
    long seq1 = getChildSequenceId(zNode1);
    long seq2 = getChildSequenceId(zNode2);
    if (seq1 == seq2) {
      return 0;
    } else {
      return seq1 < seq2 ? -1 : 1;
    }
  }
}
```

- [ ] 锁句柄AcquiredLock

锁其实就是ZK的目录，哪个进程获取了锁，就用创建的目录代表，这里又加上了版本号，用来进行精确控制。学习！

```
/**
 * Represents information about a lock held by this thread.
 */
protected static class AcquiredLock {
  private final String path;
  private final int version;

  /**
   * Store information about a lock.
   * @param path The path to a lock's ZNode
   * @param version The current version of the lock's ZNode
   */
  public AcquiredLock(String path, int version) {
    this.path = path;
    this.version = version;
  }

  public String getPath() {
    return path;
  }

  public int getVersion() {
    return version;
  }

  @Override
  public String toString() {
    return "AcquiredLockInfo{" +
        "path='" + path + '\'' +
        ", version=" + version +
        '}';
  }
}
```



下面分析ZKInterProcessLockBase的主要功能：获取锁，释放锁：

锁是通过创建临时顺序节点实现的，因此封装成一个函数：

```
private String createLockZNode() throws KeeperException {
  try {
    //注意：这里在创建目录的时候会写入metadata作为数据
    return ZKUtil.createNodeIfNotExistsNoWatch(zkWatcher, fullyQualifiedZNode,
        metadata, CreateMode.EPHEMERAL_SEQUENTIAL);
  } catch (KeeperException.NoNodeException nne) {
    //create parents, retry
    ZKUtil.createWithParents(zkWatcher, parentLockNode);
    return createLockZNode();
  }
}
```

获取锁的思路是：首先创建临时自增目录，然后获取父目录下面所有的临时目录，截取出后10位ID进行排序，最小的获取锁，如果不是最小的，达到超时时间后，返回false，表示没有获取到锁，那就监听持有锁的进程对应的那个目录，（getLockPath是个抽象函数，需要由子类实现，此处的子类指的是ZKInterProcessReadLock和ZKInterProcessWriteLock）。针对要监听的目录，注册一个DeletionListener，监听到持有锁的进程释放锁后，继续尝试获取锁，如此反复，知道获取锁或者超时为止；实现很精妙！为DeletionListener鼓掌！此处DeletionListener用到了一个局部变量CountDownLatch，用来进行跨进程间的目录删除这个异步动作的同步对象，精妙！

```
@Override
public boolean tryAcquire(long timeoutMs)
throws IOException, InterruptedException {
  boolean hasTimeout = timeoutMs != -1;
  long waitUntilMs =
      hasTimeout ?EnvironmentEdgeManager.currentTime() + timeoutMs : -1;
  String createdZNode;
  try {
    //创建临时顺序节点
    createdZNode = createLockZNode();
  } catch (KeeperException ex) {
    throw new IOException("Failed to create znode: " + fullyQualifiedZNode, ex);
  }
  //不断尝试获取锁
  while (true) {
    List<String> children;
    try {
      //先把所有的子节点选出来
      children = ZKUtil.listChildrenNoWatch(zkWatcher, parentLockNode);
    } catch (KeeperException e) {
      LOG.error("Unexpected ZooKeeper error when listing children", e);
      throw new IOException("Unexpected ZooKeeper exception", e);
    }
    String pathToWatch;
    ////getLockPath这里是附录2SharedLock的第123步的实现
    //这里读锁和写锁返回的目录不同，所以用了抽象方法，一般是返回ID最小的节点
    //这里通过判断是否为null来判断是否获取到了锁。不是null，就意味着没有获取到锁，
    //就执行循环体里面的逻辑，如果获取到了锁，就跳出循环执行后面的逻辑。
    //参见抽象函数的说明：getLockPath，所以子类实现这个函数的话，必须是获取到了锁
    //就返回null，获取不到锁，就返回需要监听的那个锁的目录。
    if ((pathToWatch = getLockPath(createdZNode, children)) == null) {
      break;
    }
    //上面的一行返回的是已经获得锁的那个目录，我们需要监听的就是这个目录
    //用来监听获得锁的进程释放锁之后，把目录删掉。当deletedLatch为0时，
    //表示监听到了目录的删除，表示锁被释放了
    CountDownLatch deletedLatch = new CountDownLatch(1);
    String zkPathToWatch =
        ZKUtil.joinZNode(parentLockNode, pathToWatch);
    DeletionListener deletionListener =
        new DeletionListener(zkWatcher, zkPathToWatch, deletedLatch);
    zkWatcher.registerListener(deletionListener);
    //~~截止到目前为止，手续到位。
    try {
      //这里是附录2SharedLock的第四五步的实现。if这里判断为true执行的是第五步的逻辑，
      //就是锁已经被别人持有，就要设置监听，然后重新循环；如果返回false，说明目录不存在了
      //那么就继续循环，不用监听（因为目录已经不存在了）。
      if (ZKUtil.setWatchIfNodeExists(zkWatcher, zkPathToWatch)) {
        // Wait for the watcher to fire
        if (hasTimeout) {
          long remainingMs = waitUntilMs - EnvironmentEdgeManager.currentTime();
          //锁等待时间，没有获取到锁
          if (remainingMs < 0 ||
              !deletedLatch.await(remainingMs, TimeUnit.MILLISECONDS)) {
            LOG.warn("Unable to acquire the lock in " + timeoutMs +
                " milliseconds.");
            try {
              //删除自己的临时节点
              ZKUtil.deleteNode(zkWatcher, createdZNode);
            } catch (KeeperException e) {
              LOG.warn("Unable to remove ZNode " + createdZNode);
            }
            return false;
          }
        } else {
          //锁不超时的话，就一直等待。
          deletedLatch.await();
        }
        if (deletionListener.hasException()) {
          Throwable t = deletionListener.getException();
          throw new IOException("Exception in the watcher", t);
        }
      }
    } catch (KeeperException e) {
      throw new IOException("Unexpected ZooKeeper exception", e);
    } finally {
      zkWatcher.unregisterListener(deletionListener);
    }
  }
  
  //至此获取到了锁，更新锁信息
  updateAcquiredLock(createdZNode);
  LOG.debug("Acquired a lock for " + createdZNode);
  return true;
}


  /**
   * Determine based on a list of children under a ZNode, whether or not a
   * process which created a specified ZNode has obtained a lock. If a lock is
   * not obtained, return the path that we should watch awaiting its deletion.
   * Otherwise, return null.
   * This method is abstract as the logic for determining whether or not a
   * lock is obtained depends on the type of lock being implemented.
   * @param myZNode The ZNode created by the process attempting to acquire
   *                a lock
   * @param children List of all child ZNodes under the lock's parent ZNode
   * @return The path to watch, or null if myZNode can represent a correctly
   *         acquired lock.
   */
  protected abstract String getLockPath(String myZNode, List<String> children)
  throws IOException;
  

/**
   * Update state as to indicate that a lock is held
   * @param createdZNode The lock znode
   * @throws IOException If an unrecoverable ZooKeeper error occurs
   */
  protected void updateAcquiredLock(String createdZNode) throws IOException {
    Stat stat = new Stat();
    byte[] data = null;
    Exception ex = null;
    try {
      data = ZKUtil.getDataNoWatch(zkWatcher, createdZNode, stat);
    } catch (KeeperException e) {
      LOG.warn("Cannot getData for znode:" + createdZNode, e);
      ex = e;
    }
    if (data == null) {
      LOG.error("Can't acquire a lock on a non-existent node " + createdZNode);
      throw new IllegalStateException("ZNode " + createdZNode +
          "no longer exists!", ex);
    }
    //这个函数最主要的就是生成这个锁的句柄！！
    AcquiredLock newLock = new AcquiredLock(createdZNode, stat.getVersion());
    if (!acquiredLock.compareAndSet(null, newLock)) {
      LOG.error("The lock " + fullyQualifiedZNode +
          " has already been acquired by another process!");
      throw new IllegalStateException(fullyQualifiedZNode +
          " is held by another process");
    }
  }
```

至此，我们获得了锁，并且把锁句柄保存到了acquiredLock这个变量中。



所得释放，很简单，从具体获取到目录，删除即可：

```
@Override
public void release() throws IOException, InterruptedException {
  AcquiredLock lock = acquiredLock.get();
  if (lock == null) {
    LOG.error("Cannot release lock" +
        ", process does not have a lock for " + fullyQualifiedZNode);
    throw new IllegalStateException("No lock held for " + fullyQualifiedZNode);
  }
  try {
    //目录必须存在
    if (ZKUtil.checkExists(zkWatcher, lock.getPath()) != -1) {
      //删除目录，必须符合指定的版本号
      boolean ret = ZKUtil.deleteNode(zkWatcher, lock.getPath(), lock.getVersion());
      if (!ret && ZKUtil.checkExists(zkWatcher, lock.getPath()) != -1) {
        throw new IllegalStateException("Couldn't delete " + lock.getPath());
      }
      if (!acquiredLock.compareAndSet(lock, null)) {
        LOG.debug("Current process no longer holds " + lock + " for " +
            fullyQualifiedZNode);
        throw new IllegalStateException("Not holding a lock for " +
            fullyQualifiedZNode +"!");
      }
    }
    if (LOG.isDebugEnabled()) {
      LOG.debug("Released " + lock.getPath());
    }
  } catch (BadVersionException e) {
    throw new IllegalStateException(e);
  } catch (KeeperException e) {
    throw new IOException(e);
  }
}
```

后面的子类**ZKInterProcessReadLock**会用到

```
//判断路径是否以write-开头
protected static boolean isChildWriteLock(String child) {
  int idx = child.lastIndexOf(ZKUtil.ZNODE_PATH_SEPARATOR);
  String suffix = child.substring(idx + 1);
  return suffix.startsWith(WRITE_LOCK_CHILD_NODE_PREFIX);
}
```

#### **ZKInterProcessReadLock**

```
public class ZKInterProcessReadLock extends ZKInterProcessLockBase {

  private static final Log LOG = LogFactory.getLog(ZKInterProcessReadLock.class);

  public ZKInterProcessReadLock(ZooKeeperWatcher zooKeeperWatcher,
      String znode, byte[] metadata, MetadataHandler handler) {
    super(zooKeeperWatcher, znode, metadata, handler, READ_LOCK_CHILD_NODE_PREFIX);
  }

  /**
   * {@inheritDoc}
   */
  @Override
  protected String getLockPath(String createdZNode, List<String> children) throws IOException {
    TreeSet<String> writeChildren =
        new TreeSet<String>(ZNodeComparator.COMPARATOR);
    for (String child : children) {
      if (isChildWriteLock(child)) {
        writeChildren.add(child);
      }
    }
    //如果没有写锁，就返回空？看父类的注释：
   * Determine based on a list of children under a ZNode, whether or not a
   * process which created a specified ZNode has obtained a lock. If a lock is
   * not obtained, return the path that we should watch awaiting its deletion.
   * Otherwise, return null.
   看上面解释，如果没有获得锁，那么返回我们应该监听的目录，否则返回空
   //为什么读锁，这里没有写就是空呢？是因为读锁人人都可以获取到！？
   //参见 附录2的SharedLock
   //3.If there are no children with a pathname starting with "write-" and having a lower 
   //sequence number than the node created in step 1, the client has the lock and 
   //can exit the protocol.
   
    if (writeChildren.isEmpty()) {
      return null;
    }
    //附录2的SharedLock第四步的实现
    //4.Otherwise, call exists( ), with watch flag, set on the node in lock directory
    //with pathname staring with "write-" having the next lowest sequence number.
    //如果有写锁，就不能获取到读锁，只能等待写锁释放！
    SortedSet<String> lowerChildren = writeChildren.headSet(createdZNode);
    if (lowerChildren.isEmpty()) {
      //如果写锁没有比读锁更早的，那么还是可以获取读锁？这里如果不为空，说明有写锁比读锁
      //早，所以必须等待写锁释放，就是要返回写锁的目录。
      return null;
    }
    //没有获得锁，返回自己应该监听的那个。这里不会有惊群效应，是因为不同的进程创建的目录最后的id
    //是不一样的，那么取得的最接近的比他小的那个节点都是不一样的，因此最终只有一个进程会获取锁。
    String pathToWatch = lowerChildren.last();
    String nodeHoldingLock = lowerChildren.first();
    String znode = ZKUtil.joinZNode(parentLockNode, nodeHoldingLock);
    handleLockMetadata(znode);

    return pathToWatch;
  }
}
```

#### **ZKInterProcessWriteLock**

```
public class ZKInterProcessWriteLock extends ZKInterProcessLockBase {

  private static final Log LOG = LogFactory.getLog(ZKInterProcessWriteLock.class);

  public ZKInterProcessWriteLock(ZooKeeperWatcher zooKeeperWatcher,
      String znode, byte[] metadata, MetadataHandler handler) {
    super(zooKeeperWatcher, znode, metadata, handler, WRITE_LOCK_CHILD_NODE_PREFIX);
  }

  /**
   * {@inheritDoc}
   */
  @Override
  protected String getLockPath(String createdZNode, List<String> children) throws IOException {
    TreeSet<String> sortedChildren =
        new TreeSet<String>(ZNodeComparator.COMPARATOR);
    sortedChildren.addAll(children);
    //比自己小的那个，如果没有，意味着获取到了锁
    String pathToWatch = sortedChildren.lower(createdZNode);
    if (pathToWatch != null) {
      //如果有比自己小的，那么获取最小的那个，然后监听最小的目录。这是写锁的逻辑。
      String nodeHoldingLock = sortedChildren.first();
      String znode = ZKUtil.joinZNode(parentLockNode, nodeHoldingLock);
      handleLockMetadata(znode);
    }
    return pathToWatch;
  }
}
```

#### **ZKInterProcessReadWriteLock**

这里最主要的就是，读锁和写锁使用了同一个目录，从而在读锁那里，可以判断出如果有写锁，就等待写锁释放。

```
public class ZKInterProcessReadWriteLock implements InterProcessReadWriteLock {

  private final ZooKeeperWatcher zkWatcher;
  private final String znode;
  private final MetadataHandler handler;

  /**
   * Creates a DistributedReadWriteLock instance.
   * @param zkWatcher
   * @param znode ZNode path for the lock
   * @param handler An object that will handle de-serializing and processing
   *                the metadata associated with reader or writer locks
   *                created by this object or null if none desired.
   */
  public ZKInterProcessReadWriteLock(ZooKeeperWatcher zkWatcher, String znode,
      MetadataHandler handler) {
    this.zkWatcher = zkWatcher;
    this.znode = znode;
    this.handler = handler;
  }

  /**
   * {@inheritDoc}
   */
  public ZKInterProcessReadLock readLock(byte[] metadata) {
    return new ZKInterProcessReadLock(zkWatcher, znode, metadata, handler);
  }

  /**
   * {@inheritDoc}
   */
  public ZKInterProcessWriteLock writeLock(byte[] metadata) {
    return new ZKInterProcessWriteLock(zkWatcher, znode, metadata, handler);
  }
}
```

### 附件：ZooKeeper 相关知识

#### 附1：Handling the errors a ZooKeeper throws at you

https://wiki.apache.org/hadoop/ZooKeeper/ErrorHandling

There are plenty of things that can go wrong when dealing with ZooKeeper. Application code and libraries need to deal with them. This document suggests best practices to deal with errors. ***<u>We distinguish an application from library code because libraries have a limited view of the overall picture, while the application has a complete view of what is going on.(这段话很好)</u>*** For example, the Application may use a Lock object from one library and a KeptSet object from another library. The Application knows that it must only do change operations on KeptSet when the Lock is held. If there is a fatal error that happens to the ZooKeeper handle, such as session expiration, only the Application knows the necessary steps to recover from the error; the Lock and KeptSet libraries should not try to recover.

When things do go wrong they are manifest as an exception or an error code. These errors can be organized into the following categories:

- ***Normal state exceptions:** trying to create a znode that already exists, calling setData on an existing znode, doing a conditional write to a znode that has an unexpected version number, etc.*
- ***Recoverable errors***: the disconnected event, connection timed out, and the connection loss exception are examples of recoverable errors, they indicate a problem that happened, but the ZooKeeper handle is still valid and future operations will succeed once the ZooKeeper library can reestablish its connection to ZooKeeper.（**断开连接，连接超时，失去连接这些是可以恢复的**）
- ***Fatal errors***: the ZooKeeper handle has become invalid. This can be due to an explicit close, authentication errors, or session expiration.（**ZK句柄无效属于严重错误，会话过期属于严重错误！不是可恢复的错误，因为他会导致ZK句柄无效。**）

The application and libraries handle the normal state exceptions as they happen. Usually they are a expected part of normal operation. For example, when doing a conditional set, usually the programmer is aware that there may be a concurrent process that may also try the same set and knows how to deal with it. The other two categories of errors can be more complicated.

##### Recoverable errors

Recoverable errors are passed back to the application because ZooKeeper itself cannot recover from them. **The ZooKeeper library does try to recover the connection, so <u>the handle should not be closed</u> on a recoverable error**, **but the application must deal with the transient error**. "But wait", you say, "if I'm doing a getData(), can't ZooKeeper just reissue it for me". Yes, of course ZooKeeper could as long as you were just doing a getData(). What if you were doing a create() or a delete() or a conditional setData()? When a ZooKeeper client loses a connection to the ZooKeeper server there may be some requests in flight; we don't know where they were in their flight at the time of the connection loss.

For example the create we sent just before the loss may not have made it out of the network stack, or it may have made it to the ZooKeeper server we were connected to and been forwarded on to other servers before our server died. So, when we reestablish the connection to the ZooKeeper service we have no good way to know if our create executed or not. (The server actually has the needed information, but there is a lot of implementation work that needs to happen to take advantage of that information. Ironically, once we make the mutating requests re-issuable, the read requests become problematic...) So, if we reissue a create() request and we get a NodeExistsException, the ZooKeeper client doesn't know if the exception resulted because the previous request went through or someone else did the create.

To handle recoverable errors, developers need to realize that there are two classes of requests: **idempotent and non-idempotent requests.** **Read requests and unconditional sets and deletes are examples of idempotent requests**, they can be reissued with the same results. (Although, the delete may throw a NoNodeException on reissue its effect on the ZooKeeper state is the same.) Non-idempotent requests need special handling, application and library writers need to keep in mind that they may need to encode information in the data or name of znodes to detect when a retries. A simple example is a create that uses a sequence flag. If a process issues a create("/x-", ..., SEQUENCE) and gets a connection loss exception, that process will reissue another create("/x-", ..., SEQUENCE) and get back x-111. When the process does a getChildren("/"), it sees x-1,x-30,x-109,x-110,x-111, now it could be that x-109 was the result of the previous create, so the process actually owns both x-109 and x-111. An easy way around this is to use "x-process id-" when doing the create. If the process is using an id of 352, before reissuing the create it will do a getChildren("/") and see "x-222-1", "x-542-30", "x-352-109", x-333-110". The process will know that the original create succeeded an the znode it created is "x-352-109".

The bottom line is that ZooKeepers aren't lazy. They don't throw up their hands when there is a problem just because they don't want to bother retrying. There is application specific logic that is needed for recovery. ZooKeeper doesn't even retry non-idempotent requests because it may violate ordering guarantees that it provides to the clients. Thus, application programs should be very skeptical of layers build on top of ZooKeeper that simply reissue requests when these kinds of errors arise.

##### Unrecoverable errors

A ZooKeeper handle becomes invalid due to an explicit close or due to a catastrophic error such as a session expiration; either way any ephemeral nodes associated with the ZooKeeper handle will go away. An application can deal with an invalid handle, but libraries cannot. An application knows the steps it took to set things up properly and can re-execute those steps; a library sees only a subset of those steps without seeing the order. For these reasons it is important that libraries do not try to recover from unrecoverable errors; they do not have the whole picture and, just like ZooKeeper, do not have sufficient knowledge of the application to recover automatically.

When a library gets an unrecoverable error it should shutdown as gracefully as it can. Obviously it is not going to be able to interact with ZooKeeper, but it can mark its state as invalid and clean up any internal data structures.

**An application that gets an unrecoverable error needs to make sure that everything that relied on the previous ZooKeeper handle is shutdown properly and then go through the process to bring everything back up**. For this to work properly libraries must be not be doing magic under the covers. If we go back to our original example of the KeptSet protected by the Lock. The KeptSet implementer may think "hey I don't have ephemeral nodes, I can recover from session expirations by just creating a new Zookeeper object". Yes, this is true for the library, but the application is now screwed: most applications now days are multi threaded; the application gets the Lock and threads start accessing the KeptSet; if the Lock is lost due to a session expiration and the KeptSet magically keeps working bad things are going to happen. You may say that the threads should shutdown when the lock is lost, but there is a race condition between getting the expiration and shutting down the threads and the KeptSet automagically recovering.

The bottom line is that libraries that recover from unrecoverable errors should be use with extreme care, if used at all.

#### 附2：Locks

http://zookeeper.apache.org/doc/current/recipes.html#sc_recipes_Locks

Fully distributed locks that are globally synchronous, meaning at any snapshot in time no two clients think they hold the same lock. These can be implemented using ZooKeeeper. As with priority queues, first define a lock node.

Note

There now exists a Lock implementation in ZooKeeper recipes directory. This is distributed with the release -- src/recipes/lock directory of the release artifact.

Clients wishing to obtain a lock do the following:

1. Call **create( )** with a pathname of "_locknode_/lock-" and the *sequence* and *ephemeral* flags set.
2. Call **getChildren( )** on the lock node *without* setting the watch flag (this is important to avoid the herd effect).
3. If the pathname created in step **1** has the lowest sequence number suffix, the client has the lock and the client exits the protocol.
4. The client calls **exists( )** with the watch flag set on the path in the lock directory with the next lowest sequence number.
5. if **exists( )** returns false, go to step **2**. Otherwise, wait for a notification for the pathname from the previous step before going to step **2**.

The unlock protocol is very simple: clients wishing to release a lock simply delete the node they created in step 1.

Here are a few things to notice:

- The removal of a node will only cause one client to wake up since each node is watched by exactly one client. In this way, you avoid the herd effect.

- There is no polling or timeouts.

- Because of the way you implement locking, it is easy to see the amount of lock contention, break locks, debug locking problems, etc.

#### Shared Locks

You can implement shared locks by with a few changes to the lock protocol:

**Obtaining a read lock:** 

1. Call **create( )** to create a node with pathname "_locknode_/read-". This is the lock node use later in the protocol. Make sure to set both the *sequence* and*ephemeral* flags.
2. Call **getChildren( )** on the lock node *without* setting the *watch* flag - this is important, as it avoids the herd effect.
3. If there are no children with a pathname starting with "write-" and having a lower sequence number than the node created in step **1**, the client has the lock and can exit the protocol.
4. Otherwise, call **exists( )**, with *watch* flag, set on the node in lock directory with pathname staring with "write-" having the next lowest sequence number.
5. If **exists( )** returns *false*, goto step **2**.
6. Otherwise, wait for a notification for the pathname from the previous step before going to step **2**



**Obtaining a write lock:**

1. Call **create( )** to create a node with pathname "_locknode_/write-". This is the lock node spoken of later in the protocol. Make sure to set both *sequence*and *ephemeral* flags.

2. Call **getChildren( )** on the lock node *without* setting the *watch* flag - this is important, as it avoids the herd effect.

3. If there are no children with a lower sequence number than the node created in step **1**, the client has the lock and the client exits the protocol.

4. Call **exists( ),** with *watch* flag set, on the node with the pathname that has the next lowest sequence number.

5. If **exists( )** returns *false*, goto step **2**. Otherwise, wait for a notification for the pathname from the previous step before going to step **2**.

   

Note

It might appear that this recipe creates a herd effect: when there is a large group of clients waiting for a read lock, and all getting notified more or less simultaneously when the "write-" node with the lowest sequence number is deleted. In fact. that's valid behavior: as all those waiting reader clients should be released since they have the lock. The herd effect refers to releasing a "herd" when in fact only a single or a small number of machines can proceed.

#### Recoverable Shared Locks

With minor modifications to the Shared Lock protocol, you make shared locks revocable by modifying the shared lock protocol:

In step **1**, of both obtain reader and writer lock protocols, call **getData( )** with *watch* set, immediately after the call to **create( )**. If the client subsequently receives notification for the node it created in step **1**, it does another **getData( )** on that node, with *watch* set and looks for the string "unlock", which signals to the client that it must release the lock. This is because, according to this shared lock protocol, you can request the client with the lock give up the lock by calling **setData()** on the lock node, writing "unlock" to that node.

Note that this protocol requires the lock holder to consent to releasing the lock. Such consent is important, especially if the lock holder needs to do some processing before releasing the lock. Of course you can always implement *Revocable Shared Locks with Freaking Laser Beams* by stipulating in your protocol that the revoker is allowed to delete the lock node if after some length of time the lock isn't deleted by the lock holder.