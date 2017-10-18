wangle

#EventBaseManager

学习单例模式

```cpp
std::atomic<EventBaseManager*> globalManager(nullptr);

EventBaseManager* EventBaseManager::get() {
  EventBaseManager* mgr = globalManager;
  if (mgr) {
    return mgr;
  }

  EventBaseManager* new_mgr = new EventBaseManager;
  bool exchanged = globalManager.compare_exchange_strong(mgr, new_mgr);
  if (!exchanged) {
    delete new_mgr;
    return mgr;
  } else {
    return new_mgr;
  }

```

每个线程一个EventBase,使用的是线程局部变量ThreadLocalPtr

```cpp
mutable folly::ThreadLocalPtr<EventBaseInfo> localStore_;
```

```cpp
EventBase * EventBaseManager::getEventBase() const {
  // have one?
  auto *info = localStore_.get();
  if (! info) {
    info = new EventBaseInfo();
    localStore_.reset(info);

    if (observer_) {
      info->eventBase->setObserver(observer_);
    }

    // start tracking the event base
    // XXX
    // note: ugly cast because this does something mutable
    // even though this method is defined as "const".
    // Simply removing the const causes trouble all over fbcode;
    // lots of services build a const EventBaseManager and errors
    // abound when we make this non-const.
    (const_cast<EventBaseManager *>(this))->trackEventBase(info->eventBase);
  }

  return info->eventBase;
}

```

为了维护所有的EventBase，在获取每个EventBase（调用EventBaseManager::getEventBase()）的时候，都会把这个EventBase保存起来，见函数：

```cpp
 void trackEventBase(EventBase *evb) {
    //自己维护了一个所有的EventBase列表.
    //EventBaseManager是全局单例的，但是EventBase是每个线程拥有的。所以需要上锁
    std::lock_guard<std::mutex> g(*&eventBaseSetMutex_);
    eventBaseSet_.insert(evb);
  }
```
