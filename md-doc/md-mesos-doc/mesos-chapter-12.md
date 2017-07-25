
#第四章 Master Leader选举和Slave启动分析，StatusUpdateMessage的一些讨论。

##1.  Master命令行参数
Master的命令行参数如下：

* root_submissions
			
	Can root submit frameworks?
	
* work_dir

      Directory path to store the persistent information stored in the 
      Registry. example: /var/lib/mesos/master))
      
* registry 

      Persistence strategy for the registry
      available options are replicated_log, in_memory .for testing
      replicated_log;
      
* quorum

      The size of the quorum of replicas when using replicated_log based
      registry. It is imperative to set this value to be a majority of
      masters i.e., quorum great than number of masters/2.
      
      NOTE: Not required if master is run in standalone mode non HA

* `zk_session_timeout`

     ZooKeeper session timeout.
     
* ip
     IP address to listen on.
            
* port

     Port to listen on.",
     
* zk

     ZooKeeper URL (used for leader election amongst masters)
      May be one of:zk://host1:port1,host2:port2,.../path

~~~cpp
src/messages/messages.proto

message RegisterFrameworkMessage {
  required FrameworkInfo framework = 1;
}
~~~


构造函数里面主要是初始化了一个MasterInfo对象：

***MasterInfo初始化***

~~~cpp
Master::Master(

// Master ID is generated randomly based on UUID.
  info_.set_id(UUID::random().toString());

  // NOTE: Currently, we store ip in MasterInfo in network order,
  // which should be fixed. See MESOS-1201 for details.
  // TODO(marco): The ip, port, hostname fields above are
  //     being deprecated; the code should be removed once
  //     the deprecation cycle is complete.
  info_.set_ip(self().address.ip.in().get().s_addr);

  info_.set_port(self().address.port);
  info_.set_pid(self());
  info_.set_version(MESOS_VERSION);

  // Determine our hostname or use the hostname provided.
  string hostname;

  if (flags.hostname.isNone()) {
    if (flags.hostname_lookup) {
      Try<string> result = net::getHostname(self().address.ip);

      if (result.isError()) {
        LOG(FATAL) << "Failed to get hostname: " << result.error();
      }

      hostname = result.get();
    } else {
      // We use the IP address for hostname if the user requested us
      // NOT to look it up, and it wasn't explicitly set via --hostname:
      hostname = stringify(self().address.ip);
    }
  } else {
    hostname = flags.hostname.get();
  }

  info_.set_hostname(hostname);

  // This uses the new `Address` message in `MasterInfo`.
  info_.mutable_address()->set_ip(stringify(self().address.ip));
  info_.mutable_address()->set_port(self().address.port);
  info_.mutable_address()->set_hostname(hostname);
  }
~~~

##Initialize函数：

Candidacy 翻译：候选资格

~~~cpp
void Master::initialize()
{
  ......
  
  contender->initialize(info_);

  // Start contending to be a leading master and detecting the current
  // leader.
  contender->contend()
    .onAny(defer(self(), &Master::contended, lambda::_1));
  detector->detect()
    .onAny(defer(self(), &Master::detected, lambda::_1));
    
    ......
 
}

void Master::contended(const Future<Future<Nothing>>& candidacy)
{
  CHECK(!candidacy.isDiscarded());

  if (candidacy.isFailed()) {
    EXIT(1) << "Failed to contend: " << candidacy.failure();
  }

  // Watch for candidacy change.
  candidacy.get()
    .onAny(defer(self(), &Master::lostCandidacy, lambda::_1));
}


void Master::lostCandidacy(const Future<Nothing>& lost)
{
  CHECK(!lost.isDiscarded());

  if (lost.isFailed()) {
    EXIT(1) << "Failed to watch for candidacy: " << lost.failure();
  }

  if (elected()) {
    EXIT(1) << "Lost leadership... committing suicide!";
  }

  LOG(INFO) << "Lost candidacy as a follower... Contend again";
  contender->contend()
    .onAny(defer(self(), &Master::contended, lambda::_1));
}

void Master::detected(const Future<Option<MasterInfo>>& _leader)
{
  CHECK(!_leader.isDiscarded());

  if (_leader.isFailed()) {
    EXIT(1) << "Failed to detect the leading master: " << _leader.failure()
            << "; committing suicide!";
  }

  bool wasElected = elected();
  leader = _leader.get();

  LOG(INFO) << "The newly elected leader is "
            << (leader.isSome()
                ? (leader.get().pid() + " with id " + leader.get().id())
                : "None");

  if (wasElected && !elected()) {
    EXIT(1) << "Lost leadership... committing suicide!";
  }

  if (elected()) {
    electedTime = Clock::now();

    if (!wasElected) {
      LOG(INFO) << "Elected as the leading master!";

      // Begin the recovery process, bail if it fails or is discarded.
      recover()
        .onFailed(lambda::bind(fail, "Recovery failed", lambda::_1))
        .onDiscarded(lambda::bind(fail, "Recovery failed", "discarded"));
    } else {
      // This happens if there is a ZK blip that causes a re-election
      // but the same leading master is elected as leader.
      LOG(INFO) << "Re-elected as the leading master";
    }
  }

  // Keep detecting.
  detector->detect(leader)
    .onAny(defer(self(), &Master::detected, lambda::_1));
}

  bool elected() const
  {
    return leader.isSome() && leader.get() == info_;
  }

~~~
开始就竞选Master和检测谁是活的Master，如果以前是Master但是现在不是了，直接退出，卧槽！
如果我们运行三个Master的话，不会退出的！这里还是需要分析一下。



##Slave注册

~~~cpp

void Master::initialize()
{
  ......
  
  //从这里可以看出，RegisterSlaveMessage消息是由registerSlave处理的。
  
    install<RegisterSlaveMessage>(
      &Master::registerSlave,
      &RegisterSlaveMessage::slave,
      &RegisterSlaveMessage::checkpointed_resources,
      &RegisterSlaveMessage::version);
      
  ......
}
~~~
首先我们可以看出SlaveID是怎么生成的。

~~~cpp
SlaveID Master::newSlaveId()
{
  SlaveID slaveId;
  slaveId.set_value(info_.id() + "-S" + stringify(nextSlaveId++));
  return slaveId;
}
~~~

然后开始分析注册流程：

~~~cpp
void Master::registerSlave(
    const UPID& from,
    const SlaveInfo& slaveInfo,
    const vector<Resource>& checkpointedResources,
    const string& version)
{
  ++metrics->messages_register_slave;

...一系列的验证措施

  slaves.registering.insert(from);

  // Create and add the slave id.
  SlaveInfo slaveInfo_ = slaveInfo;
  slaveInfo_.mutable_id()->CopyFrom(newSlaveId());

  LOG(INFO) << "Registering slave at " << from << " ("
            << slaveInfo.hostname() << ") with id " << slaveInfo_.id();

  registrar->apply(Owned<Operation>(new AdmitSlave(slaveInfo_)))
    .onAny(defer(self(),
                 &Self::_registerSlave,
                 slaveInfo_,
                 from,
                 checkpointedResources,
                 version,
                 lambda::_1));
}
~~~

主要的逻辑在RegistrarProcess::apply里面

~~~cpp

Future<bool> RegistrarProcess::apply(Owned<Operation> operation)
{
  if (recovered.isNone()) {
    return Failure("Attempted to apply the operation before recovering");
  }

  return recovered.get()->future()
    .then(defer(self(), &Self::_apply, operation));
}
~~~

先不管 recovered的逻辑，正常的逻辑肯定是都recovered了的，所以会调用_apply

~~~cpp

Future<bool> RegistrarProcess::_apply(Owned<Operation> operation)
{
  if (error.isSome()) {
    return Failure(error.get());
  }

  CHECK_SOME(variable);

  operations.push_back(operation);
  Future<bool> future = operation->future();
  if (!updating) {
    update();
  }
  return future;
}
~~~

然后会调用RegistrarProcess::update()

~~~cpp
void RegistrarProcess::update()
{
...

  updating = true;

  // Create a snapshot of the current registry.
  Registry registry = variable.get().get();

  // Create the 'slaveIDs' accumulator.
  hashset<SlaveID> slaveIDs;
  foreach (const Registry::Slave& slave, registry.slaves().slaves()) {
    slaveIDs.insert(slave.info().id());
  }

  foreach (Owned<Operation> operation, operations) {
    // No need to process the result of the operation.
    (*operation)(&registry, &slaveIDs, flags.registry_strict);
  }

  LOG(INFO) << "Applied " << operations.size() << " operations in "
            << stopwatch.elapsed() << "; attempting to update the 'registry'";

  // Perform the store, and time the operation.
  metrics.state_store.start();
  state->store(variable.get().mutate(registry))
    .after(flags.registry_store_timeout,
           lambda::bind(
               &timeout<Option<Variable<Registry> > >,
               "store",
               flags.registry_store_timeout,
               lambda::_1))
    .onAny(defer(self(), &Self::_update, lambda::_1, operations));

  // Clear the operations, _update will transition the Promises!
  operations.clear();
}
~~~


注意上面的第二个foreach那里，其实是调用Operation的operator()函数，这个函数又会调用子类的perform函数

~~~cpp

class Operation : public process::Promise<bool>
{
public:
  Operation() : success(false) {}
  virtual ~Operation() {}

  Try<bool> operator()(
      Registry* registry,
      hashset<SlaveID>* slaveIDs,
      bool strict)
  {
    const Try<bool> result = perform(registry, slaveIDs, strict);

    success = !result.isError();

    return result;
  }

  // Sets the promise based on whether the operation was successful.
  bool set() { return process::Promise<bool>::set(success); }

protected:
  virtual Try<bool> perform(
      Registry* registry,
      hashset<SlaveID>* slaveIDs,
      bool strict) = 0;

private:
  bool success;	//注意，每个动作都会返回成功与否标志
};
~~~

所以我们需要继续查看AdmitSlave::perform

~~~cpp

class AdmitSlave : public Operation
{
public:
  explicit AdmitSlave(const SlaveInfo& _info) : info(_info)
  {
    CHECK(info.has_id()) << "SlaveInfo is missing the 'id' field";
  }

protected:
  virtual Try<bool> perform(
      Registry* registry,
      hashset<SlaveID>* slaveIDs,
      bool strict)
  {
    // Check and see if this slave already exists.
    if (slaveIDs->contains(info.id())) {
      if (strict) {
        return Error("Slave already admitted");
      } else {
        return false; // No mutation.
      }
    }

    Registry::Slave* slave = registry->mutable_slaves()->add_slaves();
    slave->mutable_info()->CopyFrom(info);
    slaveIDs->insert(info.id());
    return true; // Mutation.
  }

private:
  const SlaveInfo info;
};

~~~

这里其实就是把SlaveInfo写入到了Registry里面去，这是一个PB的对象。
这里返回true，其实没有意义。重新查看
RegistrarProcess::update() 
有一行会调用：

~~~cpp
  state->store(variable.get().mutate(registry))
    .after(flags.registry_store_timeout,
           lambda::bind(
               &timeout<Option<Variable<Registry> > >,
               "store",
               flags.registry_store_timeout,
               lambda::_1))
    .onAny(defer(self(), &Self::_update, lambda::_1, operations));

void RegistrarProcess::_update(
    const Future<Option<Variable<Registry> > >& store,
    deque<Owned<Operation> > applied)
{
  updating = false;

  ...
  variable = store.get().get();

  // Remove the operations.
  while (!applied.empty()) {
    Owned<Operation> operation = applied.front();
    applied.pop_front();

    operation->set();
  }

  if (!operations.empty()) {
    update();
  }
}
~~~

这里会调用operation->set();
这就把这个Future给设置好了。

回过头来再看 Master::registerSlave(

~~~cpp
registrar->apply(Owned<Operation>(new AdmitSlave(slaveInfo_)))
    .onAny(defer(self(),
                 &Self::_registerSlave,
                 slaveInfo_,
                 from,
                 checkpointedResources,
                 version,
                 lambda::_1));
~~~    

这里无论认证成功还是失败，会调用_registerSlave函数，所以最终的逻辑就是_registerSlave这个函数

~~~cpp
void Master::_registerSlave(
    const SlaveInfo& slaveInfo,
    const UPID& pid,
    const vector<Resource>& checkpointedResources,
    const string& version,
    const Future<bool>& admit)
{
  slaves.registering.erase(pid);

  CHECK(!admit.isDiscarded());

  if (admit.isFailed()) {
    LOG(FATAL) << "Failed to admit slave " << slaveInfo.id() << " at " << pid
               << " (" << slaveInfo.hostname() << "): " << admit.failure();
  } else if (!admit.get()) {
    // This means the slave is already present in the registrar, it's
    // likely we generated a duplicate slave id!
    LOG(ERROR) << "Slave " << slaveInfo.id() << " at " << pid
               << " (" << slaveInfo.hostname() << ") was not admitted, "
               << "asking to shut down";
    slaves.removed.put(slaveInfo.id(), Nothing());

    ShutdownMessage message;
    message.set_message(
        "Slave attempted to register but got duplicate slave id " +
        stringify(slaveInfo.id()));
    send(pid, message);
  } else {
    MachineID machineId;
    machineId.set_hostname(slaveInfo.hostname());
    machineId.set_ip(stringify(pid.address.ip));

    Slave* slave = new Slave(
        slaveInfo,
        pid,
        machineId,
        version,
        Clock::now(),
        checkpointedResources);

    ++metrics->slave_registrations;

    addSlave(slave);

    Duration pingTimeout =
      flags.slave_ping_timeout * flags.max_slave_ping_timeouts;
    MasterSlaveConnection connection;
    connection.set_total_ping_timeout_seconds(pingTimeout.secs());

    SlaveRegisteredMessage message;
    message.mutable_slave_id()->CopyFrom(slave->id);
    message.mutable_connection()->CopyFrom(connection);
    send(slave->pid, message);

    LOG(INFO) << "Registered slave " << *slave
              << " with " << slave->info.resources();
  }
}
~~~

如果认证成功，addSlave，向Slave发送SlaveRegisteredMessage消息。
在这个函数里，link就是Master向Slave发起网络连接，同时启动SlaveObserver
这个Process。、从下面的代码可以看到，已经完成的，正在进行的，都有可能会重复进行注册？、
SlaveObserver其实就是心跳了。

~~~cpp

void Master::addSlave(
    Slave* slave,
    const vector<Archive::Framework>& completedFrameworks)
{
  CHECK_NOTNULL(slave);

  slaves.removed.erase(slave->id);
  slaves.registered.put(slave);

  link(slave->pid);

  // Map the slave to the machine it is running on.
  CHECK(!machines[slave->machineId].slaves.contains(slave->id));
  machines[slave->machineId].slaves.insert(slave->id);

  // Set up an observer for the slave.
  slave->observer = new SlaveObserver(
      slave->pid,
      slave->info,
      slave->id,
      self(),
      slaves.limiter,
      metrics,
      flags.slave_ping_timeout,
      flags.max_slave_ping_timeouts);

  spawn(slave->observer);

  // Add the slave's executors to the frameworks.
  foreachkey (const FrameworkID& frameworkId, slave->executors) {
    foreachvalue (const ExecutorInfo& executorInfo,
                  slave->executors[frameworkId]) {
      Framework* framework = getFramework(frameworkId);
      if (framework != NULL) { // The framework might not be re-registered yet.
        framework->addExecutor(slave->id, executorInfo);
      }
    }
  }

  // Add the slave's tasks to the frameworks.
  foreachkey (const FrameworkID& frameworkId, slave->tasks) {
    foreachvalue (Task* task, slave->tasks[frameworkId]) {
      Framework* framework = getFramework(task->framework_id());
      if (framework != NULL) { // The framework might not be re-registered yet.
        framework->addTask(task);
      } else {
        // TODO(benh): We should really put a timeout on how long we
        // keep tasks running on a slave that never have frameworks
        // reregister and claim them.
        LOG(WARNING) << "Possibly orphaned task " << task->task_id()
                     << " of framework " << task->framework_id()
                     << " running on slave " << *slave;
      }
    }
  }

  // Re-add completed tasks reported by the slave.
  // Note that a slave considers a framework completed when it has no
  // tasks/executors running for that framework. But a master considers a
  // framework completed when the framework is removed after a failover timeout.
  // TODO(vinod): Reconcile the notion of a completed framework across the
  // master and slave.
  foreach (const Archive::Framework& completedFramework, completedFrameworks) {
    Framework* framework = getFramework(
        completedFramework.framework_info().id());

    foreach (const Task& task, completedFramework.tasks()) {
      if (framework != NULL) {
        VLOG(2) << "Re-adding completed task " << task.task_id()
                << " of framework " << *framework
                << " that ran on slave " << *slave;
        framework->addCompletedTask(task);
      } else {
        // We could be here if the framework hasn't registered yet.
        // TODO(vinod): Revisit these semantics when we store frameworks'
        // information in the registrar.
        LOG(WARNING) << "Possibly orphaned completed task " << task.task_id()
                     << " of framework " << task.framework_id()
                     << " that ran on slave " << *slave;
      }
    }
  }

  CHECK(machines.contains(slave->machineId));

  // Only set unavailability if the protobuf has one set.
  Option<Unavailability> unavailability = None();
  if (machines[slave->machineId].info.has_unavailability()) {
    unavailability = machines[slave->machineId].info.unavailability();
  }

  allocator->addSlave(
      slave->id,
      slave->info,
      unavailability,
      slave->totalResources,
      slave->usedResources);
}
~~~

~~~cpp
class SlaveObserver : public ProtobufProcess<SlaveObserver>
{
public:
  SlaveObserver(const UPID& _slave,
                const SlaveInfo& _slaveInfo,
                const SlaveID& _slaveId,
                const PID<Master>& _master,
                const Option<shared_ptr<RateLimiter>>& _limiter,
                const shared_ptr<Metrics> _metrics,
                const Duration& _slavePingTimeout,
                const size_t _maxSlavePingTimeouts)
    : ProcessBase(process::ID::generate("slave-observer")),
      slave(_slave),
      slaveInfo(_slaveInfo),
      slaveId(_slaveId),
      master(_master),
      limiter(_limiter),
      metrics(_metrics),
      slavePingTimeout(_slavePingTimeout),
      maxSlavePingTimeouts(_maxSlavePingTimeouts),
      timeouts(0),
      pinged(false),
      connected(true)
  {
    install<PongSlaveMessage>(&SlaveObserver::pong);
  }

  void reconnect()
  {
    connected = true;
  }

  void disconnect()
  {
    connected = false;
  }

protected:
  virtual void initialize()
  {
    ping();
  }

  void ping()
  {
    PingSlaveMessage message;
    message.set_connected(connected);
    send(slave, message);

    pinged = true;
    delay(slavePingTimeout, self(), &SlaveObserver::timeout);
  }

  void pong()
  {
    timeouts = 0;
    pinged = false;

    // Cancel any pending shutdown.
    if (shuttingDown.isSome()) {
      // Need a copy for non-const access.
      Future<Nothing> future = shuttingDown.get();
      future.discard();
    }
  }

  void timeout()
  {
    if (pinged) {
      timeouts++; // No pong has been received before the timeout.
      if (timeouts >= maxSlavePingTimeouts) {
        // No pong has been received for the last
        // 'maxSlavePingTimeouts' pings.
        shutdown();
      }
    }

    // NOTE: We keep pinging even if we schedule a shutdown. This is
    // because if the slave eventually responds to a ping, we can
    // cancel the shutdown.
    ping();
  }

  // NOTE: The shutdown of the slave is rate limited and can be
  // canceled if a pong was received before the actual shutdown is
  // called.
  void shutdown()
  {
    if (shuttingDown.isSome()) {
      return;  // Shutdown is already in progress.
    }

    Future<Nothing> acquire = Nothing();

    if (limiter.isSome()) {
      LOG(INFO) << "Scheduling shutdown of slave " << slaveId
                << " due to health check timeout";

      acquire = limiter.get()->acquire();
    }

    shuttingDown = acquire.onAny(defer(self(), &Self::_shutdown));
    ++metrics->slave_shutdowns_scheduled;
  }

  void _shutdown()
  {
    CHECK_SOME(shuttingDown);

    const Future<Nothing>& future = shuttingDown.get();

    CHECK(!future.isFailed());

    if (future.isReady()) {
      LOG(INFO) << "Shutting down slave " << slaveId
                << " due to health check timeout";

      ++metrics->slave_shutdowns_completed;

      dispatch(master,
               &Master::shutdownSlave,
               slaveId,
               "health check timed out");
    } else if (future.isDiscarded()) {
      LOG(INFO) << "Canceling shutdown of slave " << slaveId
                << " since a pong is received!";

      ++metrics->slave_shutdowns_canceled;
    }

    shuttingDown = None();
  }

private:
  const UPID slave;
  const SlaveInfo slaveInfo;
  const SlaveID slaveId;
  const PID<Master> master;
  const Option<shared_ptr<RateLimiter>> limiter;
  shared_ptr<Metrics> metrics;
  Option<Future<Nothing>> shuttingDown;
  const Duration slavePingTimeout;
  const size_t maxSlavePingTimeouts;
  uint32_t timeouts;
  bool pinged;
  bool connected;
};
~~~


SlaveObserver在初始化的时候就是进行ping，ping发的是PingSlaveMessage
消息，如果超时时间后，就出发超时处理，这里可以看看异步的超时机制是怎么处理的。

delay(slavePingTimeout, self(), &SlaveObserver::timeout);
如果收到PongSlaveMessage则处理的多么细致。如果这时候快要到了ShutDown的关头了，
这时候把Shutdown取消，则可以缓解一下这种状况的发生。

如果超过了maxSlavePingTimeouts则要关掉这个Slave，这时候会调用 Master::shutdownSlave，原因是health check timed out

回过头来看看Master::shutdownSlave是怎么处理的。

~~~cpp
void Master::shutdownSlave(const SlaveID& slaveId, const string& message)
{
  if (!slaves.registered.contains(slaveId)) {
    // Possible when the SlaveObserver dispatched to shutdown a slave,
    // but exited() was already called for this slave.
    LOG(WARNING) << "Unable to shutdown unknown slave " << slaveId;
    return;
  }

  Slave* slave = slaves.registered.get(slaveId);
  CHECK_NOTNULL(slave);

  LOG(WARNING) << "Shutting down slave " << *slave << " with message '"
               << message << "'";

  ShutdownMessage message_;
  message_.set_message(message);
  send(slave->pid, message_);

  removeSlave(slave, message, metrics->slave_removals_reason_unhealthy);
}
~~~

这里就是发送ShutdownMessage，这时候有可能是网络问题，但是Slave还是活的好好的！
但是还是必须相应ShutdownMessage这个事件。



removeSlave会做好多工作：
1.
  foreachkey (const FrameworkID& frameworkId, utils::copy(slave->tasks)) {
    foreachvalue (Task* task, utils::copy(slave->tasks[frameworkId])) {
    
首先看看Master是怎么保存Slave的:按照FrameworkID，然后每个FrameworkID下面有Executors
后每个FrameworkID下面有tasks

每个FrameworkID的每个TaskID都会生成一个StatusUpdate消息，然后给每个Task发送
      updateTask(task, update);
      
给每个Task发送的是          TASK_LOST,
          TaskStatus::SOURCE_MASTER,
          这两个值。
      removeTask(task);
      
然后会removeExecutor

~~~cpp
  foreachkey (const FrameworkID& frameworkId, utils::copy(slave->executors)) {
    foreachkey (const ExecutorID& executorId,
                utils::copy(slave->executors[frameworkId])) {
      removeExecutor(slave, frameworkId, executorId);
~~~
      
 然后会结束掉心跳
 
~~~cpp     
 // Kill the slave observer.
  terminate(slave->observer);
  wait(slave->observer);
  delete slave->observer;
~~~
  
  然后会调用RemoveSlave这个动作。  这个动作成功之后会调用_removeSlave
  forward(update, UPID(), framework);这里需要好好分析。
  最后还会给每个framework发送LostSlaveMessage
  
~~~cpp
  foreachvalue (Framework* framework, frameworks.registered) {
    LOG(INFO) << "Notifying framework " << *framework << " of lost slave "
              << slaveInfo.id() << " (" << slaveInfo.hostname() << ") "
              << "after recovering";
    LostSlaveMessage message;
    message.mutable_slave_id()->MergeFrom(slaveInfo.id());
    framework->send(message);
  }
~~~

  最后还会调用钩子函数    HookManager::masterSlaveLostHook(slaveInfo);
  
  
**master.hpp:**

~~~cpp
struct Slave
{
// Executors running on this slave.
  hashmap<FrameworkID, hashmap<ExecutorID, ExecutorInfo>> executors;

  // Tasks present on this slave.
  // TODO(bmahler): The task pointer ownership complexity arises from the fact
  // that we own the pointer here, but it's shared with the Framework struct.
  // We should find a way to eliminate this.
  hashmap<FrameworkID, hashmap<TaskID, Task*>> tasks;

}

void Master::removeSlave(
    Slave* slave,
    const string& message,
    Option<Counter> reason)
{
  CHECK_NOTNULL(slave);

  LOG(INFO) << "Removing slave " << *slave << ": " << message;

  // We want to remove the slave first, to avoid the allocator
  // re-allocating the recovered resources.
  //
  // NOTE: Removing the slave is not sufficient for recovering the
  // resources in the allocator, because the "Sorters" are updated
  // only within recoverResources() (see MESOS-621). The calls to
  // recoverResources() below are therefore required, even though
  // the slave is already removed.
  allocator->removeSlave(slave->id);

  // Transition the tasks to lost and remove them, BUT do not send
  // updates. Rather, build up the updates so that we can send them
  // after the slave is removed from the registry.
  vector<StatusUpdate> updates;
  foreachkey (const FrameworkID& frameworkId, utils::copy(slave->tasks)) {
    foreachvalue (Task* task, utils::copy(slave->tasks[frameworkId])) {
      const StatusUpdate& update = protobuf::createStatusUpdate(
          task->framework_id(),
          task->slave_id(),
          task->task_id(),
          TASK_LOST,
          TaskStatus::SOURCE_MASTER,
          None(),
          "Slave " + slave->info.hostname() + " removed: " + message,
          TaskStatus::REASON_SLAVE_REMOVED,
          (task->has_executor_id() ?
              Option<ExecutorID>(task->executor_id()) : None()));

      updateTask(task, update);
      removeTask(task);

      updates.push_back(update);
    }
  }

  // Remove executors from the slave for proper resource accounting.
  foreachkey (const FrameworkID& frameworkId, utils::copy(slave->executors)) {
    foreachkey (const ExecutorID& executorId,
                utils::copy(slave->executors[frameworkId])) {
      removeExecutor(slave, frameworkId, executorId);
    }
  }

  foreach (Offer* offer, utils::copy(slave->offers)) {
    // TODO(vinod): We don't need to call 'Allocator::recoverResources'
    // once MESOS-621 is fixed.
    allocator->recoverResources(
        offer->framework_id(), slave->id, offer->resources(), None());

    // Remove and rescind offers.
    removeOffer(offer, true); // Rescind!
  }

  // Remove inverse offers because sending them for a slave that is
  // gone doesn't make sense.
  foreach (InverseOffer* inverseOffer, utils::copy(slave->inverseOffers)) {
    // We don't need to update the allocator because we've already called
    // `RemoveSlave()`.
    // Remove and rescind inverse offers.
    removeInverseOffer(inverseOffer, true); // Rescind!
  }

  // Mark the slave as being removed.
  slaves.removing.insert(slave->id);
  slaves.registered.remove(slave);
  slaves.removed.put(slave->id, Nothing());
  authenticated.erase(slave->pid);

  // Remove the slave from the `machines` mapping.
  CHECK(machines.contains(slave->machineId));
  CHECK(machines[slave->machineId].slaves.contains(slave->id));
  machines[slave->machineId].slaves.erase(slave->id);

  // Kill the slave observer.
  terminate(slave->observer);
  wait(slave->observer);
  delete slave->observer;

  // TODO(benh): unlink(slave->pid);

  // Remove this slave from the registrar. Once this is completed, we
  // can forward the LOST task updates to the frameworks and notify
  // all frameworks that this slave was lost.
  registrar->apply(Owned<Operation>(new RemoveSlave(slave->info)))
    .onAny(defer(self(),
                 &Self::_removeSlave,
                 slave->info,
                 updates,
                 lambda::_1,
                 message,
                 reason));

  delete slave;
}


void Master::_removeSlave(
    const SlaveInfo& slaveInfo,
    const vector<StatusUpdate>& updates,
    const Future<bool>& removed,
    const string& message,
    Option<Counter> reason)
{
  slaves.removing.erase(slaveInfo.id());

  CHECK(!removed.isDiscarded());

  if (removed.isFailed()) {
    LOG(FATAL) << "Failed to remove slave " << slaveInfo.id()
               << " (" << slaveInfo.hostname() << ")"
               << " from the registrar: " << removed.failure();
  }

  CHECK(removed.get())
    << "Slave " << slaveInfo.id() << " (" << slaveInfo.hostname() << ") "
    << "already removed from the registrar";

  LOG(INFO) << "Removed slave " << slaveInfo.id() << " ("
            << slaveInfo.hostname() << "): " << message;

  ++metrics->slave_removals;

  if (reason.isSome()) {
    ++utils::copy(reason.get()); // Remove const.
  }

  // Forward the LOST updates on to the framework.
  foreach (const StatusUpdate& update, updates) {
    Framework* framework = getFramework(update.framework_id());

    if (framework == NULL) {
      LOG(WARNING) << "Dropping update " << update << " from unknown framework "
                   << update.framework_id();
    } else {
      forward(update, UPID(), framework);
    }
  }

  // Notify all frameworks of the lost slave.
  foreachvalue (Framework* framework, frameworks.registered) {
    LOG(INFO) << "Notifying framework " << *framework << " of lost slave "
              << slaveInfo.id() << " (" << slaveInfo.hostname() << ") "
              << "after recovering";
    LostSlaveMessage message;
    message.mutable_slave_id()->MergeFrom(slaveInfo.id());
    framework->send(message);
  }

  // Finally, notify the `SlaveLost` hooks.
  if (HookManager::hooksAvailable()) {
    HookManager::masterSlaveLostHook(slaveInfo);
  }
}
~~~

TASK_LOST肯定就是true啦
这里只是更新内存里面的各种状态了，全是清楚内存里面的标记，没有网络动作。都是本地操作

~~~cpp
bool isTerminalState(const TaskState& state)
{
  return (state == TASK_FINISHED ||
          state == TASK_FAILED ||
          state == TASK_KILLED ||
          state == TASK_LOST ||
          state == TASK_ERROR);
}



void Master::updateTask(Task* task, const StatusUpdate& update)
{
  CHECK_NOTNULL(task);

  // Get the unacknowledged status.
  const TaskStatus& status = update.status();

  // NOTE: Refer to comments on `StatusUpdate` message in messages.proto for
  // the difference between `update.latest_state()` and `status.state()`.

  // Updates from the slave have 'latest_state' set.
  Option<TaskState> latestState;
  if (update.has_latest_state()) {
    latestState = update.latest_state();
  }

  // Set 'terminated' to true if this is the first time the task
  // transitioned to terminal state. Also set the latest state.
  bool terminated;
  if (latestState.isSome()) {
    terminated = !protobuf::isTerminalState(task->state()) &&
                 protobuf::isTerminalState(latestState.get());

    // If the task has already transitioned to a terminal state,
    // do not update its state.
    if (!protobuf::isTerminalState(task->state())) {
      task->set_state(latestState.get());
    }
  } else {
    terminated = !protobuf::isTerminalState(task->state()) &&
                 protobuf::isTerminalState(status.state());

    // If the task has already transitioned to a terminal state, do not update
    // its state. Note that we are being defensive here because this should not
    // happen unless there is a bug in the master code.
    if (!protobuf::isTerminalState(task->state())) {
      task->set_state(status.state());
    }
  }

  // Set the status update state and uuid for the task. Note that
  // master-generated updates are terminal and do not have a uuid
  // (in which case the master also calls `removeTask()`).
  if (update.has_uuid()) {
    task->set_status_update_state(status.state());
    task->set_status_update_uuid(update.uuid());
  }

  // TODO(brenden) Consider wiping the `message` field?
  if (task->statuses_size() > 0 &&
      task->statuses(task->statuses_size() - 1).state() == status.state()) {
    task->mutable_statuses()->RemoveLast();
  }
  task->add_statuses()->CopyFrom(status);

  // Delete data (maybe very large since it's stored by on-top framework) we
  // are not interested in to avoid OOM.
  // For example: mesos-master is running on a machine with 4GB free memory,
  // if every task stores 10MB data into TaskStatus, then mesos-master will be
  // killed by OOM killer after have 400 tasks finished.
  // MESOS-1746.
  task->mutable_statuses(task->statuses_size() - 1)->clear_data();

  LOG(INFO) << "Updating the state of task " << task->task_id()
            << " of framework " << task->framework_id()
            << " (latest state: " << task->state()
            << ", status update state: " << status.state() << ")";

  // Once the task becomes terminal, we recover the resources.
  if (terminated) {
    allocator->recoverResources(
        task->framework_id(),
        task->slave_id(),
        task->resources(),
        None());

    // The slave owns the Task object and cannot be NULL.
    Slave* slave = slaves.registered.get(task->slave_id());
    CHECK_NOTNULL(slave);

    slave->taskTerminated(task);

    Framework* framework = getFramework(task->framework_id());
    if (framework != NULL) {
      framework->taskTerminated(task);
    }

    switch (status.state()) {
      case TASK_FINISHED: ++metrics->tasks_finished; break;
      case TASK_FAILED:   ++metrics->tasks_failed;   break;
      case TASK_KILLED:   ++metrics->tasks_killed;   break;
      case TASK_LOST:     ++metrics->tasks_lost;     break;
      case TASK_ERROR:    ++metrics->tasks_error;    break;
      default:                                       break;
    }

    if (status.has_reason()) {
      metrics->incrementTasksStates(
          status.state(),
          status.source(),
          status.reason());
    }
  }
}


void Master::removeTask(Task* task)
{
  CHECK_NOTNULL(task);

  // The slave owns the Task object and cannot be NULL.
  Slave* slave = slaves.registered.get(task->slave_id());
  CHECK_NOTNULL(slave);

  if (!protobuf::isTerminalState(task->state())) {
    LOG(WARNING) << "Removing task " << task->task_id()
                 << " with resources " << task->resources()
                 << " of framework " << task->framework_id()
                 << " on slave " << *slave
                 << " in non-terminal state " << task->state();

    // If the task is not terminal, then the resources have
    // not yet been recovered.
    allocator->recoverResources(
        task->framework_id(),
        task->slave_id(),
        task->resources(),
        None());
  } else {
    LOG(INFO) << "Removing task " << task->task_id()
              << " with resources " << task->resources()
              << " of framework " << task->framework_id()
              << " on slave " << *slave;
  }

  // Remove from framework.
  Framework* framework = getFramework(task->framework_id());
  if (framework != NULL) { // A framework might not be re-connected yet.
    framework->removeTask(task);
  }

  // Remove from slave.
  slave->removeTask(task);

  delete task;
}


void Master::removeExecutor(
    Slave* slave,
    const FrameworkID& frameworkId,
    const ExecutorID& executorId)
{
  CHECK_NOTNULL(slave);
  CHECK(slave->hasExecutor(frameworkId, executorId));

  ExecutorInfo executor = slave->executors[frameworkId][executorId];

  LOG(INFO) << "Removing executor '" << executorId
            << "' with resources " << executor.resources()
            << " of framework " << frameworkId << " on slave " << *slave;

  allocator->recoverResources(
    frameworkId, slave->id, executor.resources(), None());

  Framework* framework = getFramework(frameworkId);
  if (framework != NULL) { // The framework might not be re-registered yet.
    framework->removeExecutor(slave->id, executorId);
  }

  slave->removeExecutor(frameworkId, executorId);
}

~~~

##Slave启动
然后我们再来看看RegisterSlaveMessage Slave是什么时候发出的。

slave/slave.cpp

~~~cpp
void Slave::initialize()
{
  // Do recovery.
  async(&state::recover, metaDir, flags.strict)
    .then(defer(self(), &Slave::recover, lambda::_1))
    .then(defer(self(), &Slave::_recover))
    .onAny(defer(self(), &Slave::__recover, lambda::_1));
}

void Slave::__recover(const Future<Nothing>& future)
{
......
  if (flags.recover == "reconnect") {
    state = DISCONNECTED;

    // Start detecting masters.
    detection = detector->detect()
      .onAny(defer(self(), &Slave::detected, lambda::_1));

    ......
  }
  ......
 }
 
 
 void Slave::detected(const Future<Option<MasterInfo>>& _master)
{
    latest = _master.get();
    master = UPID(_master.get().get().pid());

    LOG(INFO) << "New master detected at " << master.get();
    
    //检测到Master后，就建立连接。
    link(master.get());

    // Wait for a random amount of time before authentication or
    // registration.
    Duration duration =
      flags.registration_backoff_factor * ((double) ::random() / RAND_MAX);

    if (credential.isSome()) {
      authenticate();
    } else {
      delay(duration,
            self(),
            &Slave::doReliableRegistration,
            flags.registration_backoff_factor * 2); // Backoff.
    }
  }

  // Keep detecting masters.
  LOG(INFO) << "Detecting new master";
  detection = detector->detect(latest)
    .onAny(defer(self(), &Slave::detected, lambda::_1));
}

void Slave::authenticate()
{
   authenticating =
    authenticatee->authenticate(master.get(), self(), credential.get())
      .onAny(defer(self(), &Self::_authenticate));

//这里可以看出，认证超过五秒，就认为超时了。
  delay(Seconds(5), self(), &Self::authenticationTimeout, authenticating.get());
}


void Slave::_authenticate()
{
  ...
    // Proceed with registration.
  doReliableRegistration(flags.registration_backoff_factor * 2);
}

~~~

从上面逻辑可以看出，无论需要认证还是不需要认证都会调用doReliableRegistration函数

~~~cpp
void Slave::doReliableRegistration(Duration maxBackoff)
{
  if (!info.has_id()) {
    // Registering for the first time.
    RegisterSlaveMessage message;
    message.set_version(MESOS_VERSION);
    message.mutable_slave()->CopyFrom(info);

    // Include checkpointed resources.
    message.mutable_checkpointed_resources()->CopyFrom(checkpointedResources);

    send(master.get(), message);
  } else {
    // Re-registering, so send tasks running.
    ReregisterSlaveMessage message;
    message.set_version(MESOS_VERSION);

    // Include checkpointed resources.
    message.mutable_checkpointed_resources()->CopyFrom(checkpointedResources);

    message.mutable_slave()->CopyFrom(info);

    foreachvalue (Framework* framework, frameworks) {
      // TODO(bmahler): We need to send the executors for these
      // pending tasks, and we need to send exited events if they
      // cannot be launched: MESOS-1715 MESOS-1720.

      typedef hashmap<TaskID, TaskInfo> TaskMap;
      foreachvalue (const TaskMap& tasks, framework->pending) {
        foreachvalue (const TaskInfo& task, tasks) {
          message.add_tasks()->CopyFrom(protobuf::createTask(
              task, TASK_STAGING, framework->id()));
        }
      }

      foreachvalue (Executor* executor, framework->executors) {
        // Add launched, terminated, and queued tasks.
        // Note that terminated executors will only have terminated
        // unacknowledged tasks.
        // Note that for each task the latest state and status update
        // state (if any) is also included.
        foreach (Task* task, executor->launchedTasks.values()) {
          message.add_tasks()->CopyFrom(*task);
        }

        foreach (Task* task, executor->terminatedTasks.values()) {
          message.add_tasks()->CopyFrom(*task);
        }

        foreach (const TaskInfo& task, executor->queuedTasks.values()) {
          message.add_tasks()->CopyFrom(protobuf::createTask(
              task, TASK_STAGING, framework->id()));
        }

        // Do not re-register with Command Executors because the
        // master doesn't store them; they are generated by the slave.
        if (executor->isCommandExecutor()) {
          // NOTE: We have to unset the executor id here for the task
          // because the master uses the absence of task.executor_id()
          // to detect command executors.
          for (int i = 0; i < message.tasks_size(); ++i) {
            message.mutable_tasks(i)->clear_executor_id();
          }
        } else {
          // Ignore terminated executors because they do not consume
          // any resources.
          if (executor->state != Executor::TERMINATED) {
            ExecutorInfo* executorInfo = message.add_executor_infos();
            executorInfo->MergeFrom(executor->info);

            // Scheduler Driver will ensure the framework id is set in
            // ExecutorInfo, effectively making it a required field.
            CHECK(executorInfo->has_framework_id());
          }
        }
      }
    }

    // Add completed frameworks.
    foreach (const Owned<Framework>& completedFramework, completedFrameworks) {
      VLOG(1) << "Reregistering completed framework "
                << completedFramework->id();

      Archive::Framework* completedFramework_ =
        message.add_completed_frameworks();

      completedFramework_->mutable_framework_info()->CopyFrom(
          completedFramework->info);

      if (completedFramework->pid.isSome()) {
        completedFramework_->set_pid(completedFramework->pid.get());
      }

      foreach (const Owned<Executor>& executor,
               completedFramework->completedExecutors) {
        VLOG(2) << "Reregistering completed executor '" << executor->id
                << "' with " << executor->terminatedTasks.size()
                << " terminated tasks, " << executor->completedTasks.size()
                << " completed tasks";

        foreach (const Task* task, executor->terminatedTasks.values()) {
          VLOG(2) << "Reregistering terminated task " << task->task_id();
          completedFramework_->add_tasks()->CopyFrom(*task);
        }

        foreach (const std::shared_ptr<Task>& task, executor->completedTasks) {
          VLOG(2) << "Reregistering completed task " << task->task_id();
          completedFramework_->add_tasks()->CopyFrom(*task);
        }
      }
    }

    CHECK_SOME(master);
    send(master.get(), message);
  }

  // Bound the maximum backoff by 'REGISTER_RETRY_INTERVAL_MAX'.
  maxBackoff = std::min(maxBackoff, REGISTER_RETRY_INTERVAL_MAX);

  // Determine the delay for next attempt by picking a random
  // duration between 0 and 'maxBackoff'.
  Duration delay = maxBackoff * ((double) ::random() / RAND_MAX);

  VLOG(1) << "Will retry registration in " << delay << " if necessary";

  // Backoff.
  process::delay(delay, self(), &Slave::doReliableRegistration, maxBackoff * 2);
}
~~~

从上面可以看出，如果不需要恢复的话，则程序直接发送RegisterSlaveMessage消息给Master了。
如果是恢复的话，则需要进行恢复后发送ReregisterSlaveMessage逻辑。这个逻辑后面我们再详细论述。


然后再看Slave怎么处理SlaveRegisteredMessage消息的：

~~~cpp
void Slave::initialize()
{
  ...
  // Install protobuf handlers.
  install<SlaveRegisteredMessage>(
      &Slave::registered,
      &SlaveRegisteredMessage::slave_id,
      &SlaveRegisteredMessage::connection);
      
    install<PingSlaveMessage>(
      &Slave::ping,
      &PingSlaveMessage::connected);
  ...
}
~~~

~~~cpp
void Slave::registered(
    const UPID& from,
    const SlaveID& slaveId,
    const MasterSlaveConnection& connection)
{
  //判断一下这里的Slave是不是和ZK上面的一致。
  if (master != from) {
    LOG(WARNING) << "Ignoring registration message from " << from
                 << " because it is not the expected master: "
                 << (master.isSome() ? stringify(master.get()) : "None");
    return;
  }

  CHECK_SOME(master);

  if (connection.has_total_ping_timeout_seconds()) {
    masterPingTimeout = Seconds(connection.total_ping_timeout_seconds());
  } else {
    masterPingTimeout = DEFAULT_MASTER_PING_TIMEOUT();
  }

  switch (state) {
    case DISCONNECTED: {
      LOG(INFO) << "Registered with master " << master.get()
                << "; given slave ID " << slaveId;

      // TODO(bernd-mesos): Make this an instance method call, see comment
      // in "fetcher.hpp"".
      Try<Nothing> recovered = Fetcher::recover(slaveId, flags);
      if (recovered.isError()) {
        LOG(FATAL) << "Could not initialize fetcher cache: "
                   << recovered.error();
      }

      state = RUNNING;

      statusUpdateManager->resume(); // Resume status updates.

      info.mutable_id()->CopyFrom(slaveId); // Store the slave id.

      // Create the slave meta directory.
      paths::createSlaveDirectory(metaDir, slaveId);

      // Checkpoint slave info.
      const string path = paths::getSlaveInfoPath(metaDir, slaveId);

      VLOG(1) << "Checkpointing SlaveInfo to '" << path << "'";
      CHECK_SOME(state::checkpoint(path, info));

      // If we don't get a ping from the master, trigger a
      // re-registration. This needs to be done once registered,
      // in case we never receive an initial ping.
      Clock::cancel(pingTimer);

      pingTimer = delay(
          masterPingTimeout,
          self(),
          &Slave::pingTimeout,
          detection);

      break;
    }
    case RUNNING:
      // Already registered!
      if (!(info.id() == slaveId)) {
       EXIT(1) << "Registered but got wrong id: " << slaveId
               << "(expected: " << info.id() << "). Committing suicide";
      }
      LOG(WARNING) << "Already registered with master " << master.get();

      break;
    case TERMINATING:
      LOG(WARNING) << "Ignoring registration because slave is terminating";
      break;
    case RECOVERING:
    default:
      LOG(FATAL) << "Unexpected slave state " << state;
      break;
  }

  // Send the latest estimate for oversubscribed resources.
  if (oversubscribedResources.isSome()) {
    LOG(INFO) << "Forwarding total oversubscribed resources "
              << oversubscribedResources.get();

    UpdateSlaveMessage message;
    message.mutable_slave_id()->CopyFrom(info.id());
    message.mutable_oversubscribed_resources()->CopyFrom(
        oversubscribedResources.get());

    send(master.get(), message);
  }
}

~~~

这个逻辑很简单：如果注册成功了，就启动statusUpdateManager->resume()不断的更新状态，同时启动一个定时器，检测指定时间内是否收到了Master的Ping消息。同时发送UpdateSlaveMessage消息，汇报自己的资源？

看detection的定义：

~~~cpp
//slave/slave.hpp

 // Master detection future.
  process::Future<Option<MasterInfo>> detection;
~~~


~~~cpp
void Slave::pingTimeout(Future<Option<MasterInfo>> future)
{
  // It's possible that a new ping arrived since the timeout fired
  // and we were unable to cancel this timeout. If this occurs, don't
  // bother trying to re-detect.
  if (pingTimer.timeout().expired()) {
    LOG(INFO) << "No pings from master received within "
              << masterPingTimeout;

    future.discard();
  }
}
~~~

如果超时，这里的future就是上面的detection，这时候表示测试Master失败，所以就把这个Master丢弃掉。

回过头来再看，这个future的回调函数是：

~~~cpp
  detection = detector->detect(latest)
    .onAny(defer(self(), &Slave::detected, lambda::_1));

~~~
那么这个函数针对Discarded状态处理规则就可以了：


~~~cpp
void Slave::detected(const Future<Option<MasterInfo>>& _master)
{
  ...
  Option<MasterInfo> latest;

  if (_master.isDiscarded()) {
    LOG(INFO) << "Re-detecting master";
    latest = None();
    master = None();
  } 

  // Keep detecting masters.
  LOG(INFO) << "Detecting new master";
  detection = detector->detect(latest)
    .onAny(defer(self(), &Slave::detected, lambda::_1));
    
  ...
}
~~~

看到没有，接收Maser Ping超时，就会把master置空，然后重新选择Master。


~~~cpp
void Slave::ping(const UPID& from, bool connected)
{
  VLOG(1) << "Received ping from " << from;

  if (!connected && state == RUNNING) {
    // This could happen if there is a one way partition between
    // the master and slave, causing the master to get an exited
    // event and marking the slave disconnected but the slave
    // thinking it is still connected. Force a re-registration with
    // the master to reconcile.
    LOG(INFO) << "Master marked the slave as disconnected but the slave"
              << " considers itself registered! Forcing re-registration.";
    detection.discard();
  }

  // If we don't get a ping from the master, trigger a
  // re-registration. This can occur when the master no
  // longer considers the slave to be registered, so it is
  // essential for the slave to attempt a re-registration
  // when this occurs.
  Clock::cancel(pingTimer);

  pingTimer = delay(
      masterPingTimeout,
      self(),
      &Slave::pingTimeout,
      detection);

  send(from, PongSlaveMessage());
}
~~~

ping会处理网络延迟情况。以及发送 PongSlaveMessage 。


##Slave状态更新
下面我们会看一下 StatusUpdateManager 这个类：

~~~cpp
slave/main.cpp
int main(int argc, char** argv)
{
 ...
 StatusUpdateManager statusUpdateManager(flags);
 
  Slave* slave = new Slave(
   ...
      &statusUpdateManager,
   ...);
}
~~~

~~~cpp
// StatusUpdateManager is responsible for
// 1) Reliably sending status updates to the master.
// 2) Checkpointing the update to disk (optional).
// 3) Sending ACKs to the executor (optional).
// 4) Receiving ACKs from the scheduler.
class StatusUpdateManager
{
public:
  StatusUpdateManager(const Flags& flags);
  virtual ~StatusUpdateManager();

  // Expects a callback 'forward' which gets called whenever there is
  // a new status update that needs to be forwarded to the master.
  void initialize(const lambda::function<void(StatusUpdate)>& forward);

  // TODO(vinod): Come up with better names/signatures for the
  // checkpointing and non-checkpointing 'update()' functions.
  // Currently, it is not obvious that one version of 'update()'
  // does checkpointing while the other doesn't.

  // Checkpoints the status update and reliably sends the
  // update to the master (and hence the scheduler).
  // @return Whether the update is handled successfully
  // (e.g. checkpointed).
  process::Future<Nothing> update(
      const StatusUpdate& update,
      const SlaveID& slaveId,
      const ExecutorID& executorId,
      const ContainerID& containerId);

  // Retries the update to the master (as long as the slave is
  // alive), but does not checkpoint the update.
  // @return Whether the update is handled successfully.
  process::Future<Nothing> update(
      const StatusUpdate& update,
      const SlaveID& slaveId);

  // Checkpoints the status update to disk if necessary.
  // Also, sends the next pending status update, if any.
  // @return True if the ACK is handled successfully (e.g., checkpointed)
  //              and the task's status update stream is not terminated.
  //         False same as above except the status update stream is terminated.
  //         Failed if there are any errors (e.g., duplicate, checkpointing).
  process::Future<bool> acknowledgement(
      const TaskID& taskId,
      const FrameworkID& frameworkId,
      const UUID& uuid);

  // Recover status updates.
  process::Future<Nothing> recover(
      const std::string& rootDir,
      const Option<state::SlaveState>& state);


  // Pause sending updates.
  // This is useful when the slave is disconnected because a
  // disconnected slave will drop the updates.
  void pause();

  // Unpause and resend all the pending updates right away.
  // This is useful when the updates were pending because there was
  // no master elected (e.g., during recovery) or framework failed over.
  void resume();

  // Closes all the status update streams corresponding to this framework.
  // NOTE: This stops retrying any pending status updates for this framework.
  void cleanup(const FrameworkID& frameworkId);

private:
  StatusUpdateManagerProcess* process;
};
~~~

从上面可以看出 StatusUpdateManager 有四个功能：
1. 向Master发送Update消息
2. 将更新的状态checkpoint到本地磁盘
3. 向Executor发送ACK
4. 接收scheduler的ACK

从initialize可以看出，这里需要绑定一个函数，他的说明是把消息转发到Master，看下面这段代码：

~~~cpp
void Slave::initialize()
{
   ...
   statusUpdateManager->initialize(defer(self(), &Slave::forward, lambda::_1));
   
 }
 
 // NOTE: An acknowledgement for this update might have already been
// processed by the slave but not the status update manager.
void Slave::forward(StatusUpdate update)
{
  CHECK(state == RECOVERING || state == DISCONNECTED ||
        state == RUNNING || state == TERMINATING)
    << state;

  if (state != RUNNING) {
    LOG(WARNING) << "Dropping status update " << update
                 << " sent by status update manager because the slave"
                 << " is in " << state << " state";
    return;
  }

  // Ensure that task status uuid is set even if this update was sent by the
  // status update manager after recovering a pre 0.23.x slave/executor driver's
  // updates. This allows us to simplify the master code (in >= 0.27.0) to
  // assume the uuid is always set for retryable updates.
  CHECK(update.has_uuid())
    << "Expecting updates without 'uuid' to have been rejected";

  update.mutable_status()->set_uuid(update.uuid());

  // Update the status update state of the task and include the latest
  // state of the task in the status update.
  Framework* framework = getFramework(update.framework_id());
  if (framework != NULL) {
    const TaskID& taskId = update.status().task_id();
    Executor* executor = framework->getExecutor(taskId);
    if (executor != NULL) {
      // NOTE: We do not look for the task in queued tasks because
      // no update is expected for it until it's launched. Similarly,
      // we do not look for completed tasks because the state for a
      // completed task shouldn't be changed.
      Task* task = NULL;
      if (executor->launchedTasks.contains(taskId)) {
        task = executor->launchedTasks[taskId];
      } else if (executor->terminatedTasks.contains(taskId)) {
        task = executor->terminatedTasks[taskId];
      }

      if (task != NULL) {
        // We set the status update state of the task here because in
        // steady state master updates the status update state of the
        // task when it receives this update. If the master fails over,
        // slave re-registers with this task in this status update
        // state. Note that an acknowledgement for this update might
        // be enqueued on status update manager when we are here. But
        // that is ok because the status update state will be updated
        // when the next update is forwarded to the slave.
        task->set_status_update_state(update.status().state());
        task->set_status_update_uuid(update.uuid());

        // Include the latest state of task in the update. See the
        // comments in 'statusUpdate()' on why informing the master
        // about the latest state of the task is important.
        update.set_latest_state(task->state());
      }
    }
  }

  CHECK_SOME(master);
  LOG(INFO) << "Forwarding the update " << update << " to " << master.get();

  // NOTE: We forward the update even if framework/executor/task
  // doesn't exist because the status update manager will be expecting
  // an acknowledgement for the update. This could happen for example
  // if this is a retried terminal update and before we are here the
  // slave has already processed the acknowledgement of the original
  // update and removed the framework/executor/task. Also, slave
  // re-registration can generate updates when framework/executor/task
  // are unknown.

  // Forward the update to master.
  StatusUpdateMessage message;
  message.mutable_update()->MergeFrom(update);
  message.set_pid(self()); // The ACK will be first received by the slave.

  send(master.get(), message);
}

~~~

很简单，这里就是经过一系列检查之后，将StatusUpdateMessage发送到Master。

~~~cpp
void StatusUpdateManager::initialize(
    const function<void(StatusUpdate)>& forward)
{
  dispatch(process, &StatusUpdateManagerProcess::initialize, forward);
}

~~~
这里可以看到dispatch是怎么用的。



~~~cpp
StatusUpdateStream::StatusUpdateStream(
    const TaskID& _taskId,
    const FrameworkID& _frameworkId,
    const SlaveID& _slaveId,
    const Flags& _flags,
    bool _checkpoint,
    const Option<ExecutorID>& executorId,
    const Option<ContainerID>& containerId)
    : checkpoint(_checkpoint),
      terminated(false),
      taskId(_taskId),
      frameworkId(_frameworkId),
      slaveId(_slaveId),
      flags(_flags),
      error(None())
{
  if (checkpoint) {
    CHECK_SOME(executorId);
    CHECK_SOME(containerId);

    path = paths::getTaskUpdatesPath(
        paths::getMetaRootDir(flags.work_dir),
        slaveId,
        frameworkId,
        executorId.get(),
        containerId.get(),
        taskId);

    // Create the base updates directory, if it doesn't exist.
    Try<Nothing> directory = os::mkdir(Path(path.get()).dirname());
    if (directory.isError()) {
      error = "Failed to create " + Path(path.get()).dirname();
      return;
    }

    // Open the updates file.
    Try<int> result = os::open(
        path.get(),
        O_CREAT | O_WRONLY | O_APPEND | O_SYNC | O_CLOEXEC,
        S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);

    if (result.isError()) {
      error = "Failed to open '" + path.get() + "' for status updates";
      return;
    }

    // We keep the file open through the lifetime of the task, because it
    // makes it easy to append status update records to the file.
    fd = result.get();
  }
}

~~~

从上面可以看出，StatusUpdateStream的存放目录是：
{work_dir}/meta

##StatusUpdateStream

~~~cpp
void StatusUpdateManagerProcess::resume()
{
  LOG(INFO) << "Resuming sending status updates";
  paused = false;

  foreachkey (const FrameworkID& frameworkId, streams) {
    foreachvalue (StatusUpdateStream* stream, streams[frameworkId]) {
      if (!stream->pending.empty()) {
        const StatusUpdate& update = stream->pending.front();
        LOG(WARNING) << "Resending status update " << update;
        stream->timeout = forward(update, STATUS_UPDATE_RETRY_INTERVAL_MIN);
      }
    }
  }
}
~~~

无论是恢复过来的还是后续进来的，都会转发到Master那里去。



~~~cpp
Future<Nothing> StatusUpdateManager::update(
    const StatusUpdate& update,
    const SlaveID& slaveId)
{
  return dispatch(
      process,
      &StatusUpdateManagerProcess::update,
      update,
      slaveId);
}


Future<bool> StatusUpdateManager::acknowledgement(
    const TaskID& taskId,
    const FrameworkID& frameworkId,
    const UUID& uuid)
{
  return dispatch(
      process,
      &StatusUpdateManagerProcess::acknowledgement,
      taskId,
      frameworkId,
      uuid);
}
~~~
上面的两个函数一定有地方调用，因为这两个函数最后都会调用到

~~~cpp
Try<Nothing> StatusUpdateStream::handle(
    const StatusUpdate& update,
    const StatusUpdateRecord::Type& type)
{
  CHECK_NONE(error);

  // Checkpoint the update if necessary.
  if (checkpoint) {
    LOG(INFO) << "Checkpointing " << type << " for status update " << update;

    CHECK_SOME(fd);

    StatusUpdateRecord record;
    record.set_type(type);

    if (type == StatusUpdateRecord::UPDATE) {
      record.mutable_update()->CopyFrom(update);
    } else {
      record.set_uuid(update.uuid());
    }

    Try<Nothing> write = ::protobuf::write(fd.get(), record);
    if (write.isError()) {
      error = "Failed to write status update " + stringify(update) +
              " to '" + path.get() + "': " + write.error();
      return Error(error.get());
    }
  }

  // Now actually handle the update.
  _handle(update, type);

  return Nothing();
}


void StatusUpdateStream::_handle(
    const StatusUpdate& update,
    const StatusUpdateRecord::Type& type)
{
  CHECK_NONE(error);

  if (type == StatusUpdateRecord::UPDATE) {
    // Record this update.
    received.insert(UUID::fromBytes(update.uuid()));

    // Add it to the pending updates queue.
    pending.push(update);
  } else {
    // Record this ACK.
    acknowledged.insert(UUID::fromBytes(update.uuid()));

    // Remove the corresponding update from the pending queue.
    pending.pop();

    if (!terminated) {
      terminated = protobuf::isTerminalState(update.status().state());
    }
  }
}

~~~
上面就是往pending里面push状态的！。


果然是！看

~~~cpp
void Slave::initialize()
{
  install<StatusUpdateAcknowledgementMessage>(
      &Slave::statusUpdateAcknowledgement,
      &StatusUpdateAcknowledgementMessage::slave_id,
      &StatusUpdateAcknowledgementMessage::framework_id,
      &StatusUpdateAcknowledgementMessage::task_id,
      &StatusUpdateAcknowledgementMessage::uuid);
}

void Slave::statusUpdateAcknowledgement(
    const UPID& from,
    const SlaveID& slaveId,
    const FrameworkID& frameworkId,
    const TaskID& taskId,
    const string& uuid)
{
  // Originally, all status update acknowledgements were sent from the
  // scheduler driver. We'd like to have all acknowledgements sent by
  // the master instead. See: MESOS-1389.
  // For now, we handle acknowledgements from the leading master and
  // from the scheduler driver, for backwards compatibility.
  // TODO(bmahler): Aim to have the scheduler driver no longer
  // sending acknowledgements in 0.20.0. Stop handling those messages
  // here in 0.21.0.
  // NOTE: We must reject those acknowledgements coming from
  // non-leading masters because we may have already sent the terminal
  // un-acknowledged task to the leading master! Unfortunately, the
  // master's pid will not change across runs on the same machine, so
  // we may process a message from the old master on the same machine,
  // but this is a more general problem!
  if (strings::startsWith(from.id, "master")) {
    if (state != RUNNING) {
      LOG(WARNING) << "Dropping status update acknowledgement message for "
                   << frameworkId << " because the slave is in "
                   << state << " state";
      return;
    }

    if (master != from) {
      LOG(WARNING) << "Ignoring status update acknowledgement message from "
                   << from << " because it is not the expected master: "
                   << (master.isSome() ? stringify(master.get()) : "None");
      return;
    }
  }

  statusUpdateManager->acknowledgement(
      taskId, frameworkId, UUID::fromBytes(uuid))
    .onAny(defer(self(),
                 &Slave::_statusUpdateAcknowledgement,
                 lambda::_1,
                 taskId,
                 frameworkId,
                 UUID::fromBytes(uuid)));
}


void Slave::_statusUpdateAcknowledgement(
    const Future<bool>& future,
    const TaskID& taskId,
    const FrameworkID& frameworkId,
    const UUID& uuid)
{
  // The future could fail if this is a duplicate status update acknowledgement.
  if (!future.isReady()) {
    LOG(ERROR) << "Failed to handle status update acknowledgement (UUID: "
               << uuid << ") for task " << taskId
               << " of framework " << frameworkId << ": "
               << (future.isFailed() ? future.failure() : "future discarded");
    return;
  }

  VLOG(1) << "Status update manager successfully handled status update"
          << " acknowledgement (UUID: " << uuid
          << ") for task " << taskId
          << " of framework " << frameworkId;

  CHECK(state == RECOVERING || state == DISCONNECTED ||
        state == RUNNING || state == TERMINATING)
    << state;

  Framework* framework = getFramework(frameworkId);
  if (framework == NULL) {
    LOG(ERROR) << "Status update acknowledgement (UUID: " << uuid
               << ") for task " << taskId
               << " of unknown framework " << frameworkId;
    return;
  }

  CHECK(framework->state == Framework::RUNNING ||
        framework->state == Framework::TERMINATING)
    << framework->state;

  // Find the executor that has this update.
  Executor* executor = framework->getExecutor(taskId);
  if (executor == NULL) {
    LOG(ERROR) << "Status update acknowledgement (UUID: " << uuid
               << ") for task " << taskId
               << " of unknown executor";
    return;
  }

  CHECK(executor->state == Executor::REGISTERING ||
        executor->state == Executor::RUNNING ||
        executor->state == Executor::TERMINATING ||
        executor->state == Executor::TERMINATED)
    << executor->state;

  // If the task has reached terminal state and all its updates have
  // been acknowledged, mark it completed.
  if (executor->terminatedTasks.contains(taskId) && !future.get()) {
    executor->completeTask(taskId);
  }

  // Remove the executor if it has terminated and there are no more
  // incomplete tasks.
  if (executor->state == Executor::TERMINATED && !executor->incompleteTasks()) {
    removeExecutor(framework, executor);
  }

  // Remove this framework if it has no pending executors and tasks.
  if (framework->executors.empty() && framework->pending.empty()) {
    removeFramework(framework);
  }
}
~~~

然后我们就要看看 StatusUpdateAcknowledgementMessage 这个消息是什么时候谁发送给谁的？！！
上面的就是StatusUpdateAcknowledgementMessage的收尾了！



看下面的函数，只有这里会发送 StatusUpdateAcknowledgementMessage 消息

~~~cpp

void Master::acknowledge(
    Framework* framework,
    const scheduler::Call::Acknowledge& acknowledge)
{
  CHECK_NOTNULL(framework);

  metrics->messages_status_update_acknowledgement++;

  const SlaveID slaveId = acknowledge.slave_id();
  const TaskID taskId = acknowledge.task_id();
  const UUID uuid = UUID::fromBytes(acknowledge.uuid());

  Slave* slave = slaves.registered.get(slaveId);

  if (slave == NULL) {
    LOG(WARNING)
      << "Cannot send status update acknowledgement " << uuid
      << " for task " << taskId << " of framework " << *framework
      << " to slave " << slaveId << " because slave is not registered";
    metrics->invalid_status_update_acknowledgements++;
    return;
  }

  if (!slave->connected) {
    LOG(WARNING)
      << "Cannot send status update acknowledgement " << uuid
      << " for task " << taskId << " of framework " << *framework
      << " to slave " << *slave << " because slave is disconnected";
    metrics->invalid_status_update_acknowledgements++;
    return;
  }

  LOG(INFO) << "Processing ACKNOWLEDGE call " << uuid << " for task " << taskId
            << " of framework " << *framework << " on slave " << slaveId;

  Task* task = slave->getTask(framework->id(), taskId);

  if (task != NULL) {
    // Status update state and uuid should be either set or unset
    // together.
    CHECK_EQ(task->has_status_update_uuid(), task->has_status_update_state());

    if (!task->has_status_update_state()) {
      // Task should have status update state set because it must have
      // been set when the update corresponding to this
      // acknowledgement was processed by the master. But in case this
      // acknowledgement was intended for the old run of the master
      // and the task belongs to a 0.20.0 slave, we could be here.
      // Dropping the acknowledgement is safe because the slave will
      // retry the update, at which point the master will set the
      // status update state.
      LOG(ERROR)
        << "Ignoring status update acknowledgement " << uuid
        << " for task " << taskId << " of framework " << *framework
        << " to slave " << *slave << " because the update was not"
        << " sent by this master";
      metrics->invalid_status_update_acknowledgements++;
      return;
    }

    // Remove the task once the terminal update is acknowledged.
    if (protobuf::isTerminalState(task->status_update_state()) &&
        UUID::fromBytes(task->status_update_uuid()) == uuid) {
      removeTask(task);
     }
  }

  StatusUpdateAcknowledgementMessage message;
  message.mutable_slave_id()->CopyFrom(slaveId);
  message.mutable_framework_id()->CopyFrom(framework->id());
  message.mutable_task_id()->CopyFrom(taskId);
  message.set_uuid(uuid.toBytes());

  send(slave->pid, message);

  metrics->valid_status_update_acknowledgements++;
}
~~~

只有下面这个函数调用上面这个函数

~~~cpp

void Master::statusUpdateAcknowledgement(
    const UPID& from,
    const SlaveID& slaveId,
    const FrameworkID& frameworkId,
    const TaskID& taskId,
    const string& uuid)
{
  // TODO(bmahler): Consider adding a message validator abstraction
  // for the master that takes care of all this boilerplate. Ideally
  // by the time we process messages in the critical master code, we
  // can assume that they are valid. This will become especially
  // important as validation logic is moved out of the scheduler
  // driver and into the master.

  Framework* framework = getFramework(frameworkId);

  if (framework == NULL) {
    LOG(WARNING)
      << "Ignoring status update acknowledgement " << UUID::fromBytes(uuid)
      << " for task " << taskId << " of framework " << frameworkId
      << " on slave " << slaveId << " because the framework cannot be found";
    metrics->invalid_status_update_acknowledgements++;
    return;
  }

  if (framework->pid != from) {
    LOG(WARNING)
      << "Ignoring status update acknowledgement " << UUID::fromBytes(uuid)
      << " for task " << taskId << " of framework " << *framework
      << " on slave " << slaveId << " because it is not expected from " << from;
    metrics->invalid_status_update_acknowledgements++;
    return;
  }

  scheduler::Call::Acknowledge message;
  message.mutable_slave_id()->CopyFrom(slaveId);
  message.mutable_task_id()->CopyFrom(taskId);
  message.set_uuid(uuid);

  acknowledge(framework, message);
}

~~~

~~~cpp
void Master::initialize()
{
  install<StatusUpdateAcknowledgementMessage>(
      &Master::statusUpdateAcknowledgement,
      &StatusUpdateAcknowledgementMessage::slave_id,
      &StatusUpdateAcknowledgementMessage::framework_id,
      &StatusUpdateAcknowledgementMessage::task_id,
      &StatusUpdateAcknowledgementMessage::uuid);
}
~~~


下面是Slave调用StatusUpdateAcknowledgementMessage的逻辑。
这两句话很重要
// 1) When a status update from the executor is received.
// 2) When slave generates task updates (e.g LOST/KILLED/FAILED).

看来凡是接受到StatusUpdateMessage的消息的，都要回一个StatusUpdateAcknowledgementMessage消息。

Master,Slave,exec,Sched都会发这个消息！。后面继续讨论。

find . -name "*.cpp" -type f|xargs grep StatusUpdateMessage|grep -v tests


~~~cpp
// This can be called in two ways:
// 1) When a status update from the executor is received.
// 2) When slave generates task updates (e.g LOST/KILLED/FAILED).
// NOTE: We set the pid in 'Slave::___statusUpdate()' to 'pid' so that
// whoever sent this update will get an ACK. This is important because
// we allow executors to send updates for tasks that belong to other
// executors. Currently we allow this because we cannot guarantee
// reliable delivery of status updates. Since executor driver caches
// unacked updates it is important that whoever sent the update gets
// acknowledgement for it.
void Slave::statusUpdate(StatusUpdate update, const Option<UPID>& pid)
{
  LOG(INFO) << "Handling status update " << update
            << (pid.isSome() ? " from " + stringify(pid.get()) : "");

  CHECK(state == RECOVERING || state == DISCONNECTED ||
        state == RUNNING || state == TERMINATING)
    << state;

  if (!update.has_uuid()) {
    LOG(WARNING) << "Ignoring status update " << update << " without 'uuid'";
    metrics.invalid_status_updates++;
    return;
  }

  // TODO(bmahler): With the HTTP API, we must validate the UUID
  // inside the TaskStatus. For now, we ensure that the uuid of task
  // status matches the update's uuid, in case the executor is using
  // pre 0.23.x driver.
  update.mutable_status()->set_uuid(update.uuid());

  // Set the source and UUID before forwarding the status update.
  update.mutable_status()->set_source(
      pid == UPID() ? TaskStatus::SOURCE_SLAVE : TaskStatus::SOURCE_EXECUTOR);

  // Set TaskStatus.executor_id if not already set; overwrite existing
  // value if already set.
  if (update.has_executor_id()) {
    if (update.status().has_executor_id() &&
        update.status().executor_id() != update.executor_id()) {
      LOG(WARNING) << "Executor ID mismatch in status update"
                   << (pid.isSome() ? " from " + stringify(pid.get()) : "")
                   << "; overwriting received '"
                   << update.status().executor_id() << "' with expected'"
                   << update.executor_id() << "'";
    }
    update.mutable_status()->mutable_executor_id()->CopyFrom(
        update.executor_id());
  }

  Framework* framework = getFramework(update.framework_id());
  if (framework == NULL) {
    LOG(WARNING) << "Ignoring status update " << update
                 << " for unknown framework " << update.framework_id();
    metrics.invalid_status_updates++;
    return;
  }

  CHECK(framework->state == Framework::RUNNING ||
        framework->state == Framework::TERMINATING)
    << framework->state;

  // We don't send update when a framework is terminating because
  // it cannot send acknowledgements.
  if (framework->state == Framework::TERMINATING) {
    LOG(WARNING) << "Ignoring status update " << update
                 << " for terminating framework " << framework->id();
    metrics.invalid_status_updates++;
    return;
  }

  if (HookManager::hooksAvailable()) {
    // Even though the hook(s) return a TaskStatus, we only use two fields:
    // container_status and labels. Remaining fields are discarded.
    TaskStatus statusFromHooks =
      HookManager::slaveTaskStatusDecorator(
          update.framework_id(), update.status());
    if (statusFromHooks.has_labels()) {
      update.mutable_status()->mutable_labels()->CopyFrom(
          statusFromHooks.labels());
    }

    if (statusFromHooks.has_container_status()) {
      update.mutable_status()->mutable_container_status()->CopyFrom(
          statusFromHooks.container_status());
    }
  }

  const TaskStatus& status = update.status();

  Executor* executor = framework->getExecutor(status.task_id());
  if (executor == NULL) {
    LOG(WARNING)  << "Could not find the executor for "
                  << "status update " << update;
    metrics.valid_status_updates++;

    // NOTE: We forward the update here because this update could be
    // generated by the slave when the executor is unknown to it
    // (e.g., killTask(), _runTask()) or sent by an executor for a
    // task that belongs to another executor.
    // We also end up here if 1) the previous slave died after
    // checkpointing a _terminal_ update but before it could send an
    // ACK to the executor AND 2) after recovery the status update
    // manager successfully retried the update, got the ACK from the
    // scheduler and cleaned up the stream before the executor
    // re-registered. In this case, the slave cannot find the executor
    // corresponding to this task because the task has been moved to
    // 'Executor::completedTasks'.
    //
    // NOTE: We do not set the `ContainerStatus` (including the
    // `NetworkInfo` within the `ContainerStatus)  for this case,
    // because the container is unknown. We cannot use the slave IP
    // address here (for the `NetworkInfo`) since we do not know the
    // type of network isolation used for this container.
    statusUpdateManager->update(update, info.id())
      .onAny(defer(self(), &Slave::___statusUpdate, lambda::_1, update, pid));

    return;
  }

  CHECK(executor->state == Executor::REGISTERING ||
        executor->state == Executor::RUNNING ||
        executor->state == Executor::TERMINATING ||
        executor->state == Executor::TERMINATED)
    << executor->state;

  // Failing this validation on the executor driver used to cause the
  // driver to abort. Now that the validation is done by the slave, it
  // should shutdown the executor to be consistent.
  //
  // TODO(arojas): Once the HTTP API is the default, return a
  // 400 Bad Request response, indicating the reason in the body.
  if (status.source() == TaskStatus::SOURCE_EXECUTOR &&
      status.state() == TASK_STAGING) {
    LOG(ERROR) << "Received TASK_STAGING from executor " << *executor
               << " which is not allowed. Shutting down the executor";

    _shutdownExecutor(framework, executor);
    return;
  }

  // TODO(vinod): Revisit these semantics when we disallow executors
  // from sending updates for tasks that belong to other executors.
  if (pid.isSome() &&
      pid != UPID() &&
      executor->pid.isSome() &&
      executor->pid != pid) {
    LOG(WARNING) << "Received status update " << update << " from " << pid.get()
                 << " on behalf of a different executor '" << executor->id
                 << "' (" << executor->pid.get() << ")";
  }

  metrics.valid_status_updates++;

  // Before sending update, we need to retrieve the container status.
  containerizer->status(executor->containerId)
    .onAny(defer(self(),
                 &Slave::_statusUpdate,
                 update,
                 pid,
                 executor->id,
                 lambda::_1));
}


void Slave::_statusUpdate(
    StatusUpdate update,
    const Option<process::UPID>& pid,
    const ExecutorID& executorId,
    const Future<ContainerStatus>& future)
{
  ContainerStatus* containerStatus =
    update.mutable_status()->mutable_container_status();

  // There can be cases where a container is already removed from the
  // containerizer before the `status` call is dispatched to the
  // containerizer, leading to the failure of the returned `Future`.
  // In such a case we should simply not update the `ContainerStatus`
  // with the return `Future` but continue processing the
  // `StatusUpdate`.
  if (future.isReady()) {
    containerStatus->MergeFrom(future.get());

    // Fill in the container IP address with the IP from the agent
    // PID, if not already filled in.
    //
    // TODO(karya): Fill in the IP address by looking up the executor PID.
    if (containerStatus->network_infos().size() == 0) {
      NetworkInfo* networkInfo = containerStatus->add_network_infos();

      // TODO(CD): Deprecated -- Remove after 0.27.0.
      networkInfo->set_ip_address(stringify(self().address.ip));

      NetworkInfo::IPAddress* ipAddress =
        networkInfo->add_ip_addresses();
      ipAddress->set_ip_address(stringify(self().address.ip));
    }
  }


  const TaskStatus& status = update.status();

  Executor* executor = getExecutor(update.framework_id(), executorId);
  if (executor == NULL) {
    LOG(WARNING) << "Ignoring container status update for framework "
                 << update.framework_id()
                 << "for a non-existent executor";
    return;
  }

  // We set the latest state of the task here so that the slave can
  // inform the master about the latest state (via status update or
  // ReregisterSlaveMessage message) as soon as possible. Master can
  // use this information, for example, to release resources as soon
  // as the latest state of the task reaches a terminal state. This
  // is important because status update manager queues updates and
  // only sends one update per task at a time; the next update for a
  // task is sent only after the acknowledgement for the previous one
  // is received, which could take a long time if the framework is
  // backed up or is down.
  executor->updateTaskState(status);

  // Handle the task appropriately if it is terminated.
  // TODO(vinod): Revisit these semantics when we disallow duplicate
  // terminal updates (e.g., when slave recovery is always enabled).
  if (protobuf::isTerminalState(status.state()) &&
      (executor->queuedTasks.contains(status.task_id()) ||
       executor->launchedTasks.contains(status.task_id()))) {
    executor->terminateTask(status.task_id(), status);

    // Wait until the container's resources have been updated before
    // sending the status update.
    containerizer->update(executor->containerId, executor->resources)
      .onAny(defer(self(),
                   &Slave::__statusUpdate,
                   lambda::_1,
                   update,
                   pid,
                   executor->id,
                   executor->containerId,
                   executor->checkpoint));
  } else {
    // Immediately send the status update.
    __statusUpdate(None(),
                  update,
                  pid,
                  executor->id,
                  executor->containerId,
                  executor->checkpoint);
  }
}


void Slave::__statusUpdate(
    const Option<Future<Nothing>>& future,
    const StatusUpdate& update,
    const Option<UPID>& pid,
    const ExecutorID& executorId,
    const ContainerID& containerId,
    bool checkpoint)
{
  if (future.isSome() && !future->isReady()) {
    LOG(ERROR) << "Failed to update resources for container " << containerId
               << " of executor '" << executorId
               << "' running task " << update.status().task_id()
               << " on status update for terminal task, destroying container: "
               << (future->isFailed() ? future->failure() : "discarded");

    containerizer->destroy(containerId);

    Executor* executor = getExecutor(update.framework_id(), executorId);
    if (executor != NULL) {
      containerizer::Termination termination;
      termination.set_state(TASK_LOST);
      termination.add_reasons(TaskStatus::REASON_CONTAINER_UPDATE_FAILED);
      termination.set_message(
          "Failed to update resources for container: " +
          (future->isFailed() ? future->failure() : "discarded"));

      executor->pendingTermination = termination;

      // TODO(jieyu): Set executor->state to be TERMINATING.
    }
  }

  if (checkpoint) {
    // Ask the status update manager to checkpoint and reliably send the update.
    statusUpdateManager->update(update, info.id(), executorId, containerId)
      .onAny(defer(self(), &Slave::___statusUpdate, lambda::_1, update, pid));
  } else {
    // Ask the status update manager to just retry the update.
    statusUpdateManager->update(update, info.id())
      .onAny(defer(self(), &Slave::___statusUpdate, lambda::_1, update, pid));
  }
}


void Slave::___statusUpdate(
    const Future<Nothing>& future,
    const StatusUpdate& update,
    const Option<UPID>& pid)
{
  CHECK_READY(future) << "Failed to handle status update " << update;

  VLOG(1) << "Status update manager successfully handled status update "
          << update;

  if (pid == UPID()) {
    return;
  }

  StatusUpdateAcknowledgementMessage message;
  message.mutable_framework_id()->MergeFrom(update.framework_id());
  message.mutable_slave_id()->MergeFrom(update.slave_id());
  message.mutable_task_id()->MergeFrom(update.status().task_id());
  message.set_uuid(update.uuid());

  // Status update manager successfully handled the status update.
  // Acknowledge the executor, if we have a valid pid.
  if (pid.isSome()) {
    LOG(INFO) << "Sending acknowledgement for status update " << update
              << " to " << pid.get();

    send(pid.get(), message);
  } else {
    // Acknowledge the HTTP based executor.
    Framework* framework = getFramework(update.framework_id());
    if (framework == NULL) {
      LOG(WARNING) << "Ignoring sending acknowledgement for status update "
                   << update << " of unknown framework";
      return;
    }

    Executor* executor = framework->getExecutor(update.status().task_id());
    if (executor == NULL) {
      // Refer to the comments in 'statusUpdate()' on when this can
      // happen.
      LOG(WARNING) << "Ignoring sending acknowledgement for status update "
                   << update << " of unknown executor";
      return;
    }

    executor->send(message);
  }
}


~~~


