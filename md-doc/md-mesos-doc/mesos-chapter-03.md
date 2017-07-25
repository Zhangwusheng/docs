#第三章. Libprocess分析

libprocess是mesos的RPC框架，实现了actor通信模式。

主要功能点是：

1. 网络IO，使用的是libev异步框架 
2. 消息传递和解析，主要是Http协议
3. 事件的派发和处理。内存消息队列

工具类：
首先需要分析的是Future和Promise类，这两个类是最为重要的基础类。

**Future**      < include/process/future.hpp >


 ~~~cpp
 // Definition of a "shared" future. A future can hold any
// copy-constructible value. A future is considered "shared" because
// by default a future can be accessed concurrently.
template <typename T>
class Future
{
public:
  // Constructs a failed future.
  static Future<T> failed(const std::string& message);

  
  ...构造函数列表，略过
  
  // Helpers to get the current state of this future.
  bool isPending() const;
  bool isReady() const;
  bool isDiscarded() const;
  bool isFailed() const;
  bool hasDiscard() const;


  // Waits for this future to become ready, discarded, or failed.
  // 默认无限等待
  bool await(const Duration& duration = Seconds(-1)) const;

  // Return the value associated with this future, waits indefinitely
  // until a value gets associated or until the future is discarded.
  const T& get() const;
  const T* operator->() const;
  
  // 回调相关
  // Type of the callback functions that can get invoked when the
  // future gets set, fails, or is discarded.
  typedef lambda::function<void()> DiscardCallback;
  typedef lambda::function<void(const T&)> ReadyCallback;
  typedef lambda::function<void(const std::string&)> FailedCallback;
  typedef lambda::function<void()> DiscardedCallback;
  typedef lambda::function<void(const Future<T>&)> AnyCallback;

  // Installs callbacks for the specified events and returns a const
  // reference to 'this' in order to easily support chaining.
  const Future<T>& onDiscard(DiscardCallback&& callback) const;
  const Future<T>& onReady(ReadyCallback&& callback) const;
  const Future<T>& onFailed(FailedCallback&& callback) const;
  const Future<T>& onDiscarded(DiscardedCallback&& callback) const;
  const Future<T>& onAny(AnyCallback&& callback) const;

 ......
 
 private:
  friend class Promise<T>;
  friend class WeakFuture<T>;

  enum State
  {
    PENDING,
    READY,
    FAILED,
    DISCARDED,
  };
  
 ~~~

从上面可以看出，Future一共有四种状态，挂起，就绪，失败和丢弃。

 ~~~cpp

  struct Data
  {
    Data();
    ~Data() = default;

    void clearAllCallbacks();

    std::atomic_flag lock = ATOMIC_FLAG_INIT;
    State state;
    bool discard;
    bool associated;



 ~~~cpp

    // One of:
    //   1. None, the state is PENDING or DISCARDED.
    //   2. Some, the state is READY.
    //   3. Error, the state is FAILED; 'error()' stores the message.
    Result<T> result;

    std::vector<DiscardCallback> onDiscardCallbacks;
    std::vector<ReadyCallback> onReadyCallbacks;
    std::vector<FailedCallback> onFailedCallbacks;
    std::vector<DiscardedCallback> onDiscardedCallbacks;
    std::vector<AnyCallback> onAnyCallbacks;
  };

  // Sets the value for this future, unless the future is already set,
  // failed, or discarded, in which case it returns false.
  bool set(const T& _t);
  
  //这里是相关的Promise对象设置的，所以不是public的，Promise和Future都是成双成对出现的。

  // Sets this future as failed, unless the future is already set,
  // failed, or discarded, in which case it returns false.
  bool fail(const std::string& _message);

 //这里也是相关的Promise对象设置的，所以不是public的，Promise和Future都是成双成对出现的。

  std::shared_ptr<Data> data;
};
 ~~~

上面首先列出Future的数据成员，后续使用到的话会逐步引出它的成员函数。

由上面可以看出，一个Future包含四个状态，暂停，就绪，失败，丢弃。各种状态有不同的回调函数可以用来处理。     

下面结合Promise看看两者的实现逻辑：

 ~~~cpp
template <typename T>
class Promise
{
public:
  Promise();
  explicit Promise(const T& t);
  virtual ~Promise();

  Promise(Promise<T>&& that);

  bool discard();
  bool set(const T& _t);
  bool set(const Future<T>& future); // Alias for associate.
  bool associate(const Future<T>& future);
  bool fail(const std::string& message);

  // Returns a copy of the future associated with this promise.
  Future<T> future() const;

private:
  template <typename U>
  friend void internal::discarded(Future<U> future);

  // Not copyable, not assignable.
  Promise(const Promise<T>&);
  Promise<T>& operator=(const Promise<T>&);

  // Helper for doing the work of actually discarding a future (called
  // from Promise::discard as well as internal::discarded).
  static bool discard(Future<T> future);

  Future<T> f;
};
 ~~~
 
  从上面可以看出，Promise包含了一个
  Future，如果我们想要实现Promise-Future逻辑，一定要使用
  Promise的future()函数返回的Future，才能将两者关联起来。
  当然也可以通过associate手工关联两个对象。


如果在构造函数里面直接给Promise设置一个值，

 ~~~cpp
template < typename T >
Promise<T>::Promise(const T& t)
      : f(t) {}  
 ~~~  
则会直接触发Future的构造函数：

~~~cpp

template <typename T>
Future<T>::Future(const T& _t)
  : data(new Data())
{
  set(_t);
}
~~~
~~~cpp
template <typename T>
bool Future<T>::set(const T& _t)
{
  bool result = false;

  synchronized (data->lock) {
    if (data->state == PENDING) {
      data->result = _t;
      data->state = READY;
      result = true;
    }
  }

  // Invoke all callbacks associated with this future being READY. We
  // don't need a lock because the state is now in READY so there
  // should not be any concurrent modications.
  if (result) {
    internal::run(data->onReadyCallbacks, data->result.get());
    internal::run(data->onAnyCallbacks, *this);

    data->clearAllCallbacks();
  }

  return result;
}

~~~

然后我们分析一下Future的get函数：

~~~cpp
template <typename T>
const T& Future<T>::get() const
{
  if (!isReady()) {
    await();
  }

  CHECK(!isPending()) << "Future was in PENDING after await()";
  // We can't use CHECK_READY here due to check.hpp depending on future.hpp.
  if (!isReady()) {
    CHECK(!isFailed()) << "Future::get() but state == FAILED: " << failure();
    CHECK(!isDiscarded()) << "Future::get() but state == DISCARDED";
  }

  assert(data->result.isSome());
  return data->result.get();
}
~~~

从上面可以看到，如果数据没有准备好，get是会阻塞的，一直阻塞在await函数里面。

 ~~~cpp
namespace internal {

inline void awaited(Owned<Latch> latch)
{
  latch->trigger();
}

} // namespace internal {


template <typename T>
bool Future<T>::await(const Duration& duration) const
{
  ...
  
  Owned<Latch> latch(new Latch());

  bool pending = false;

  synchronized (data->lock) {
    if (data->state == PENDING) {
      pending = true;
      data->onAnyCallbacks.push_back(lambda::bind(&internal::awaited, latch));
    }
  }

  if (pending) {
    return latch->await(duration);
  }

  return true;
}
 ~~~
    
 当状态为PENDING的时候，将此Future的任何状态变化均通过internal::awaited回调来触发latch，如果Latch已经被触发了，则直接返回true。
 
 *src/latch.cpp*
 
 ~~~cpp
 
bool Latch::trigger()
{
  bool expected = false;
  if (triggered.compare_exchange_strong(expected, true)) {
    terminate(pid);
    return true;
  }
  return false;
}


bool Latch::await(const Duration& duration)
{
  if (!triggered.load()) {
    process::wait(pid, duration); 
    ...
    return triggered.load();
  }

  return true;
}
 ~~~

**Event基类**


~~~cpp
struct EventVisitor
{
  virtual ~EventVisitor() {}
  virtual void visit(const MessageEvent& event) {}
  virtual void visit(const DispatchEvent& event) {}
  virtual void visit(const HttpEvent& event) {}
  virtual void visit(const ExitedEvent& event) {}
  virtual void visit(const TerminateEvent& event) {}
};


struct Event
{
  virtual ~Event() {}

  virtual void visit(EventVisitor* visitor) const = 0;

  template <typename T>
  bool is() const
  {
    bool result = false;
    struct IsVisitor : EventVisitor
    {
      explicit IsVisitor(bool* _result) : result(_result) {}
      virtual void visit(const T& t) { *result = true; }
      bool* result;
    } visitor(&result);
    visit(&visitor);
    return result;
  }

  template <typename T>
  const T& as() const
  {
    const T* result = NULL;
    struct AsVisitor : EventVisitor
    {
      explicit AsVisitor(const T** _result) : result(_result) {}
      virtual void visit(const T& t) { *result = &t; }
      const T** result;
    } visitor(&result);
    visit(&visitor);
    if (result == NULL) {
      ABORT("Attempting to \"cast\" event incorrectly!");
    }
    return *result;
  }
};

~~~	
	
这两个类可以好好消化一下，需要对虚函数的执行机制有非常清楚的了解，方便的实现了dynamic_cast的机制。

~~~cpp
struct ExitedEvent : Event
{
  explicit ExitedEvent(const UPID& _pid)
    : pid(_pid) {}

  virtual void visit(EventVisitor* visitor) const
  {
    visitor->visit(*this);
  }

  const UPID pid;

private:
  // Keep ExitedEvent not assignable even though we made it copyable.
  // Note that we are violating the "rule of three" here but it helps
  // keep the fields const.
  ExitedEvent& operator=(const ExitedEvent&);
};
~~~

**TODO：**  补充一些例子


最主要的函数入口就是 include/process/process.hpp 里面的这个函数：

 ~~~cpp
void initialize(const std::string& delegate = "");

void initialize(const string& delegate)
{
  // Create a new ProcessManager and SocketManager.
  process_manager = new ProcessManager(delegate);
  socket_manager = new SocketManager();

  // Initialize the event loop.
  EventLoop::initialize();   //1️⃣ 
  
  // Setup processing threads.
  long cpus = process_manager->init_threads(); //2️⃣

  Clock::initialize(lambda::bind(&timedout, lambda::_1));

  __address__ = Address::LOCALHOST_ANY();

  // Check environment for ip.
  Option<string> value = os::getenv("LIBPROCESS_IP");
  if (value.isSome()) {
    Try<net::IP> ip = net::IP::parse(value.get(), AF_INET);
    if (ip.isError()) {
      LOG(FATAL) << "Parsing LIBPROCESS_IP=" << value.get()
                 << " failed: " << ip.error();
    }
    __address__.ip = ip.get();
  }

  // Check environment for port.
  value = os::getenv("LIBPROCESS_PORT");
  if (value.isSome()) {
    Try<int> result = numify<int>(value.get().c_str());
    if (result.isSome() && result.get() >=0 && result.get() <= USHRT_MAX) {
      __address__.port = result.get();
    } else {
      LOG(FATAL) << "LIBPROCESS_PORT=" << value.get() << " is not a valid port";
    }
  }

  // Create a "server" socket for communicating.
  Try<Socket> create = Socket::create();
  if (create.isError()) {
    PLOG(FATAL) << "Failed to construct server socket:" << create.error();
  }
  __s__ = new Socket(create.get());

  // Allow address reuse.
  int on = 1;
  if (setsockopt(__s__->get(), SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0) {
    PLOG(FATAL) << "Failed to initialize, setsockopt(SO_REUSEADDR)";
  }

  Try<Address> bind = __s__->bind(__address__);
  if (bind.isError()) {
    PLOG(FATAL) << "Failed to initialize: " << bind.error();
  }

  __address__ = bind.get();

  // If advertised IP and port are present, use them instead.
  value = os::getenv("LIBPROCESS_ADVERTISE_IP");
  if (value.isSome()) {
    Try<net::IP> ip = net::IP::parse(value.get(), AF_INET);
    if (ip.isError()) {
      LOG(FATAL) << "Parsing LIBPROCESS_ADVERTISE_IP=" << value.get()
                 << " failed: " << ip.error();
    }
    __address__.ip = ip.get();
  }

  value = os::getenv("LIBPROCESS_ADVERTISE_PORT");
  if (value.isSome()) {
    Try<int> result = numify<int>(value.get().c_str());
    if (result.isSome() && result.get() >=0 && result.get() <= USHRT_MAX) {
      __address__.port = result.get();
    } else {
      LOG(FATAL) << "LIBPROCESS_ADVERTISE_PORT=" << value.get()
                 << " is not a valid port";
    }
  }

  // Lookup hostname if missing ip or if ip is 0.0.0.0 in case we
  // actually have a valid external ip address. Note that we need only
  // one ip address, so that other processes can send and receive and
  // don't get confused as to whom they are sending to.
  if (__address__.ip.isAny()) {
    char hostname[512];

    if (gethostname(hostname, sizeof(hostname)) < 0) {
      LOG(FATAL) << "Failed to initialize, gethostname: "
                 << hstrerror(h_errno);
    }

    // Lookup IP address of local hostname.
    Try<net::IP> ip = net::getIP(hostname, __address__.ip.family());

    if (ip.isError()) {
      EXIT(EXIT_FAILURE)
        << "Failed to obtain the IP address for '" << hostname << "';"
        << " the DNS service may not be able to resolve it: " << ip.error();
    }

    __address__.ip = ip.get();
  }

  Try<Nothing> listen = __s__->listen(LISTEN_BACKLOG);
  if (listen.isError()) {
    PLOG(FATAL) << "Failed to initialize: " << listen.error();
  }

  // Need to set `initialize_complete` here so that we can actually
  // invoke `accept()` and `spawn()` below.
  initialize_complete.store(true);

  __s__->accept()
    .onAny(lambda::bind(&internal::on_accept, lambda::_1));//3️⃣

  // TODO(benh): Make sure creating the garbage collector, logging
  // process, and profiler always succeeds and use supervisors to make
  // sure that none terminate.

  // Create global garbage collector process.
  gc = spawn(new GarbageCollector());//4️⃣

  // Create global help process.
  help = spawn(new Help(), true);//5️⃣

  // Create the global logging process.
  spawn(new Logging(), true);//6️⃣

  // Create the global profiler process.
  spawn(new Profiler(), true);//7️⃣

  // Create the global system statistics process.
  spawn(new System(), true);//8️⃣

  // Create the global HTTP authentication router.
  authenticator_manager = new AuthenticatorManager();//9️⃣

  // Ensure metrics process is running.
  // TODO(bmahler): Consider initializing this consistently with
  // the other global Processes.
  metrics::internal::MetricsProcess* metricsProcess =
    metrics::internal::MetricsProcess::instance();

  CHECK_NOTNULL(metricsProcess);

  // Initialize the mime types.
  mime::initialize();

  // Add a route for getting process information.
  lambda::function<Future<Response>(const Request&)> __processes__ =
    lambda::bind(&ProcessManager::__processes__, process_manager, lambda::_1);

  new Route("/__processes__", None(), __processes__);//🔟

  VLOG(1) << "libprocess is initialized on " << address() << " for " << cpus
          << " cpus";
}

 ~~~

**全局变量**

下面是实现的cpp文件里面的几个静态全局变量，主要的逻辑在这里：

 ~~~cpp
 // Server socket listen backlog.
static const int LISTEN_BACKLOG = 500000;

// Local server socket.
static Socket* __s__ = NULL;

// Local socket address.
static Address __address__;

// Active SocketManager (eventually will probably be thread-local).
static SocketManager* socket_manager = NULL;

// Active ProcessManager (eventually will probably be thread-local).
static ProcessManager* process_manager = NULL;

// Scheduling gate that threads wait at when there is nothing to run.
static Gate* gate = new Gate();

// Used for authenticating HTTP requests.
static AuthenticatorManager* authenticator_manager = NULL;
// Filter. Synchronized support for using the filterer needs to be
// recursive in case a filterer wants to do anything fancy (which is
// possible and likely given that filters will get used for testing).
static Filter* filterer = NULL;
static std::recursive_mutex* filterer_mutex = new std::recursive_mutex();

// Global garbage collector.
PID<GarbageCollector> gc;

// Global help.
PID<Help> help;

// Per thread process pointer.
THREAD_LOCAL ProcessBase* __process__ = NULL;

// Per thread executor pointer.
THREAD_LOCAL Executor* _executor_ = NULL;
 ~~~
**工具类**


 
 

1️⃣网络的初始化

 ~~~cpp
 //1️⃣
event_base* base = NULL;

void EventLoop::initialize()
{
  static Once* initialized = new Once();

  if (initialized->once()) {
    return;
  }

  ...
  
  struct event_config* config = event_config_new();
  event_config_avoid_method(config, "epoll");

  base = event_base_new_with_config(config);

  initialized->done();
}

或者 libev.cpp

ev_async async_watcher;
// We need an asynchronous watcher to receive the request to shutdown.
ev_async shutdown_watcher;

// Define the initial values for all of the declarations made in
// libev.hpp (since these need to live in the static data space).
struct ev_loop* loop = NULL;


THREAD_LOCAL bool* _in_event_loop_ = NULL;

void EventLoop::initialize()
{
  loop = ev_default_loop(EVFLAG_AUTO);

  ev_async_init(&async_watcher, handle_async);
  ev_async_init(&shutdown_watcher, handle_shutdown);

  ev_async_start(loop, &async_watcher);
  ev_async_start(loop, &shutdown_watcher);
}

void handle_async(struct ev_loop* loop, ev_async* _, int revents)
{
  std::queue<lambda::function<void()>> run_functions;
  synchronized (watchers_mutex) {
    // Start all the new I/O watchers.
    while (!watchers->empty()) {
      ev_io* watcher = watchers->front();
      watchers->pop();
      ev_io_start(loop, watcher);
    }

    // Swap the functions into a temporary queue so that we can invoke
    // them outside of the mutex.
    std::swap(run_functions, *functions);
  }

  // Running the functions outside of the mutex reduces locking
  // contention as these are arbitrary functions that can take a long
  // time to execute. Doing this also avoids a deadlock scenario where
  // (A) mutexes are acquired before calling `run_in_event_loop`,
  // followed by locking (B) `watchers_mutex`. If we executed the
  // functions inside the mutex, then the locking order violation
  // would be this function acquiring the (B) `watchers_mutex`
  // followed by the arbitrary function acquiring the (A) mutexes.
  while (!run_functions.empty()) {
    (run_functions.front())();
    run_functions.pop();
  }
}

 ~~~
 
 2️⃣初始化线程
 
 ~~~cpp
class ProcessManager
{
   ...
    // Boolean used to signal processing threads to stop running.
  std::atomic_bool joining_threads;
}

long ProcessManager::init_threads()
{
  joining_threads.store(false);

  // We create no fewer than 8 threads because some tests require
  // more worker threads than `sysconf(_SC_NPROCESSORS_ONLN)` on
  // computers with fewer cores.
  // e.g. https://issues.apache.org/jira/browse/MESOS-818
  //
  // TODO(xujyan): Use a smarter algorithm to allocate threads.
  // Allocating a static number of threads can cause starvation if
  // there are more waiting Processes than the number of worker
  // threads.
  long cpus = std::max(8L, sysconf(_SC_NPROCESSORS_ONLN));
  threads.reserve(cpus+1);

  // Create processing threads.
  for (long i = 0; i < cpus; i++) {
    // Retain the thread handles so that we can join when shutting down.
    threads.emplace_back(
        // We pass a constant reference to `joining` to make it clear that this
        // value is only being tested (read), and not manipulated.
        new std::thread(std::bind([](const std::atomic_bool& joining) {
          do {
          
          //这里有点double check的味道。
            ProcessBase* process = process_manager->dequeue();
            if (process == NULL) {
              //队列里面没有消息需要处理，Idle计数器加1
              Gate::state_t old = gate->approach();
              process = process_manager->dequeue();
              if (process == NULL) {
                if (joining.load()) {
                  break;
                }
                //一直等待队列有数据为止。看来往队列里里面插入数据时一定会调用Gate的某个方法来通知这里。
                gate->arrive(old); // Wait at gate if idle.
                continue;
              } else {
                gate->leave();
              }
            }
            process_manager->resume(process);  //✺11
          } while (true);
        },
        std::cref(joining_threads))));
  }

  // Create a thread for the event loop.
  threads.emplace_back(new std::thread(&EventLoop::run)); //✺12

  return cpus;
}
 ~~~
 
 主要是根据CPU个数启动了N个线程，以及EventLoop的一个线程。
 Worker线程主要是监听ProcessManager的消息队列，如果有数据，则进行处理，没有数据则阻塞等待。
 

我们首先分析 ✺12 ：

 ~~~cpp
class EventLoop
{
public:
  ...
  static void run();
};

extern THREAD_LOCAL bool* _in_event_loop_;

#define __in_event_loop__ *(_in_event_loop_ == NULL ?                \
  _in_event_loop_ = new bool(false) : _in_event_loop_)


void EventLoop::run()
{
  __in_event_loop__ = true;

  do {
    int result = event_base_loop(base, EVLOOP_ONCE);
    if (result < 0) {
      LOG(FATAL) << "Failed to run event loop";
    } else if (result > 0) {
      // All events are handled, continue event loop.
      continue;
    } else {
      CHECK_EQ(0, result);
      if (event_base_got_break(base)) {
        break;
      } else if (event_base_got_exit(base)) {
        break;
      }
    }
  } while (true);

  __in_event_loop__ = false;
}
 ~~~
这里仅仅是启动了网络的事件监听循环。

 ~~~cpp
class Socket
{
public:
 ...
  class Impl : public std::enable_shared_from_this<Impl>
  {
  public:
    virtual Try<Nothing> listen(int backlog) = 0;
    virtual Future<Socket> accept() = 0;
  protected:
    int s;
  };
  
  Future<Socket> accept()
  {
    return impl->accept();
  }
private:  
  std::shared_ptr<Impl> impl;
};

class PollSocketImpl : public Socket::Impl
{
public:
  ...
  virtual Future<Socket> accept();
  };
};

namespace internal {

Future<Socket> accept(int fd)
{
  Try<int> accepted = network::accept(fd);
  if (accepted.isError()) {
    return Failure(accepted.error());
  }

  int s = accepted.get();
  Try<Nothing> nonblock = os::nonblock(s);
  if (nonblock.isError()) {
    LOG_IF(INFO, VLOG_IS_ON(1)) << "Failed to accept, nonblock: "
                                << nonblock.error();
    os::close(s);
    return Failure("Failed to accept, nonblock: " + nonblock.error());
  }

  Try<Nothing> cloexec = os::cloexec(s);
  if (cloexec.isError()) {
    LOG_IF(INFO, VLOG_IS_ON(1)) << "Failed to accept, cloexec: "
                                << cloexec.error();
    os::close(s);
    return Failure("Failed to accept, cloexec: " + cloexec.error());
  }

  // Turn off Nagle (TCP_NODELAY) so pipelined requests don't wait.
  int on = 1;
  if (setsockopt(s, SOL_TCP, TCP_NODELAY, &on, sizeof(on)) < 0) {
    const string error = os::strerror(errno);
    VLOG(1) << "Failed to turn off the Nagle algorithm: " << error;
    os::close(s);
    return Failure(
      "Failed to turn off the Nagle algorithm: " + stringify(error));
  }

  Try<Socket> socket = Socket::create(Socket::DEFAULT_KIND(), s);
  if (socket.isError()) {
    os::close(s);
    return Failure("Failed to accept, create socket: " + socket.error());
  }
  return socket.get();
}

} // namespace internal {


Future<Socket> PollSocketImpl::accept()
{
  return io::poll(get(), io::READ)
    .then(lambda::bind(&internal::accept, get()));
}
 ~~~
 这里就是使用libev的事件循环机制来实现了
 
 ~~~cpp


Future<short> poll(int fd, short events)
{
  process::initialize();

  // TODO(benh): Check if the file descriptor is non-blocking?

  return run_in_event_loop<short>(lambda::bind(&internal::poll, fd, events));
}

// Helper for running a function in the event loop.
template <typename T>
Future<T> run_in_event_loop(const lambda::function<Future<T>()>& f)
{
  // If this is already the event loop then just run the function.
  if (__in_event_loop__) {
    return f();
  }

  Owned<Promise<T>> promise(new Promise<T>());

  Future<T> future = promise->future();

  // Enqueue the function.
  synchronized (watchers_mutex) {
    functions->push(lambda::bind(&_run_in_event_loop<T>, f, promise));
  }

  // Interrupt the loop.
  ev_async_send(loop, &async_watcher);

  return future;
}
 ~~~
libev里面，async_watcher的回调函数是：

 ~~~cpp
 std::queue<ev_io*>* watchers = new std::queue<ev_io*>();

std::mutex* watchers_mutex = new std::mutex();

std::queue<lambda::function<void()>>* functions =
  new std::queue<lambda::function<void()>>();

void handle_async(struct ev_loop* loop, ev_async* _, int revents)
{
  std::queue<lambda::function<void()>> run_functions;
  synchronized (watchers_mutex) {
    // Start all the new I/O watchers.
    while (!watchers->empty()) {
      ev_io* watcher = watchers->front();
      watchers->pop();
      ev_io_start(loop, watcher);
    }

    // Swap the functions into a temporary queue so that we can invoke
    // them outside of the mutex.
    std::swap(run_functions, *functions);
  }

  // Running the functions outside of the mutex reduces locking
  // contention as these are arbitrary functions that can take a long
  // time to execute. Doing this also avoids a deadlock scenario where
  // (A) mutexes are acquired before calling `run_in_event_loop`,
  // followed by locking (B) `watchers_mutex`. If we executed the
  // functions inside the mutex, then the locking order violation
  // would be this function acquiring the (B) `watchers_mutex`
  // followed by the arbitrary function acquiring the (A) mutexes.
  while (!run_functions.empty()) {
    (run_functions.front())();
    run_functions.pop();
  }
}

 ~~~
 
 上述就是为了确保poll是在某个线程的evloop里面去执行，这时候evloop已经创建成功了。最终调用的是函数internal::poll：
 
 ~~~cpp
 struct Poll
{
  Poll()
  {
    // Need to explicitly instantiate the watchers.
    watcher.io.reset(new ev_io());
    watcher.async.reset(new ev_async());
  }

  // An I/O watcher for checking for readability or writeability and
  // an async watcher for being able to discard the polling.
  struct {
    std::shared_ptr<ev_io> io;
    std::shared_ptr<ev_async> async;
  } watcher;

  Promise<short> promise;
};
 ~~~
 为了方便回调处理，设置了上述类型Poll
 
 ~~~cpp

// Event loop callback when I/O is ready on polling file descriptor.
void polled(struct ev_loop* loop, ev_io* watcher, int revents)
{
  Poll* poll = (Poll*) watcher->data;

  ev_io_stop(loop, poll->watcher.io.get());

  // Stop the async watcher (also clears if pending so 'discard_poll'
  // will not get invoked and we can delete 'poll' here).
  ev_async_stop(loop, poll->watcher.async.get());

  poll->promise.set(revents);

  delete poll;
}


// Event loop callback when future associated with polling file
// descriptor has been discarded.
void discard_poll(struct ev_loop* loop, ev_async* watcher, int revents)
{
  Poll* poll = (Poll*) watcher->data;

  // Check and see if we have a pending 'polled' callback and if so
  // let it "win".
  if (ev_is_pending(poll->watcher.io.get())) {
    return;
  }

  ev_async_stop(loop, poll->watcher.async.get());

  // Stop the I/O watcher (but note we check if pending above) so it
  // won't get invoked and we can delete 'poll' here.
  ev_io_stop(loop, poll->watcher.io.get());

  poll->promise.discard();

  delete poll;
}


namespace io {
namespace internal {

// Helper/continuation of 'poll' on future discard.
void _poll(const std::shared_ptr<ev_async>& async)
{
  ev_async_send(loop, async.get());
}


Future<short> poll(int fd, short events)
{
  Poll* poll = new Poll();

  // Have the watchers data point back to the struct.
  poll->watcher.async->data = poll;
  poll->watcher.io->data = poll;

  // Get a copy of the future to avoid any races with the event loop.
  Future<short> future = poll->promise.future();

  // Initialize and start the async watcher.
  ev_async_init(poll->watcher.async.get(), discard_poll);✺20
  ev_async_start(loop, poll->watcher.async.get());

  // Make sure we stop polling if a discard occurs on our future.
  // Note that it's possible that we'll invoke '_poll' when someone
  // does a discard even after the polling has already completed, but
  // in this case while we will interrupt the event loop since the
  // async watcher has already been stopped we won't cause
  // 'discard_poll' to get invoked.
  
  //注意，这里设置了onDiscard回调，当watcher.async的事件被回调时，会调用这里设置的方法。
  future.onDiscard(lambda::bind(&_poll, poll->watcher.async));✺21

  // Initialize and start the I/O watcher.
  ev_io_init(poll->watcher.io.get(), polled, fd, events);✺22
  ev_io_start(loop, poll->watcher.io.get());

  return future;
}

} 

 ~~~
 从✺20处可以看到，一定是在某个地方可以触发停止监听pool的，但是从现在的分析中还没有看到，我们看看后续的代码分析中能否发现一些端倪。代码✺21处，就是当POOL监听停止的时候，我们这个future会得到通知，然后会执行_poll这个回调。代码✺22处，就是开始监听这个socket是否有数据到来。所以我们先分析polled函数，从这个函数名称也可以看出，已经是有数据到来了，所以才加了ed这个后缀，表示事情已经发生😛。
 
 回过头来看polled函数，很明显，revents就是我们想要的结果，这时候把相关的promise设置返回值，关联的future就得到通知了。但是他得到的是onReady通知。这时候accept函数就可以返回了。函数成功返回了之后就会调用then设置的回调函数，就是accept函数。这时候开始正常的网络流程，就是调用network::accept进行系统调用。设置socket的一些基本属性，然后返回Socket对象。这时候返回的Future实际上要么是失败的，要么是成功设置值的，就是Future已经有结论了。因为他是从一个Try对象初始化的。
 
 下面简单看一下Future的这个构造函数：
 
~~~cpp
  
template <typename T>
Future<T>::Future(const Try<T>& t)
  : data(new Data())
{
  if (t.isSome()){
    set(t.get());
  } else {
    fail(t.error());
  }
}

~~~


说到这里，我们可以开始介绍Future<T>::then函数。因为前面我们也看到了，我们需要在某个异步函数调用成功之后再启用我们的accept机制。尽管所有的操作都是异步的，但是有时候不同的操作时间还是需要同步完成。


~~~cpp
template <typename T>
template <typename X>
Future<X> Future<T>::then(const lambda::function<X(const T&)>& f) const
{
  std::shared_ptr<Promise<X>> promise(new Promise<X>());

  lambda::function<void(const Future<T>&)> then =
    lambda::bind(&internal::then<T, X>, f, promise, lambda::_1);  ✺30

  onAny(then);

  // Propagate discarding up the chain. To avoid cyclic dependencies,
  // we keep a weak future in the callback.
  promise->future().onDiscard(
      lambda::bind(&internal::discard<T>, WeakFuture<T>(*this))); ✺31

  return promise->future();
}
~~~
这里为啥要有两个template呢，当然是为了后面函数的返回值了。从上面函数可以看出，首先new了一个新的promise，这个Promise当然是返回X的值了。✺30显示前面的Future的任何的状态变化都是要回调then这个函数的，这样我们就完成了同步，是不是？第一个Future的状态变化表示第一个已经完成，所以才会调用第二个！，但是万一用户Cancel了第二步的操作呢？前面的Future也必须得到通知是不是？这就是✺31行代码的作用，后向反馈！整条链条全部取消！

接下来我们看看internal::then做了什么？

~~~cpp
template <typename T, typename X>
void then(const lambda::function<X(const T&)>& f,
          const std::shared_ptr<Promise<X>>& promise,
          const Future<T>& future)
{
  if (future.isReady()) {
    if (future.hasDiscard()) {
      promise->discard();
    } else {
      promise->set(f(future.get()));
    }
  } else if (future.isFailed()) {
    promise->fail(future.failure());
  } else if (future.isDiscarded()) {
    promise->discard();
  }
}
~~~

如果Future状态为READY的话，就调用get，然后调用f这个函数，这样调用链就起来了。同时也考虑了其他的状态情况。从上面我们也可以看出，一个Future，在任何状态下都有可能被discard。这里引申一下Promise的discard和Future的discard的不同，当丢弃Promise的时候，Future的状态才是DISCARDED状态，当丢弃Future的时候，不会修改状态，只是标记discard，可以理解为：管道的写端不关闭，读端做个标记，表示是否可读。写端一旦关闭，读端立即关闭。注意50和51处的区别。

~~~cpp
template <typename T>
bool Promise<T>::discard(Future<T> future)
{
  std::shared_ptr<typename Future<T>::Data> data = future.data;

  bool result = false;

  synchronized (data->lock) {
    if (data->state == Future<T>::PENDING) {  ✺50
      data->state = Future<T>::DISCARDED;
      result = true;
    }
  }
  ...
  return result;
}

template <typename T>
bool Future<T>::discard()
{
  bool result = false;

  std::vector<DiscardCallback> callbacks;
  synchronized (data->lock) {
    if (!data->discard && data->state == PENDING) {
      result = data->discard = true;      ✺51

       callbacks = data->onDiscardCallbacks;
      data->onDiscardCallbacks.clear();
    }
  }
  ...

  return result;
}

~~~

OK，我们继续分析process::initialize处的第3处代码：

~~~cpp
__s__->accept()
    .onAny(lambda::bind(&internal::on_accept, lambda::_1));
~~~

这时候网络IO走到了accept这一步，真的不容易！

~~~cpp
//[process.cpp]

void on_accept(const Future<Socket>& socket)
{
  if (socket.isReady()) {
    // Inform the socket manager for proper bookkeeping.
    socket_manager->accepted(socket.get());

    const size_t size = 80 * 1024;
    char* data = new char[size];
    memset(data, 0, size);

    DataDecoder* decoder = new DataDecoder(socket.get());

    socket.get().recv(data, size)
      .onAny(lambda::bind(
          &internal::decode_recv,
          lambda::_1,
          data,
          size,
          new Socket(socket.get()),
          decoder));
  }

  __s__->accept()
    .onAny(lambda::bind(&on_accept, lambda::_1));
}
~~~

这一步，Socket已经准备好了，等待这个Socket的IO动作，并且继续等待下一个连接到来。
SocketManager保存这个Socket以做后用。

~~~cpp
void SocketManager::accepted(const Socket& socket)
{
  synchronized (mutex) {
    sockets[socket] = new Socket(socket);
  }
}

//[poll_socket.cpp]

Future<size_t> PollSocketImpl::recv(char* data, size_t size)
{
  return io::read(get(), data, size);
}

~~~

~~~cpp
//[io.cpp]
const size_t BUFFERED_READ_SIZE = 16*4096;

Future<size_t> read(int fd, void* data, size_t size)
{
  process::initialize();

  std::shared_ptr<Promise<size_t>> promise(new Promise<size_t>());

  // Check the file descriptor.
  Try<bool> nonblock = os::isNonblock(fd);
  if (nonblock.isError()) {
    // The file descriptor is not valid (e.g., has been closed).
    promise->fail(
        "Failed to check if file descriptor was non-blocking: " +
        nonblock.error());
    return promise->future();
  } else if (!nonblock.get()) {
    // The file descriptor is not non-blocking.
    promise->fail("Expected a non-blocking file descriptor");
    return promise->future();
  }

  // Because the file descriptor is non-blocking, we call read()
  // immediately. The read may in turn call poll if necessary,
  // avoiding unnecessary polling. We also observed that for some
  // combination of libev and Linux kernel versions, the poll would
  // block for non-deterministically long periods of time. This may be
  // fixed in a newer version of libev (we use 3.8 at the time of
  // writing this comment).
  internal::read(fd, data, size, internal::NONE, promise, io::READ);

  return promise->future();
}
~~~

~~~cpp

void read(
    int fd,
    void* data,
    size_t size,
    ReadFlags flags,
    const std::shared_ptr<Promise<size_t>>& promise,
    const Future<short>& future)
{
  // Ignore this function if the read operation has been discarded.
  if (promise->future().hasDiscard()) {
    CHECK(!future.isPending());
    promise->discard();
    return;
  }

  if (size == 0) {
    promise->set(0);
    return;
  }

  if (future.isDiscarded()) {
    promise->fail("Failed to poll: discarded future");
  } else if (future.isFailed()) {
    promise->fail(future.failure());
  } else {
    ssize_t length;
    if (flags == NONE) {
      length = ::read(fd, data, size);
    } else { // PEEK.
      // In case 'fd' is not a socket ::recv() will fail with ENOTSOCK and the
      // error will be propagted out.
      length = ::recv(fd, data, size, MSG_PEEK);
    }

    if (length < 0) {
      if (errno == EINTR || errno == EAGAIN || errno == EWOULDBLOCK) {
        // Restart the read operation.
        Future<short> future =
          io::poll(fd, process::io::READ).onAny(
              lambda::bind(&internal::read,
                           fd,
                           data,
                           size,
                           flags,
                           promise,
                           lambda::_1));

        // Stop polling if a discard occurs on our future.
        promise->future().onDiscard(
            lambda::bind(&process::internal::discard<short>,
                         WeakFuture<short>(future)));
      } else {
        // Error occurred.
        promise->fail(os::strerror(errno));
      }
    } else {
      promise->set(length);
    }
  }
}
~~~

从这里看出，异步就是一种有序的循环。注意这里是怎么处理EINTR, EAGAIN 等常见网络错误的。

至此，数据已经读取完毕，可以开始解析数据了。上面可以看出，recv获取了数据之后会回调decode_recv函数。

~~~cpp
void decode_recv(
    const Future<size_t>& length,
    char* data,
    size_t size,
    Socket* socket,
    DataDecoder* decoder)
{
  ...
  // Decode as much of the data as possible into HTTP requests.
  const deque<Request*> requests = decoder->decode(data, length.get());

  ...

  if (!requests.empty()) {
    // Get the peer address to augment the requests.
    Try<Address> address = socket->peer();

	...

    foreach (Request* request, requests) {
      request->client = address.get();
      process_manager->handle(decoder->socket(), request);
    }
  }

  socket->recv(data, size)
    .onAny(lambda::bind(&decode_recv, lambda::_1, data, size, socket, decoder));
}
~~~
解析完了request，继续接受数据，并且解析这一流程

辅助函数，罗列一下：

~~~cpp
static Message* parse(Request* request)
{
  // TODO(benh): Do better error handling (to deal with a malformed
  // libprocess message, malicious or otherwise).

  // First try and determine 'from'.
  Option<UPID> from = None();

  if (request->headers.contains("Libprocess-From")) {
    from = UPID(strings::trim(request->headers["Libprocess-From"]));
  } else {
    // Try and get 'from' from the User-Agent.
    const string& agent = request->headers["User-Agent"];
    const string identifier = "libprocess/";
    size_t index = agent.find(identifier);
    if (index != string::npos) {
      from = UPID(agent.substr(index + identifier.size(), agent.size()));
    }
  }

  if (from.isNone()) {
    return NULL;
  }

  // Now determine 'to'.
  size_t index = request->url.path.find('/', 1);
  index = index != string::npos ? index - 1 : string::npos;

  // Decode possible percent-encoded 'to'.
  Try<string> decode = http::decode(request->url.path.substr(1, index));

  if (decode.isError()) {
    VLOG(2) << "Failed to decode URL path: " << decode.get();
    return NULL;
  }

  //!!!这里的decode很重要，是UPID的id，这里对应的是某个process，解析出来的事件会放到这个process的事件队列里面去
  const UPID to(decode.get(), __address__);

  // And now determine 'name'.
  index = index != string::npos ? index + 2: request->url.path.size();
  const string name = request->url.path.substr(index);

  VLOG(2) << "Parsed message name '" << name
          << "' for " << to << " from " << from.get();

  Message* message = new Message();
  message->name = name;
  message->from = from.get();
  message->to = to;
  message->body = request->body;

  return message;
}

ProcessReference ProcessManager::use(const UPID& pid)
{
  if (pid.address == __address__) {
    synchronized (processes_mutex) {
      if (processes.count(pid.id) > 0) {
        // Note that the ProcessReference constructor _must_ get
        // called while holding the lock on processes so that waiting
        // for references is atomic (i.e., race free).
        return ProcessReference(processes[pid.id]);
      }
    }
  }

  return ProcessReference(NULL);
}

class ProcessManager
{
map<string, ProcessBase*> processes;
}

bool ProcessManager::deliver(
    const UPID& to,
    Event* event,
    ProcessBase* sender /* = NULL*/ )
{
  CHECK(event != NULL);

  if (ProcessReference receiver = use(to)) {
    return deliver(receiver, event, sender);
  }
  VLOG(2) << "Dropping event for process " << to;

  delete event;
  return false;
}

bool ProcessManager::deliver(
    ProcessBase* receiver,
    Event* event,
    ProcessBase* sender /* = NULL */)
{
  CHECK(event != NULL);

  ...

  receiver->enqueue(event);

  return true;
}

void ProcessBase::enqueue(Event* event, bool inject)
{
  CHECK(event != NULL);

  synchronized (mutex) {
    if (state != TERMINATING && state != TERMINATED) {
      if (!inject) {
        events.push_back(event);
      } else {
        events.push_front(event);
      }

      if (state == BLOCKED) {
        state = READY;
        process_manager->enqueue(this);
      }

      CHECK(state == BOTTOM ||
            state == READY ||
            state == RUNNING);
    } else {
      delete event;
    }
  }
}

ProcessBase* ProcessManager::dequeue()
{
  ProcessBase* process = NULL;

  synchronized (runq_mutex) {
    if (!runq.empty()) {
      process = runq.front();
      runq.pop_front();
      // Increment the running count of processes in order to support
      // the Clock::settle() operation (this must be done atomically
      // with removing the process from the runq).
      running.fetch_add(1);
    }
  }

  return process;
}

~~~

这时候就把时间放到了本机的某个Process的event队列里面去了

~~~cpp

//[process.cpp:2256]
void ProcessManager::handle(
    const Socket& socket,
    Request* request)
{
  CHECK(request != NULL);

  // Check if this is a libprocess request (i.e., 'User-Agent:
  // libprocess/id@ip:port') and if so, parse as a message.
  if (libprocess(request)) {
    Message* message = parse(request);
    if (message != NULL) {
      // TODO(benh): Use the sender PID when delivering in order to
      // capture happens-before timing relationships for testing.
      bool accepted = deliver(message->to, new MessageEvent(message));

      // Get the HttpProxy pid for this socket.
      PID<HttpProxy> proxy = socket_manager->proxy(socket);

      // Only send back an HTTP response if this isn't from libprocess
      // (which we determine by looking at the User-Agent). This is
      // necessary because older versions of libprocess would try and
      // recv the data and parse it as an HTTP request which would
      // fail thus causing the socket to get closed (but now
      // libprocess will ignore responses, see ignore_data).
      Option<string> agent = request->headers.get("User-Agent");
      if (agent.getOrElse("").find("libprocess/") == string::npos) {
        if (accepted) {
          VLOG(2) << "Accepted libprocess message to " << request->url.path;
          dispatch(proxy, &HttpProxy::enqueue, Accepted(), *request);
        } else {
          VLOG(1) << "Failed to handle libprocess message to "
                  << request->url.path << ": not found";
          dispatch(proxy, &HttpProxy::enqueue, NotFound(), *request);
        }
      }

      delete request;
      return;
    }

    VLOG(1) << "Failed to handle libprocess message: "
            << request->method << " " << request->url.path
            << " (User-Agent: " << request->headers["User-Agent"] << ")";

    delete request;
    return;
  }

  // Treat this as an HTTP request. Start by checking that the path
  // starts with a '/' (since the code below assumes as much).
  if (request->url.path.find('/') != 0) {
    VLOG(1) << "Returning '400 Bad Request' for '" << request->url.path << "'";

    // Get the HttpProxy pid for this socket.
    PID<HttpProxy> proxy = socket_manager->proxy(socket);

    // Enqueue the response with the HttpProxy so that it respects the
    // order of requests to account for HTTP/1.1 pipelining.
    dispatch(proxy, &HttpProxy::enqueue, BadRequest(), *request);

    // Cleanup request.
    delete request;
    return;
  }

  // Ignore requests with relative paths (i.e., contain "/..").
  if (request->url.path.find("/..") != string::npos) {
    VLOG(1) << "Returning '404 Not Found' for '" << request->url.path
            << "' (ignoring requests with relative paths)";

    // Get the HttpProxy pid for this socket.
    PID<HttpProxy> proxy = socket_manager->proxy(socket);

    // Enqueue the response with the HttpProxy so that it respects the
    // order of requests to account for HTTP/1.1 pipelining.
    dispatch(proxy, &HttpProxy::enqueue, NotFound(), *request);

    // Cleanup request.
    delete request;
    return;
  }

  // Split the path by '/'.
  vector<string> tokens = strings::tokenize(request->url.path, "/");

  // Try and determine a receiver, otherwise try and delegate.
  UPID receiver;

  if (tokens.size() == 0 && delegate != "") {
    request->url.path = "/" + delegate;
    receiver = UPID(delegate, __address__);
  } else if (tokens.size() > 0) {
    // Decode possible percent-encoded path.
    Try<string> decode = http::decode(tokens[0]);
    if (!decode.isError()) {
      receiver = UPID(decode.get(), __address__);
    } else {
      VLOG(1) << "Failed to decode URL path: " << decode.error();
    }
  }

  if (!use(receiver) && delegate != "") {
    // Try and delegate the request.
    request->url.path = "/" + delegate + request->url.path;
    receiver = UPID(delegate, __address__);
  }

  synchronized (firewall_mutex) {
    // Don't use a const reference, since it cannot be guaranteed
    // that the rules don't keep an internal state.
    foreach (Owned<firewall::FirewallRule>& rule, firewallRules) {
      Option<Response> rejection = rule->apply(socket, *request);
      if (rejection.isSome()) {
        VLOG(1) << "Returning '"<< rejection.get().status << "' for '"
                << request->url.path << "' (firewall rule forbids request)";

        // TODO(arojas): Get rid of the duplicated code to return an
        // error.

        // Get the HttpProxy pid for this socket.
        PID<HttpProxy> proxy = socket_manager->proxy(socket);

        // Enqueue the response with the HttpProxy so that it respects
        // the order of requests to account for HTTP/1.1 pipelining.
        dispatch(
            proxy,
            &HttpProxy::enqueue,
            rejection.get(),
            *request);

        // Cleanup request.
        delete request;
        return;
      }
    }
  }

  if (use(receiver)) {
    // The promise is created here but its ownership is passed
    // into the HttpEvent created below.
    Promise<Response>* promise(new Promise<Response>());

    PID<HttpProxy> proxy = socket_manager->proxy(socket);

    // Enqueue the response with the HttpProxy so that it respects the
    // order of requests to account for HTTP/1.1 pipelining.
    dispatch(proxy, &HttpProxy::handle, promise->future(), *request);

    // TODO(benh): Use the sender PID in order to capture
    // happens-before timing relationships for testing.
    deliver(receiver, new HttpEvent(request, promise));

    return;
  }

  // This has no receiver, send error response.
  VLOG(1) << "Returning '404 Not Found' for '" << request->url.path << "'";

  // Get the HttpProxy pid for this socket.
  PID<HttpProxy> proxy = socket_manager->proxy(socket);

  // Enqueue the response with the HttpProxy so that it respects the
  // order of requests to account for HTTP/1.1 pipelining.
  dispatch(proxy, &HttpProxy::enqueue, NotFound(), *request);

  // Cleanup request.
  delete request;
}
~~~

上述主要是解析http数据，生成Event，然后加入到本机的ProcessBase对应的事件队列里面去。

注意processes这个成员变量，每次spawn的时候都会加入process到这个变量中，这样这个process就可以处理事件了。

继续最前面，线程从队列里面取出数据后，会调用process_manager->resume(process);
这个函数负责调用process的

~~~cpp

void ProcessManager::resume(ProcessBase* process)
{
  __process__ = process;

  VLOG(2) << "Resuming " << process->pid << " at " << Clock::now();

  bool terminate = false;
  bool blocked = false;

  CHECK(process->state == ProcessBase::BOTTOM ||
        process->state == ProcessBase::READY);

  if (process->state == ProcessBase::BOTTOM) {
    process->state = ProcessBase::RUNNING;
    try { process->initialize(); }
    catch (...) { terminate = true; }
  }

  while (!terminate && !blocked) {
    Event* event = NULL;

    synchronized (process->mutex) {
      if (process->events.size() > 0) {
        event = process->events.front();
        process->events.pop_front();
        process->state = ProcessBase::RUNNING;
      } else {
        process->state = ProcessBase::BLOCKED;
        blocked = true;
      }
    }

    if (!blocked) {
      CHECK(event != NULL);

      // Determine if we should filter this event.
      synchronized (filterer_mutex) {
        if (filterer != NULL) {
          bool filter = false;
          struct FilterVisitor : EventVisitor
          {
            explicit FilterVisitor(bool* _filter) : filter(_filter) {}

            virtual void visit(const MessageEvent& event)
            {
              *filter = filterer->filter(event);
            }

            virtual void visit(const DispatchEvent& event)
            {
              *filter = filterer->filter(event);
            }

            virtual void visit(const HttpEvent& event)
            {
              *filter = filterer->filter(event);
            }

            virtual void visit(const ExitedEvent& event)
            {
              *filter = filterer->filter(event);
            }

            bool* filter;
          } visitor(&filter);

          event->visit(&visitor);

          if (filter) {
            delete event;
            continue; // Try and execute the next event.
          }
        }
      }

      // Determine if we should terminate.
      terminate = event->is<TerminateEvent>();

      // Now service the event.
      try {
        process->serve(*event);
      } catch (const std::exception& e) {
        std::cerr << "libprocess: " << process->pid
                  << " terminating due to "
                  << e.what() << std::endl;
        terminate = true;
      } catch (...) {
        std::cerr << "libprocess: " << process->pid
                  << " terminating due to unknown exception" << std::endl;
        terminate = true;
      }

      delete event;

      if (terminate) {
        cleanup(process);
      }
    }
  }

  __process__ = NULL;

  CHECK_GE(running.load(), 1);
  running.fetch_sub(1);
}
~~~


resume这里是最主要的，他会实现对Process的初始化逻辑，调用他的事件处理机制。这里的事件处理都是在单个线程里面完成的。可以实现类似确保某个逻辑线程安全的思路。至此Process的运行机制大概有了了解，然后我们看看进程Process是怎么加入到系统的。

接下来我们继续分析这一行代码：

~~~cpp
gc = spawn(new GarbageCollector());//4️⃣

UPID spawn(ProcessBase* process, bool manage)
{
  process::initialize();

  if (process != NULL) {
    // If using a manual clock, try and set current time of process
    // using happens before relationship between spawner (__process__)
    // and spawnee (process)!
    if (Clock::paused()) {
      Clock::update(process, Clock::now(__process__));
    }

    return process_manager->spawn(process, manage);
  } else {
    return UPID();
  }
}

UPID ProcessManager::spawn(ProcessBase* process, bool manage)
{
  CHECK(process != NULL);

  synchronized (processes_mutex) {
    if (processes.count(process->pid.id) > 0) {
      return UPID();
    } else {
      processes[process->pid.id] = process;
    }
  }

  // Use the garbage collector if requested.
  if (manage) {
    dispatch(gc, &GarbageCollector::manage<ProcessBase>, process);
  }

   UPID pid = process->self();

  enqueue(process);
  
  return pid;
}

void ProcessManager::enqueue(ProcessBase* process)
{
...

  synchronized (runq_mutex) {
    CHECK(find(runq.begin(), runq.end(), process) == runq.end());
    runq.push_back(process);
  }

  // Wake up the processing thread if necessary.
  gate->open();
}

~~~

spawn主要是把这个PRocess enqueue到自己的process队列中，这样就可以给这个process发送事件，来让ProcessManager resume他处理事件了~

最后看看terminate

~~~cpp
void ProcessManager::terminate(
    const UPID& pid,
    bool inject,
    ProcessBase* sender)
{
  if (ProcessReference process = use(pid)) {
    if (Clock::paused()) {
      Clock::update(process, Clock::now(sender != NULL ? sender : __process__));
    }

    if (sender != NULL) {
      process->enqueue(new TerminateEvent(sender->self()), inject);
    } else {
      process->enqueue(new TerminateEvent(UPID()), inject);
    }
  }
}

void dispatch(
    const UPID& pid,
    const std::shared_ptr<lambda::function<void(ProcessBase*)>>& f,
    const Option<const std::type_info*>& functionType)
{
  process::initialize();

  DispatchEvent* event = new DispatchEvent(pid, f, functionType);
  process_manager->deliver(pid, event, __process__);
}
~~~


注意：Process的生命周期：创建是外面创建的，ProcessManger的成员变量processes保存了所有的Process实例，一般如果不是系统GC自动管理的话，是不会销毁的。前面的spawn和resume只是不断的把这些实例入队和出队列处理数据而已。


至此，流程已经分析清楚了！

ProcessBase的几个函数对于我们了解也很重要，罗列如下：
都很简单，复杂的后面会解释一下。

~~~cpp
ProcessBase::ProcessBase(const string& id)
{
  process::initialize();

  state = ProcessBase::BOTTOM;

  refs = 0;

  pid.id = id != "" ? id : ID::generate();
  pid.address = __address__;

  // If using a manual clock, try and set current time of process
  // using happens before relationship between creator (__process__)
  // and createe (this)!
  if (Clock::paused()) {
    Clock::update(this, Clock::now(__process__), Clock::FORCE);
  }
}


ProcessBase::~ProcessBase() {}


void ProcessBase::enqueue(Event* event, bool inject)
{
  CHECK(event != NULL);

  synchronized (mutex) {
    if (state != TERMINATING && state != TERMINATED) {
      if (!inject) {
        events.push_back(event);
      } else {
        events.push_front(event);
      }

      if (state == BLOCKED) {
        state = READY;
        process_manager->enqueue(this);
      }

      CHECK(state == BOTTOM ||
            state == READY ||
            state == RUNNING);
    } else {
      delete event;
    }
  }
}


void ProcessBase::inject(
    const UPID& from,
    const string& name,
    const char* data,
    size_t length)
{
  if (!from)
    return;

  Message* message = encode(from, pid, name, string(data, length));

  enqueue(new MessageEvent(message), true);
}


void ProcessBase::send(
    const UPID& to,
    const string& name,
    const char* data,
    size_t length)
{
  if (!to) {
    return;
  }

  // Encode and transport outgoing message.
  transport(encode(pid, to, name, string(data, length)), this);
}


void ProcessBase::visit(const MessageEvent& event)
{
  if (handlers.message.count(event.message->name) > 0) {
    handlers.message[event.message->name](
        event.message->from,
        event.message->body);
  } else if (delegates.count(event.message->name) > 0) {
    VLOG(1) << "Delegating message '" << event.message->name
            << "' to " << delegates[event.message->name];
    Message* message = new Message(*event.message);
    message->to = delegates[event.message->name];
    transport(message, this);
  }
}


void ProcessBase::visit(const DispatchEvent& event)
{
  (*event.f)(this);
}


//参考一下dispatch的实现：
dispatch的原型是：void(ProcessBase*)；
所以上面的那一行(*event.f)(this);的this就容易理解了。

struct DispatchEvent : Event
{
  DispatchEvent(
      const UPID& _pid,
      const std::shared_ptr<lambda::function<void(ProcessBase*)>>& _f,
      const Option<const std::type_info*>& _functionType)
    : pid(_pid),
      f(_f),
      functionType(_functionType)
  {}

  virtual void visit(EventVisitor* visitor) const
  {
    visitor->visit(*this);
  }

  // PID receiving the dispatch.
  const UPID pid;

  // Function to get invoked as a result of this dispatch event.
  const std::shared_ptr<lambda::function<void(ProcessBase*)>> f;

  const Option<const std::type_info*> functionType;

private:
  // Not copyable, not assignable.
  DispatchEvent(const DispatchEvent&);
  DispatchEvent& operator=(const DispatchEvent&);
};


void dispatch(
    const UPID& pid,
    const std::shared_ptr<lambda::function<void(ProcessBase*)>>& f,
    const Option<const std::type_info*>& functionType)
{
  process::initialize();

  DispatchEvent* event = new DispatchEvent(pid, f, functionType);
  process_manager->deliver(pid, event, __process__);
}

下面两个函数existed和finalize处理ExitedEvent和TerminateEvent

void ProcessBase::visit(const ExitedEvent& event)
{
  exited(event.pid);
}

void ProcessBase::visit(const TerminateEvent& event)
{
  finalize();
}

有太多的route，这些其实都是帮助😂，吓我一跳。
void ProcessBase::route(
    const string& name,
    const Option<string>& help_,
    const HttpRequestHandler& handler)
{
  // Routes must start with '/'.
  CHECK(name.find('/') == 0);

  HttpEndpoint endpoint;
  endpoint.handler = handler;

  handlers.http[name.substr(1)] = endpoint;

  dispatch(help, &Help::add, pid.id, name, help_);
}
~~~


##async函数

~~~cpp
template <typename F>
Future<Nothing> async(
    const F& f,
    typename boost::enable_if<boost::is_void<typename result_of<F()>::type> >::type*) // NOLINT(whitespace/line_length)
{
  return AsyncExecutor().execute(f);
}

class AsyncExecutor
{
  AsyncExecutor()
  {
    process = new AsyncExecutorProcess();
    spawn(process, true); // Automatically GC.
  }

  virtual ~AsyncExecutor() {}
}

  template <typename F>
  Future<typename result_of<F()>::type> execute(
      const F& f,
      typename boost::disable_if<boost::is_void<typename result_of<F()>::type> >::type* = NULL) // NOLINT(whitespace/line_length)
  {
    // Need to disambiguate overloaded method.
    typename result_of<F()>::type(AsyncExecutorProcess::*method)(const F&, typename boost::disable_if<boost::is_void<typename result_of<F()>::type> >::type*) = // NOLINT(whitespace/line_length)
      &AsyncExecutorProcess::execute<F>;

    return dispatch(process, method, f, (void*) NULL);
  }

~~~

很明显，这里就是new了一个Process，然后dispatch了一个方法。就把某个函数变成异步的了。
