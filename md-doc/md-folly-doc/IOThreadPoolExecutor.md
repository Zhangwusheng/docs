#NamedThreadFactory
这里首先学习怎么创建线程

###线程命名
直接使用std::thread,但是需要使用bind把需要运行的变成一个funciton

```cpp
class NamedThreadFactory : public ThreadFactory {
 public:
  explicit NamedThreadFactory(folly::StringPiece prefix)
    : prefix_(prefix.str()), suffix_(0) {}
  std::thread newThread(folly::Func&& func) override {
    auto thread = std::thread(std::move(func));
    folly::setThreadName(
        thread.native_handle(),
        folly::to<std::string>(prefix_, suffix_++));
    return thread;
  }

  void setNamePrefix(folly::StringPiece prefix) {
    prefix_ = prefix.str();
  }

  std::string getNamePrefix() {
    return prefix_;
  }

 private:
  std::string prefix_;
  std::atomic<uint64_t> suffix_;
};
```


#IOThreadPoolExecutor

###启动线程
```cpp
  explicit IOThreadPoolExecutor(
      size_t numThreads,
      std::shared_ptr<ThreadFactory> threadFactory =
          std::make_shared<NamedThreadFactory>("IOThreadPool"),
      folly::EventBaseManager* ebm = folly::EventBaseManager::get(),
      bool waitForAll = false);
      
  
  
// threadListLock_ is writelocked
void ThreadPoolExecutor::addThreads(size_t n) {
  std::vector<ThreadPtr> newThreads;
  for (size_t i = 0; i < n; i++) {
    newThreads.push_back(makeThread());
  }
  for (auto& thread : newThreads) {
    // TODO need a notion of failing to create the thread
    // and then handling for that case
    这里学习怎么生成一个新的线程
    thread->handle = threadFactory_->newThread(
        std::bind(&ThreadPoolExecutor::threadRun, this, thread));
    threadList_.add(thread);
  }
  
  等待线程启动完毕
  for (auto& thread : newThreads) {
    thread->startupBaton.wait();
  }
  
  通知线程已经完成，观察者模式
  for (auto& o : observers_) {
    for (auto& thread : newThreads) {
      o->threadStarted(thread.get());
    }
  }
}

      
```

###确保函数在同一线程中运行：
如何判断提交的函数是否在同一线程中运行：

```cpp
bool EventBase::loopBody(int flags) {

......
   在自己的运行线程中,保存线程ID 

  loopThread_.store(std::this_thread::get_id(), std::memory_order_release);

  if (!name_.empty()) {
    setThreadName(name_);
  }


---------------------------------------------------
  bool EventBase::runInEventBaseThread(Func fn) {
  // Send the message.
  // It will be received by the FunctionRunner in the EventBase's thread.

  // We try not to schedule nullptr callbacks
  if (!fn) {
    LOG(ERROR) << "EventBase " << this
               << ": Scheduling nullptr callbacks is not allowed";
    return false;
  }

  // Short-circuit if we are already in our event base
  //看看这里是怎么判断是不是同一个线程的
  if (inRunningEventBaseThread()) {
    runInLoop(std::move(fn));
    return true;

  }

  try {
    queue_->putMessage(std::move(fn));
  } catch (const std::exception& ex) {
    LOG(ERROR) << "EventBase " << this << ": failed to schedule function "
               << "for EventBase thread: " << ex.what();
    return false;
  }

  return true;
}

-----------------------------------------------------
  bool inRunningEventBaseThread() const {
    return loopThread_.load(std::memory_order_relaxed) ==
        std::this_thread::get_id();
  }
  
```