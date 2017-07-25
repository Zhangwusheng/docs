



#最后一章:好用的工具类
**Mutex**

~~~cpp

class Mutex
{
public:
  Mutex() : data(new Data()) {}

  Future<Nothing> lock()
  {
    Future<Nothing> future = Nothing();

    synchronized (data->lock) {
      if (!data->locked) {
        data->locked = true;
      } else {
        Owned<Promise<Nothing>> promise(new Promise<Nothing>());
        data->promises.push(promise);
        future = promise->future();
      }
    }

    return future;
  }

  void unlock()
  {
    // NOTE: We need to grab the promise 'date->promises.front()' but
    // set it outside of the critical section because setting it might
    // trigger callbacks that try to reacquire the lock.
    Owned<Promise<Nothing>> promise;

    synchronized (data->lock) {
      if (!data->promises.empty()) {
        // TODO(benh): Skip a future that has been discarded?
        promise = data->promises.front();
        data->promises.pop();
      } else {
        data->locked = false;
      }
    }

    if (promise.get() != NULL) {
      promise->set(Nothing());
    }
  }

private:
  struct Data
  {
    Data() : locked(false) {}

    ~Data()
    {
      // TODO(benh): Fail promises?
    }

    // Rather than use a process to serialize access to the mutex's
    // internal data we use a 'std::atomic_flag'.
    std::atomic_flag lock = ATOMIC_FLAG_INIT;

    // Describes the state of this mutex.
    bool locked;

    // Represents "waiters" for this lock.
    std::queue<Owned<Promise<Nothing>>> promises;
  };

  std::shared_ptr<Data> data;
};


~~~

**Once**

 ~~~cpp
 // Provides a _blocking_ abstraction that's useful for performing a
// task exactly once.
class Once
{
public:
  Once() : started(false), finished(false) {}

  ~Once() = default;

  // Returns true if this Once instance has already transitioned to a
  // 'done' state (i.e., the action you wanted to perform "once" has
  // been completed). Note that this BLOCKS until Once::done has been
  // called.
  bool once()
  {
    bool result = false;

    synchronized (mutex) {
      if (started) {
        while (!finished) {
          synchronized_wait(&cond, &mutex);
        }
        result = true;
      } else {
        started = true;
      }
    }

    return result;
  }

  // Transitions this Once instance to a 'done' state.
  void done()
  {
    synchronized (mutex) {
      if (started && !finished) {
        finished = true;
        cond.notify_all();
      }
    }
  }

private:
  // Not copyable, not assignable.
  Once(const Once& that);
  Once& operator=(const Once& that);

  std::mutex mutex;
  std::condition_variable cond;
  bool started;
  bool finished;
};
 ~~~

**Gate**
 
**Gate**的实现也比较有意思，可以仔细分析一下，利用一个计数器来实现状态的变化通知，Gate 类是类似于 futex 的实现（futex：The futex() system call provides a method for waiting until a certain
       condition becomes true.）：

~~~cpp
class Gate
{
public:
  typedef intptr_t state_t;

private:
  int waiters;
  state_t state;
  std::mutex mutex;
  std::condition_variable cond;

public:
  Gate() : waiters(0), state(0) {}

  ~Gate() = default;

  // Signals the state change of the gate to any (at least one) or
  // all (if 'all' is true) of the threads waiting on it.
  void open(bool all = true)
  {
    synchronized (mutex) {
      state++;
      if (all) {
        cond.notify_all();
      } else {
        cond.notify_one();
      }
    }
  }

  // Blocks the current thread until the gate's state changes from
  // the current state.
  void wait()
  {
    synchronized (mutex) {
      waiters++;
      state_t old = state;
      while (old == state) {
        synchronized_wait(&cond, &mutex);
      }
      waiters--;
    }
  }

  // Gets the current state of the gate and notifies the gate about
  // the intention to wait for its state change.
  // Call 'leave()' if no longer interested in the state change.
  state_t approach()
  {
    synchronized (mutex) {
      waiters++;
      return state;
    }
  }

  // Blocks the current thread until the gate's state changes from
  // the specified 'old' state. The 'old' state can be obtained by
  // calling 'approach()'. Returns the number of remaining waiters.
  int arrive(state_t old)
  {
    int remaining;

    synchronized (mutex) {
      while (old == state) {
        synchronized_wait(&cond, &mutex);
      }

      waiters--;
      remaining = waiters;
    }

    return remaining;
  }

  // Notifies the gate that a waiter (the current thread) is no
  // longer waiting for the gate's state change.
  void leave()
  {
    synchronized (mutex) {
      waiters--;
    }
  }
};
~~~
