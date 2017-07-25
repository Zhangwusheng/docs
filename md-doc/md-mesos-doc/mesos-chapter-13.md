#MesosSchedulerDriver


~~~cpp

MesosSchedulerDriver::MesosSchedulerDriver(
    Scheduler* _scheduler,
    const FrameworkInfo& _framework,
    const string& _master)
  : detector(NULL),
    scheduler(_scheduler),
    framework(_framework),
    master(_master),
    process(NULL),
    status(DRIVER_NOT_STARTED),
    implicitAcknowlegements(true),
    credential(NULL),
    schedulerId("scheduler-" + UUID::random().toString())
{
  initialize();
}

~~~

这里注意一下这个参数：implicitAcknowlegements，默认会进行ack的。
schedulerId的生成规则是 scheduler-加上UUID。

紧跟着就是调用initialize函数，只有一行函数。

~~~cpp

void MesosSchedulerDriver::initialize() {
...
  local::Flags flags;
  Try<Nothing> load = flags.load("MESOS_");


  // Initialize libprocess.
  process::initialize(schedulerId);

  spawn(new VersionProcess(), true);

  latch = new Latch();

  if (framework.user().empty()) {
    Result<string> user = os::user();
    CHECK_SOME(user);

    framework.set_user(user.get());
  }

  if (framework.hostname().empty()) {
    Try<string> hostname = net::hostname();
    if (hostname.isSome()) {
      framework.set_hostname(hostname.get());
    }
  }

  // Launch a local cluster if necessary.
  Option<UPID> pid;
  if (master == "local") {
    pid = local::launch(flags);
  }


  url = pid.isSome() ? static_cast<string>(pid.get()) : master;
}

~~~

这里最主要的就是进行了libprocess的初始化逻辑，生成了Latch变量，url在生产环境中就是zk那一串字符串。


然后就是schedulerDriver->run()函数了：

~~~cpp
Status MesosSchedulerDriver::run()
{
  Status status = start();
  return status != DRIVER_RUNNING ? status : join();
}

Status MesosSchedulerDriver::start()
{
   ...
      Try<shared_ptr<MasterDetector>> detector_ = DetectorPool::get(url);
     detector = detector_.get();
 
       process = new SchedulerProcess(
          this,
          scheduler,
          framework,
          None(),
          implicitAcknowlegements,
          schedulerId,
          detector.get(),
          flags,
          &mutex,
          latch);
      spawn(process);

    return status = DRIVER_RUNNING;
  }
}
~~~

这时候我们首先应该看这个Process的initialize函数：

~~~cpp
 virtual void initialize()
  {
    install<Event>(&SchedulerProcess::receive);

    // TODO(benh): Get access to flags so that we can decide whether
    // or not to make ZooKeeper verbose.
    install<FrameworkRegisteredMessage>(
        &SchedulerProcess::registered,
        &FrameworkRegisteredMessage::framework_id,
        &FrameworkRegisteredMessage::master_info);

    install<FrameworkReregisteredMessage>(
        &SchedulerProcess::reregistered,
        &FrameworkReregisteredMessage::framework_id,
        &FrameworkReregisteredMessage::master_info);

    install<ResourceOffersMessage>(
        &SchedulerProcess::resourceOffers,
        &ResourceOffersMessage::offers,
        &ResourceOffersMessage::pids);

    install<RescindResourceOfferMessage>(
        &SchedulerProcess::rescindOffer,
        &RescindResourceOfferMessage::offer_id);

    install<StatusUpdateMessage>(
        &SchedulerProcess::statusUpdate,
        &StatusUpdateMessage::update,
        &StatusUpdateMessage::pid);

    install<LostSlaveMessage>(
        &SchedulerProcess::lostSlave,
        &LostSlaveMessage::slave_id);

    install<ExitedExecutorMessage>(
        &SchedulerProcess::lostExecutor,
        &ExitedExecutorMessage::executor_id,
        &ExitedExecutorMessage::slave_id,
        &ExitedExecutorMessage::status);

    install<ExecutorToFrameworkMessage>(
        &SchedulerProcess::frameworkMessage,
        &ExecutorToFrameworkMessage::slave_id,
        &ExecutorToFrameworkMessage::executor_id,
        &ExecutorToFrameworkMessage::data);

    install<FrameworkErrorMessage>(
        &SchedulerProcess::error,
        &FrameworkErrorMessage::message);

    // Start detecting masters.
    detector->detect()
      .onAny(defer(self(), &SchedulerProcess::detected, lambda::_1));
  }
  
~~~

从上面我们可以看处，Scheduler可以处理的消息就是上面那么多了。

首先看
SchedulerProcess::detected，Scheduler首先查找系统的master。

~~~cpp
从这里看怎么处理这种重复的逻辑，妙哉！把初始化和后续逻辑都能合在一起处理！

void detected(const Future<Option<MasterInfo> >& _master)
  {
    if (!running.load()) {
      VLOG(1) << "Ignoring the master change because the driver is not"
              << " running!";
      return;
    }

    CHECK(!_master.isDiscarded());

    if (_master.isFailed()) {
      EXIT(1) << "Failed to detect a master: " << _master.failure();
    }

    if (_master.get().isSome()) {
      master = _master.get().get();
    } else {
      master = None();
    }

    if (connected) {
      // There are three cases here:
      //   1. The master failed.
      //   2. The master failed over to a new master.
      //   3. The master failed over to the same master.
      // In any case, we will reconnect (possibly immediately), so we
      // must notify schedulers of the disconnection.
      Stopwatch stopwatch;
      if (FLAGS_v >= 1) {
        stopwatch.start();
      }

      //这里的disconnected是我们自己的scheduler的回调。
      scheduler->disconnected(driver);

      VLOG(1) << "Scheduler::disconnected took " << stopwatch.elapsed();
    }

    connected = false;

    if (master.isSome()) {
      LOG(INFO) << "New master detected at " << master.get().pid();
      link(master.get().pid());

      if (credential.isSome()) {
        // Authenticate with the master.
        // TODO(vinod): Do a backoff for authentication similar to what
        // we do for registration.
        authenticate();
      } else {
        // Proceed with registration without authentication.
        LOG(INFO) << "No credentials provided."
                  << " Attempting to register without authentication";

        // TODO(vinod): Similar to the slave add a random delay to the
        // first registration attempt too. This needs fixing tests
        // that expect scheduler to register even with clock paused
        // (e.g., rate limiting tests).
        doReliableRegistration(flags.registration_backoff_factor);
      }
    } else {
      // In this case, we don't actually invoke Scheduler::error
      // since we might get reconnected to a master imminently.
      LOG(INFO) << "No master detected";
    }

    // Keep detecting masters.
    detector->detect(_master.get())
      .onAny(defer(self(), &SchedulerProcess::detected, lambda::_1));
  }
~~~

很简单，也是建立和Master的连接，然后进行注册doReliableRegistration。
主要这里的失败处理逻辑，如果以前和Master注册过了，这时候又detected到Master变了（有可能仅仅是网络抖动），这里一律从调用disconnected，这里的处理逻辑就相对比较简单一些了。这是需要学习的地方！

      // There are three cases here:
      //   1. The master failed.
      //   2. The master failed over to a new master.
      //   3. The master failed over to the same master.
      // In any case, we will reconnect (possibly immediately), so we



~~~cpp
Call在include/mesos/v1/scheduler/scheduler.proto里面定义的。
注意这里的failover，构造函数里面是这么设置的：

failover(_framework.has_id() && !framework.id().value().empty())
就是看Framework有没有一个非空的ID。这个和故障恢复重新注册有关系。


void doReliableRegistration(Duration maxBackoff)
  {
    if (!running.load()) {
      return;
    }

    if (connected || master.isNone()) {
      return;
    }

    if (credential.isSome() && !authenticated) {
      return;
    }

    VLOG(1) << "Sending SUBSCRIBE call to " << master.get().pid();

    Call call;
    call.set_type(Call::SUBSCRIBE);

    Call::Subscribe* subscribe = call.mutable_subscribe();
    subscribe->mutable_framework_info()->CopyFrom(framework);

    if (framework.has_id() && !framework.id().value().empty()) {
      subscribe->set_force(failover);
      call.mutable_framework_id()->CopyFrom(framework.id());
    }

    send(master.get().pid(), call);

    // Bound the maximum backoff by 'REGISTRATION_RETRY_INTERVAL_MAX'.
    maxBackoff =
      std::min(maxBackoff, scheduler::REGISTRATION_RETRY_INTERVAL_MAX);

    // If failover timeout is present, bound the maximum backoff
    // by 1/10th of the failover timeout.
    if (framework.has_failover_timeout()) {
      Try<Duration> duration = Duration::create(framework.failover_timeout());
      if (duration.isSome()) {
        maxBackoff = std::min(maxBackoff, duration.get() / 10);
      }
    }

    // Determine the delay for next attempt by picking a random
    // duration between 0 and 'maxBackoff'.
    // TODO(vinod): Use random numbers from <random> header.
    Duration delay = maxBackoff * ((double) ::random() / RAND_MAX);

    VLOG(1) << "Will retry registration in " << delay << " if necessary";

    // Backoff.
    process::delay(
        delay, self(), &Self::doReliableRegistration, maxBackoff * 2);
  }
  
  
  ~~~
上面send了一个Call，这个send是在ProtobufProcess里面实现的，他把消息序列化，然后发送  

~~~cpp
void send(const process::UPID& to,
            const google::protobuf::Message& message)
  {
    std::string data;
    message.SerializeToString(&data);
    process::Process<T>::send(to, message.GetTypeName(),
                              data.data(), data.size());
  }
~~~

而上面的Call恰恰是Message的一个子类。
所以我们应该看看Master如何响应这个消息的。


下面我们看Master::initialize函数针对这个消息的处理：

~~~cpp

注意一下里面的注释：


看一下Call的PB定义：

message Call {
  enum Type {
    SUBSCRIBE = 1;   // See 'Subscribe' below.
    TEARDOWN = 2;    // Shuts down all tasks/executors and removes framework.
    ACCEPT = 3;      // See 'Accept' below.
    DECLINE = 4;     // See 'Decline' below.
    REVIVE = 5;      // Removes any previous filters set via ACCEPT or DECLINE.
    KILL = 6;        // See 'Kill' below.
    SHUTDOWN = 7;    // See 'Shutdown' below.
    ACKNOWLEDGE = 8; // See 'Acknowledge' below.
    RECONCILE = 9;   // See 'Reconcile' below.
    MESSAGE = 10;    // See 'Message' below.
    REQUEST = 11;    // See 'Request' below.
    SUPPRESS = 12;    // Inform master to stop sending offers to the framework.

  }

  // Subscribes the scheduler with the master to receive events. A
  // scheduler must send other calls only after it has received the
  // SUBCRIBED event.
  message Subscribe {
    // See the comments below on 'framework_id' on the semantics for
    // 'framework_info.id'.
    required FrameworkInfo framework_info = 1;

    // NOTE: 'force' field is not present in v1/scheduler.proto because it is
    // only used by the scheduler driver. The driver sets it to true when the
    // scheduler re-registers for the first time after a failover. Once
    // re-registered all subsequent re-registration attempts (e.g., due to ZK
    // blip) will have 'force' set to false. This is important because master
    // uses this field to know when it needs to send FrameworkRegisteredMessage
    // vs FrameworkReregisteredMessage.
    optional bool force = 2;
  }


Master::initialize(){
...
install<scheduler::Call>(&Master::receive);
...
}

void Master::receive(
    const UPID& from,
    const scheduler::Call& call)
{
  // TODO(vinod): Add metrics for calls.

  Option<Error> error = validation::scheduler::call::validate(call);

  if (error.isSome()) {
    drop(from, call, error.get().message);
    return;
  }

  if (call.type() == scheduler::Call::SUBSCRIBE) {
    subscribe(from, call.subscribe());
    return;
  }

  // We consolidate the framework lookup and pid validation logic here
  // because they are common for all the call handlers.
  Framework* framework = getFramework(call.framework_id());

  if (framework == NULL) {
    drop(from, call, "Framework cannot be found");
    return;
  }

  if (framework->pid != from) {
    drop(from, call, "Call is not from registered framework");
    return;
  }

  switch (call.type()) {
    case scheduler::Call::TEARDOWN:
      teardown(framework);
      break;

    case scheduler::Call::ACCEPT:
      accept(framework, call.accept());
      break;

    case scheduler::Call::DECLINE:
      decline(framework, call.decline());
      break;

    case scheduler::Call::REVIVE:
      revive(framework);
      break;

    case scheduler::Call::KILL:
      kill(framework, call.kill());
      break;

    case scheduler::Call::SHUTDOWN:
      shutdown(framework, call.shutdown());
      break;

    case scheduler::Call::ACKNOWLEDGE:
      acknowledge(framework, call.acknowledge());
      break;

    case scheduler::Call::RECONCILE:
      reconcile(framework, call.reconcile());
      break;

    case scheduler::Call::MESSAGE:
      message(framework, call.message());
      break;

    case scheduler::Call::REQUEST:
      request(framework, call.request());
      break;

    case scheduler::Call::SUPPRESS:
      suppress(framework);
      break;

    default:
      // Should be caught during call validation above.
      LOG(FATAL) << "Unexpected " << call.type() << " call"
                 << " from framework " << call.framework_id() << " at " << from;
      break;
  }
}
~~~
又是一套分派！


~~~cpp

void Master::subscribe(
    const UPID& from,
    const scheduler::Call::Subscribe& subscribe)
{
  const FrameworkInfo& frameworkInfo = subscribe.framework_info();

  //如果这个请求正在被认证，则将此请求缓存，等待认证完毕后回调这个函数。因为网络是什么事情都可能发生的。
  
  if (authenticating.contains(from)) {
    // TODO(vinod): Consider dropping this request and fix the tests
    // to deal with the drop. Currently there is a race between master
    // realizing the framework is authenticated and framework sending
    // a subscribe call. Dropping this message will cause the
    // framework to retry slowing down the tests.
    LOG(INFO) << "Queuing up SUBSCRIBE call for"
              << " framework '" << frameworkInfo.name() << "' at " << from
              << " because authentication is still in progress";

    // Need to disambiguate for the compiler.
    void (Master::*f)(const UPID&, const scheduler::Call::Subscribe&)
      = &Self::subscribe;

    authenticating[from]
      .onReady(defer(self(), f, from, subscribe));
    return;
  }

  Option<Error> validationError = None();

  if (validationError.isNone() && !isWhitelistedRole(frameworkInfo.role())) {
    validationError = Error("Role '" + frameworkInfo.role() + "' is not" +
                            " present in the master's --roles");
  }

  // TODO(vinod): Deprecate this in favor of authorization.
  // 判断ROOT能够提交请求。一般我们不以root用户运行
  if (validationError.isNone() &&
      frameworkInfo.user() == "root" && !flags.root_submissions) {
    validationError = Error("User 'root' is not allowed to run frameworks"
                            " without --root_submissions set");
  }

  if (validationError.isNone() && frameworkInfo.has_id()) {
    foreach (const shared_ptr<Framework>& framework, frameworks.completed) {
      if (framework->id() == frameworkInfo.id()) {
        // This could happen if a framework tries to subscribe after
        // its failover timeout has elapsed or it unregistered itself
        // by calling 'stop()' on the scheduler driver.
        //
        // TODO(vinod): Master should persist admitted frameworks to the
        // registry and remove them from it after failover timeout.
        validationError = Error("Framework has been removed");
        break;
      }
    }
  }

  // Note that re-authentication errors are already handled above.
  if (validationError.isNone()) {
    validationError = validateFrameworkAuthentication(frameworkInfo, from);
  }

  if (validationError.isSome()) {
    LOG(INFO) << "Refusing subscription of framework"
              << " '" << frameworkInfo.name() << "' at " << from << ": "
              << validationError.get().message;

    FrameworkErrorMessage message;
    message.set_message(validationError.get().message);
    send(from, message);
    return;
  }

  LOG(INFO) << "Received SUBSCRIBE call for"
            << " framework '" << frameworkInfo.name() << "' at " << from;

  // We allow an authenticated framework to not specify a principal
  // in FrameworkInfo but we'd prefer if it did so we log a WARNING
  // here when it happens.
  if (!frameworkInfo.has_principal() && authenticated.contains(from)) {
    LOG(WARNING) << "Framework at " << from
                 << " (authenticated as '" << authenticated[from] << "')"
                 << " does not set 'principal' in FrameworkInfo";
  }

  // Need to disambiguate for the compiler.
  void (Master::*_subscribe)(
      const UPID&,
      const scheduler::Call::Subscribe&,
      const Future<bool>&) = &Self::_subscribe;

  authorizeFramework(frameworkInfo)
    .onAny(defer(self(),
                 _subscribe,
                 from,
                 subscribe,
                 lambda::_1));
}



Future<bool> Master::authorizeFramework(
    const FrameworkInfo& frameworkInfo)
{
  if (authorizer.isNone()) {
    return true; // Authorization is disabled.
  }

  LOG(INFO) << "Authorizing framework principal '" << frameworkInfo.principal()
            << "' to receive offers for role '" << frameworkInfo.role() << "'";

  mesos::ACL::RegisterFramework request;
  if (frameworkInfo.has_principal()) {
    request.mutable_principals()->add_values(frameworkInfo.principal());
  } else {
    // Framework doesn't have a principal set.
    request.mutable_principals()->set_type(mesos::ACL::Entity::ANY);
  }
  request.mutable_roles()->add_values(frameworkInfo.role());

  return authorizer.get()->authorize(request);
}


~~~

这里首先要认证，我们不分析认证的过程，但是可以看看认证的代码：authorizeFramework，这里会返回成功与否的Future。认证完毕之后就会调用_subscribe这个函数，这里才是真正的重点：这里好复杂

~~~cpp

void Master::_subscribe(
    const UPID& from,
    const scheduler::Call::Subscribe& subscribe,
    const Future<bool>& authorized)
{

  //如果认证失败，那么发送FrameworkErrorMessage消息给Scheduler。
  
  const FrameworkInfo& frameworkInfo = subscribe.framework_info();

  CHECK(!authorized.isDiscarded());

  Option<Error> authorizationError = None();

  if (authorized.isFailed()) {
    authorizationError =
      Error("Authorization failure: " + authorized.failure());
  } else if (!authorized.get()) {
    authorizationError =
      Error("Not authorized to use role '" + frameworkInfo.role() + "'");
  }

  if (authorizationError.isSome()) {
    LOG(INFO) << "Refusing subscription of framework"
              << " '" << frameworkInfo.name() << "' at " << from
              << ": " << authorizationError.get().message;

    FrameworkErrorMessage message;
    message.set_message(authorizationError.get().message);

    send(from, message);
    return;
  }

  // At this point, authentications errors will be due to
  // re-authentication during the authorization process,
  // so we drop the subscription.
  Option<Error> authenticationError =
    validateFrameworkAuthentication(frameworkInfo, from);

  if (authenticationError.isSome()) {
    LOG(INFO) << "Dropping SUBSCRIBE call for framework"
              << " '" << frameworkInfo.name() << "' at " << from
              << ": " << authenticationError.get().message;
    return;
  }

  LOG(INFO) << "Subscribing framework " << frameworkInfo.name()
            << " with checkpointing "
            << (frameworkInfo.checkpoint() ? "enabled" : "disabled")
            << " and capabilities " << frameworkInfo.capabilities();

  if (!frameworkInfo.has_id() || frameworkInfo.id().value().empty()) {
    // If we are here the framework is subscribing for the first time.
    // Check if this framework is already subscribed (because it retries).
    foreachvalue (Framework* framework, frameworks.registered) {
      if (framework->pid == from) {
        LOG(INFO) << "Framework " << *framework
                  << " already subscribed, resending acknowledgement";
        FrameworkRegisteredMessage message;
        message.mutable_framework_id()->MergeFrom(framework->id());
        message.mutable_master_info()->MergeFrom(info_);
        framework->send(message);
        return;
      }
    }

    
    // Assign a new FrameworkID.
    FrameworkInfo frameworkInfo_ = frameworkInfo;
    frameworkInfo_.mutable_id()->CopyFrom(newFrameworkId());

    Framework* framework = new Framework(this, flags, frameworkInfo_, from);

    addFramework(framework);

    FrameworkRegisteredMessage message;
    message.mutable_framework_id()->MergeFrom(framework->id());
    message.mutable_master_info()->MergeFrom(info_);
    framework->send(message);

    return;
  }
  
  /////走到上面，说明是一个新的FrameWork请求，那么给Scheduler回复一个FrameworkRegisteredMessage消息，就返回了。
  
  下面的这个是重新注册的情况。
  
  看看这个解释：
    // The driver sets it to true when the
    // scheduler re-registers for the first time after a failover. Once
    // re-registered all subsequent re-registration attempts (e.g., due to ZK
    // blip) will have 'force' set to false. This is important because master
    // uses this field to know when it needs to send FrameworkRegisteredMessage
    // vs FrameworkReregisteredMessage.
 就是当失败之后的第一次注册时需要设置成true，表示强制注册。当重新注册后，以后所有的重注册都设置为false。
 啥意思？

  // If we are here framework has already been assigned an id.
  CHECK(!frameworkInfo.id().value().empty());

  if (frameworks.registered.contains(frameworkInfo.id())) {
    // Using the "force" field of the scheduler allows us to keep a
    // scheduler that got partitioned but didn't die (in ZooKeeper
    // speak this means didn't lose their session) and then
    // eventually tried to connect to this master even though
    // another instance of their scheduler has reconnected.

    Framework* framework =
      CHECK_NOTNULL(frameworks.registered[frameworkInfo.id()]);

    // Test for the error case first.
    if ((framework->pid != from) && !subscribe.force()) {
      LOG(ERROR) << "Disallowing subscription attempt of"
                 << " framework " << *framework
                 << " because it is not expected from " << from;

      FrameworkErrorMessage message;
      message.set_message("Framework failed over");
      send(from, message);
      return;
    }

    // It is now safe to update the framework fields since the request is now
    // guaranteed to be successful. We use the fields passed in during
    // re-registration.
    LOG(INFO) << "Updating info for framework " << framework->id();

    framework->updateFrameworkInfo(frameworkInfo);
    allocator->updateFramework(framework->id(), framework->info);

    framework->reregisteredTime = Clock::now();

	//从这里看出，如果是同一台机器，可以不强制，使用同一个ID进行重连，这时候不需要failoverFramework
	//如果不是同一台机器，那么如果不强制，就出错了，是上面的逻辑（因为有一个FrameworkID了，所以一定是旧的连接，从这里我们也可以看出怎么编写一个可以重连的FrameWrok来。）所以如果需要从不同的机器重连，那么一定要加上force标识。而一旦加了这个表示，那么就要调用下面的函数failoverFramework来把原来机器的资源清除掉！
    if (subscribe.force()) {
      // TODO(vinod): Now that the scheduler pid is unique we don't
      // need to call 'failoverFramework()' if the pid hasn't changed
      // (i.e., duplicate message). Instead we can just send the
      // FrameworkReregisteredMessage back and activate the framework
      // if necesssary.
      LOG(INFO) << "Framework " << *framework << " failed over";
      failoverFramework(framework, from);
    } else {
      LOG(INFO) << "Allowing framework " << *framework
                << " to subscribe with an already used id";

      // Remove any offers sent to this framework.
      // NOTE: We need to do this because the scheduler might have
      // replied to the offers but the driver might have dropped
      // those messages since it wasn't connected to the master.
      foreach (Offer* offer, utils::copy(framework->offers)) {
        allocator->recoverResources(
            offer->framework_id(),
            offer->slave_id(),
            offer->resources(),
            None());
        //这里后续再看，但是这里确实需要重新协商资源。
        removeOffer(offer, true); // Rescind.
      }

      // Also remove inverse offers.
      foreach (InverseOffer* inverseOffer,
               utils::copy(framework->inverseOffers)) {
        allocator->updateInverseOffer(
            inverseOffer->slave_id(),
            inverseOffer->framework_id(),
            UnavailableResources{
                inverseOffer->resources(),
                inverseOffer->unavailability()},
            None());

        removeInverseOffer(inverseOffer, true); // Rescind.
      }

      // TODO(bmahler): Shouldn't this re-link with the scheduler?
      framework->connected = true;

      // Reactivate the framework.
      // NOTE: We do this after recovering resources (above) so that
      // the allocator has the correct view of the framework's share.
      if (!framework->active) {
        framework->active = true;
        allocator->activateFramework(framework->id());
      }
      
      //这里一定是重新注册的消息了！重新注册，只说明这个FrameworkID被注册过了，但是在新的机器上可能是第一次注册，因为他在其他机器上出测过了！但是failoverFramework 里面也会发送FrameworkRegisteredMessage消息，所以无论是FrameworkRegisteredMessage还是FrameworkReregisteredMessage仅仅当做一种通知就可以了，最好不要做逻辑，要做也要做幂等的逻辑，因为这里会因为网络partition，重启，等因素导致这些消息被重复发送！。

      FrameworkReregisteredMessage message;
      message.mutable_framework_id()->MergeFrom(frameworkInfo.id());
      message.mutable_master_info()->MergeFrom(info_);
      framework->send(message);
      return;
    }
  } else {
    // We don't have a framework with this ID, so we must be a newly
    // elected Mesos master to which either an existing scheduler or a
    // failed-over one is connecting. Create a Framework object and add
    // any tasks it has that have been reported by reconnecting slaves.
    Framework* framework = new Framework(this, flags, frameworkInfo, from);

    // Add active tasks and executors to the framework.
    
    //当一个Slave注册到Master的时候，都会把自己加入到slaves.registered这个变量中去
    //参见Master::addSlave函数。这里只是把已有的属于这个Framework的Task和executor加入到自己里面去。这种情况不太会发生，但是在异步情况下有可能。
    
    foreachvalue (Slave* slave, slaves.registered) {
      foreachvalue (Task* task, slave->tasks[framework->id()]) {
        framework->addTask(task);
      }
      foreachvalue (const ExecutorInfo& executor,
                    slave->executors[framework->id()]) {
        framework->addExecutor(slave->id, executor);
      }
    }

    // N.B. Need to add the framework _after_ we add its tasks
    // (above) so that we can properly determine the resources it's
    // currently using!
    addFramework(framework);

    // TODO(bmahler): We have to send a registered message here for
    // the re-registering framework, per the API contract. Send
    // re-register here per MESOS-786; requires deprecation or it
    // will break frameworks.
    FrameworkRegisteredMessage message;
    message.mutable_framework_id()->MergeFrom(framework->id());
    message.mutable_master_info()->MergeFrom(info_);
    framework->send(message);
  }

  CHECK(frameworks.registered.contains(frameworkInfo.id()))
    << "Unknown framework " << frameworkInfo.id()
    << " (" << frameworkInfo.name() << ")";

  // Broadcast the new framework pid to all the slaves. We have to
  // broadcast because an executor might be running on a slave but
  // it currently isn't running any tasks.
  foreachvalue (Slave* slave, slaves.registered) {
    UpdateFrameworkMessage message;
    message.mutable_framework_id()->MergeFrom(frameworkInfo.id());
    message.set_pid(from);
    send(slave->pid, message);
  }
  
  //上面会把所有的slave都发送UpdateFrameworkMessage消息，看来如果我先注册了Framework，后面扩容的时候加入了的机器，也会给新的Salve发送这个消息。参见下面的函数！
}
比较一下addSlave和__reregisterSlave的区别，当新的slave加入的时候，已有的Framework不会加入发送到新的slave上，亦即不会发送UpdateFrameworkMessage，只有在__reregisterSlave重新注册的时候才会发送这个消息！。

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

void Master::__reregisterSlave(Slave* slave, const vector<Task>& tasks)
{
  CHECK_NOTNULL(slave);

  // Send the latest framework pids to the slave.
  hashset<FrameworkID> ids;

  foreach (const Task& task, tasks) {
    Framework* framework = getFramework(task.framework_id());

    if (framework != NULL && !ids.contains(framework->id())) {
      UpdateFrameworkMessage message;
      message.mutable_framework_id()->MergeFrom(framework->id());

      // TODO(anand): We set 'pid' to UPID() for http frameworks
      // as 'pid' was made optional in 0.24.0. In 0.25.0, we
      // no longer have to set pid here for http frameworks.
      message.set_pid(framework->pid.getOrElse(UPID()));

      send(slave->pid, message);

      ids.insert(framework->id());
    }
  }

  // NOTE: Here we always send the message. Slaves whose version are
  // less than 0.22.0 will drop it silently which is OK.
  LOG(INFO) << "Sending updated checkpointed resources "
            << slave->checkpointedResources
            << " to slave " << *slave;

  CheckpointResourcesMessage message;
  message.mutable_resources()->CopyFrom(slave->checkpointedResources);

  send(slave->pid, message);
}

void Master::failoverFramework(Framework* framework, const HttpConnection& http)
{
  // Notify the old connected framework that it has failed over.
  // This is safe to do even if it is a retry because the framework is expected
  // to close the old connection (and hence not receive any more responses)
  // before sending subscription request on a new connection.
  if (framework->connected) {
    FrameworkErrorMessage message;
    message.set_message("Framework failed over");
    framework->send(message);
  }

  // If this is an upgrade, clear the authentication related data.
  if (framework->pid.isSome()) {
    authenticated.erase(framework->pid.get());

    CHECK(frameworks.principals.contains(framework->pid.get()));
    Option<string> principal = frameworks.principals[framework->pid.get()];

    frameworks.principals.erase(framework->pid.get());

    // Remove the metrics for the principal if this framework is the
    // last one with this principal.
    if (principal.isSome() &&
        !frameworks.principals.containsValue(principal.get())) {
      CHECK(metrics->frameworks.contains(principal.get()));
      metrics->frameworks.erase(principal.get());
    }
  }

  framework->updateConnection(http);

  http.closed()
    .onAny(defer(self(), &Self::exited, framework->id(), http));

  _failoverFramework(framework);

  // Start the heartbeat after sending SUBSCRIBED event.
  framework->heartbeat();
}

void Master::_failoverFramework(Framework* framework)
{
  // Remove the framework's offers (if they weren't removed before).
  // We do this after we have updated the pid and sent the framework
  // registered message so that the allocator can immediately re-offer
  // these resources to this framework if it wants.
  foreach (Offer* offer, utils::copy(framework->offers)) {
    allocator->recoverResources(
        offer->framework_id(), offer->slave_id(), offer->resources(), None());

    removeOffer(offer);
  }

  // Also remove the inverse offers.
  foreach (InverseOffer* inverseOffer, utils::copy(framework->inverseOffers)) {
    allocator->updateInverseOffer(
        inverseOffer->slave_id(),
        inverseOffer->framework_id(),
        UnavailableResources{
            inverseOffer->resources(),
            inverseOffer->unavailability()},
        None());

    removeInverseOffer(inverseOffer);
  }

  // Reconnect and reactivate the framework.
  framework->connected = true;

  // Reactivate the framework.
  // NOTE: We do this after recovering resources (above) so that
  // the allocator has the correct view of the framework's share.
  if (!framework->active) {
    framework->active = true;
    allocator->activateFramework(framework->id());
  }

  // The scheduler driver safely ignores any duplicate registration
  // messages, so we don't need to compare the old and new pids here.
  
  //注意这里只是调用framework的send函数
  FrameworkRegisteredMessage message;
  message.mutable_framework_id()->MergeFrom(framework->id());
  message.mutable_master_info()->MergeFrom(info_);
  framework->send(message);
}

void Master::addFramework(Framework* framework)
{
  CHECK_NOTNULL(framework);

  CHECK(!frameworks.registered.contains(framework->id()))
    << "Framework " << *framework << " already exists!";

  frameworks.registered[framework->id()] = framework;

  //这里检查的好仔细！
  if (framework->pid.isSome()) {
  	//建立与Framework的连接。
    link(framework->pid.get());
  } else {
    CHECK_SOME(framework->http);

    HttpConnection http = framework->http.get();

    http.closed()
      .onAny(defer(self(), &Self::exited, framework->id(), http));
  }

  const string& role = framework->info.role();
  CHECK(isWhitelistedRole(role))
    << "Unknown role " << role
    << " of framework " << *framework;

  if (!activeRoles.contains(role)) {
    activeRoles[role] = new Role();
  }
  activeRoles[role]->addFramework(framework);

  // There should be no offered resources yet!
  CHECK_EQ(Resources(), framework->totalOfferedResources);

  allocator->addFramework(
      framework->id(),
      framework->info,
      framework->usedResources);

  // Export framework metrics.

  // If the framework is authenticated, its principal should be in
  // 'authenticated'. Otherwise look if it's supplied in
  // FrameworkInfo.
  if (framework->pid.isSome()) {
    Option<string> principal = authenticated.get(framework->pid.get());
    if (principal.isNone() && framework->info.has_principal()) {
      principal = framework->info.principal();
    }

    CHECK(!frameworks.principals.contains(framework->pid.get()));
    frameworks.principals.put(framework->pid.get(), principal);

    // Export framework metrics if a principal is specified.
    if (principal.isSome()) {
      // Create new framework metrics if this framework is the first
      // one of this principal. Otherwise existing metrics are reused.
      if (!metrics->frameworks.contains(principal.get())) {
        metrics->frameworks.put(
            principal.get(),
            Owned<Metrics::Frameworks>(
              new Metrics::Frameworks(principal.get())));
      }
    }
  }
}

~~~


Framework与Master的沟通到一段落，然后我们看看slave接收到了UpdateFrameworkMessage之后会做些什么事情。


~~~
slave.cpp
Slave::initialize()
{
  install<UpdateFrameworkMessage>(
      &Slave::updateFramework,
      &UpdateFrameworkMessage::framework_id,
      &UpdateFrameworkMessage::pid);
}


void Slave::updateFramework(
    const FrameworkID& frameworkId,
    const UPID& pid)
{
  ....
  Framework* framework = getFramework(frameworkId);
  if (framework == NULL) {
    LOG(WARNING) << "Ignoring updating pid for framework " << frameworkId
                 << " because it does not exist";
    return;
  }

  switch (framework->state) {
    case Framework::TERMINATING:
      LOG(WARNING) << "Ignoring updating pid for framework " << frameworkId
                   << " because it is terminating";
      break;
    case Framework::RUNNING: {
      LOG(INFO) << "Updating framework " << frameworkId << " pid to " << pid;

      if (pid == UPID()) {
        framework->pid = None();
      } else {
        framework->pid = pid;
      }

      if (framework->info.checkpoint()) {
        // Checkpoint the framework pid, note that when the 'pid'
        // is None, we checkpoint a default UPID() because
        // 0.23.x slaves consider a missing pid file to be an
        // error.
        const string path = paths::getFrameworkPidPath(
            metaDir, info.id(), frameworkId);

        VLOG(1) << "Checkpointing framework pid"
                << " '" << framework->pid.getOrElse(UPID()) << "'"
                << " to '" << path << "'";

        CHECK_SOME(state::checkpoint(path, framework->pid.getOrElse(UPID())));
      }

      // Inform status update manager to immediately resend any pending
      // updates.
      statusUpdateManager->resume();

      break;
    }
    default:
      LOG(FATAL) << "Framework " << framework->id()
                << " is in unexpected state " << framework->state;
      break;
  }
}
~~~

回过头来看看Master的main函数：
看看
allocator是怎么生成的。

~~~cpp
  // Create an instance of allocator.
  const string allocatorName = flags.allocator;
  Try<Allocator*> allocator = Allocator::create(allocatorName);

  if (allocator.isError()) {
    EXIT(EXIT_FAILURE)
      << "Failed to create '" << allocatorName
      << "' allocator: " << allocator.error();
  }
  
Try<Allocator*> Allocator::create(const string& name)
{
  // Create an instance of the default allocator. If other than the
  // default allocator is requested, search for it in loaded modules.
  // NOTE: We do not need an extra not-null check, because both
  // ModuleManager and built-in allocator factory do that already.
  if (name == mesos::internal::master::DEFAULT_ALLOCATOR) {
    return HierarchicalDRFAllocator::create();
  }

  return modules::ModuleManager::create<Allocator>(name);
}
  
~~~
系统默认的资源分配器是HierarchicalDRFAllocator。
因为在Maser::initialize里面进行了allocator的初始化，所以我们看看HierarchicalDRFAllocator初始化做了什么动作：

~~~cpp
master/allocator/mesos/hierarchical.hpp

typedef MesosAllocator<HierarchicalDRFAllocatorProcess>
HierarchicalDRFAllocator;


template <typename AllocatorProcess>
inline void MesosAllocator<AllocatorProcess>::initialize(
    const Duration& allocationInterval,
    const lambda::function<
        void(const FrameworkID&,
             const hashmap<SlaveID, Resources>&)>& offerCallback,
    const lambda::function<
        void(const FrameworkID&,
              const hashmap<SlaveID, UnavailableResources>&)>&
      inverseOfferCallback,
    const hashmap<std::string, double>& weights)
{
  process::dispatch(
      process,
      &MesosAllocatorProcess::initialize,
      allocationInterval,
      offerCallback,
      inverseOfferCallback,
      weights);
}
很明显这里的初始化会调用到MesosAllocatorProcess的初始化函数，而HierarchicalDRFAllocatorProcess是从这个类继承过来的，肯定会调用HierarchicalDRFAllocatorProcess::initialize函数的。

template <typename AllocatorProcess>
MesosAllocator<AllocatorProcess>::MesosAllocator()
{
  process = new AllocatorProcess();
  process::spawn(process);
}

void HierarchicalAllocatorProcess::initialize(
    const Duration& _allocationInterval,
    const lambda::function<
        void(const FrameworkID&,
             const hashmap<SlaveID, Resources>&)>& _offerCallback,
    const lambda::function<
        void(const FrameworkID&,
             const hashmap<SlaveID, UnavailableResources>&)>&
      _inverseOfferCallback,
    const hashmap<string, double>& _weights)
{
  allocationInterval = _allocationInterval;
  offerCallback = _offerCallback;
  inverseOfferCallback = _inverseOfferCallback;
  weights = _weights;
  initialized = true;
  paused = false;

  // Resources for quota'ed roles are allocated separately and prior to
  // non-quota'ed roles, hence a dedicated sorter for quota'ed roles is
  // necessary. We create an instance of the same sorter type we use for
  // all roles.
  //
  // TODO(alexr): Consider introducing a sorter type for quota'ed roles.
  roleSorter = roleSorterFactory();
  quotaRoleSorter = roleSorterFactory();

  VLOG(1) << "Initialized hierarchical allocator process";

  delay(allocationInterval, self(), &Self::batch);
}



template <typename RoleSorter, typename FrameworkSorter>
class HierarchicalAllocatorProcess
  : public internal::HierarchicalAllocatorProcess
{
public:
  HierarchicalAllocatorProcess()
    : internal::HierarchicalAllocatorProcess(
          []() -> Sorter* { return new RoleSorter(); },
          []() -> Sorter* { return new FrameworkSorter(); }) {}
};

typedef HierarchicalAllocatorProcess<DRFSorter, DRFSorter>
HierarchicalDRFAllocatorProcess;

从这里看出，roleSorter和quotaRoleSorter都是DRFSorter的一个实例。

最主要的逻辑在这一行：
delay(allocationInterval, self(), &Self::batch);

~~~

下面我们看看

~~~cpp
void HierarchicalAllocatorProcess::batch()
{
  allocate();
  delay(allocationInterval, self(), &Self::batch);
}
~~~

这就是个定时器！

~~~cpp
void HierarchicalAllocatorProcess::allocate()
{
  if (paused) {
    VLOG(1) << "Skipped allocation because the allocator is paused";

    return;
  }

  Stopwatch stopwatch;
  stopwatch.start();

  allocate(slaves.keys());

  VLOG(1) << "Performed allocation for " << slaves.size() << " slaves in "
            << stopwatch.elapsed();
}
~~~
~~~cpp
class HierarchicalAllocatorProcess : public MesosAllocatorProcess
{

hashmap<SlaveID, Slave> slaves;

}

为每台机器分配资源
void HierarchicalAllocatorProcess::allocate(
    const hashset<SlaveID>& slaveIds_)
{
  // Compute the offerable resources, per framework:
  //   (1) For reserved resources on the slave, allocate these to a
  //       framework having the corresponding role.
  //   (2) For unreserved resources on the slave, allocate these
  //       to a framework of any role.
  hashmap<FrameworkID, hashmap<SlaveID, Resources>> offerable;

  // NOTE: This function can operate on a small subset of slaves, we have to
  // make sure that we don't assume cluster knowledge when summing resources
  // from that set.

  vector<SlaveID> slaveIds;
  slaveIds.reserve(slaveIds_.size());

  //首先过滤一遍机器，看看那些是可用的资源。
  // Filter out non-whitelisted and deactivated slaves in order not to send
  // offers for them.
  foreach (const SlaveID& slaveId, slaveIds_) {
    if (isWhitelisted(slaveId) && slaves[slaveId].activated) {
      slaveIds.push_back(slaveId);
    }
  }

  // Randomize the order in which slaves' resources are allocated.
  //
  // TODO(vinod): Implement a smarter sorting algorithm.
  std::random_shuffle(slaveIds.begin(), slaveIds.end());

  // Returns the __quantity__ of resources allocated to a quota role. Since we
  // account for reservations and persistent volumes toward quota, we strip
  // reservation and persistent volume related information for comparability.
  // The result is used to determine whether a role's quota is satisfied, and
  // also to determine how many resources the role would need in order to meet
  // its quota.
  //
  // NOTE: Revocable resources are excluded in `quotaRoleSorter`.
  auto getQuotaRoleAllocatedResources = [this](const string& role) {
    CHECK(quotas.contains(role));

    // NOTE: `allocationScalarQuantities` omits dynamic reservation and
    // persistent volume info, but we additionally strip `role` here.
    Resources resources;

    foreach (Resource resource,
             quotaRoleSorter->allocationScalarQuantities(role)) {
      CHECK(!resource.has_reservation());
      CHECK(!resource.has_disk());

      resource.set_role("*");
      resources += resource;
    }

    return resources;
  };

  // Quota comes first and fair share second. Here we process only those
  // roles, for which quota is set (quota'ed roles). Such roles form a
  // special allocation group with a dedicated sorter.
  foreach (const SlaveID& slaveId, slaveIds) {
    foreach (const string& role, quotaRoleSorter->sort()) {
      CHECK(quotas.contains(role));

      // If there are no active frameworks in this role, we do not
      // need to do any allocations for this role.
      if (!activeRoles.contains(role)) {
        continue;
      }

      // Get the total quantity of resources allocated to a quota role. The
      // value omits role, reservation, and persistence info.
      Resources roleConsumedResources = getQuotaRoleAllocatedResources(role);

      // If quota for the role is satisfied, we do not need to do any further
      // allocations for this role, at least at this stage.
      //
      // TODO(alexr): Skipping satisfied roles is pessimistic. Better
      // alternatives are:
      //   * A custom sorter that is aware of quotas and sorts accordingly.
      //   * Removing satisfied roles from the sorter.
      if (roleConsumedResources.contains(quotas[role].info.guarantee())) {
        continue;
      }

      // Fetch frameworks according to their fair share.
      foreach (const string& frameworkId_, frameworkSorters[role]->sort()) {
        FrameworkID frameworkId;
        frameworkId.set_value(frameworkId_);

        // If the framework has suppressed offers, ignore. The Unallocated
        // part of the quota will not be allocated to other roles.
        if (frameworks[frameworkId].suppressed) {
          continue;
        }

        // Calculate the currently available resources on the slave.
        Resources available = slaves[slaveId].total - slaves[slaveId].allocated;

        // The resources we offer are the unreserved resources as well as the
        // reserved resources for this particular role. This is necessary to
        // ensure that we don't offer resources that are reserved for another
        // role.
        //
        // NOTE: Currently, frameworks are allowed to have '*' role.
        // Calling reserved('*') returns an empty Resources object.
        //
        // Quota is satisfied from the available non-revocable resources on the
        // agent. It's important that we include reserved resources here since
        // reserved resources are accounted towards the quota guarantee. If we
        // were to rely on stage 2 to offer them out, they would not be checked
        // against the quota guarantee.
        Resources resources =
          (available.unreserved() + available.reserved(role)).nonRevocable();

        // NOTE: The resources may not be allocatable here, but they can be
        // accepted by one of the frameworks during the second allocation
        // stage.
        if (!allocatable(resources)) {
          continue;
        }

        // If the framework filters these resources, ignore. The unallocated
        // part of the quota will not be allocated to other roles.
        if (isFiltered(frameworkId, slaveId, resources)) {
          continue;
        }

        VLOG(2) << "Allocating " << resources << " on slave " << slaveId
                << " to framework " << frameworkId
                << " as part of its role quota";

        // NOTE: We perform "coarse-grained" allocation for quota'ed
        // resources, which may lead to overcommitment of resources beyond
        // quota. This is fine since quota currently represents a guarantee.
        offerable[frameworkId][slaveId] += resources;
        slaves[slaveId].allocated += resources;

        // Resources allocated as part of the quota count towards the
        // role's and the framework's fair share.
        //
        // NOTE: Revocable resources have already been excluded.
        frameworkSorters[role]->add(slaveId, resources);
        frameworkSorters[role]->allocated(frameworkId_, slaveId, resources);
        roleSorter->allocated(role, slaveId, resources);
        quotaRoleSorter->allocated(role, slaveId, resources);
      }
    }
  }

  // Calculate the total quantity of scalar resources (including revocable
  // and reserved) that are available for allocation in the next round. We
  // need this in order to ensure we do not over-allocate resources during
  // the second stage.
  //
  // For performance reasons (MESOS-4833), this omits information about
  // dynamic reservations or persistent volumes in the resources.
  //
  // NOTE: We use total cluster resources, and not just those based on the
  // agents participating in the current allocation (i.e. provided as an
  // argument to the `allocate()` call) so that frameworks in roles without
  // quota are not unnecessarily deprived of resources.
  Resources remainingClusterResources = roleSorter->totalScalarQuantities();
  foreachkey (const string& role, activeRoles) {
    remainingClusterResources -= roleSorter->allocationScalarQuantities(role);
  }

  // Frameworks in a quota'ed role may temporarily reject resources by
  // filtering or suppressing offers. Hence quotas may not be fully allocated.
  Resources unallocatedQuotaResources;
  foreachpair (const string& name, const Quota& quota, quotas) {
    // Compute the amount of quota that the role does not have allocated.
    //
    // NOTE: Revocable resources are excluded in `quotaRoleSorter`.
    // NOTE: Only scalars are considered for quota.
    Resources allocated = getQuotaRoleAllocatedResources(name);
    const Resources required = quota.info.guarantee();
    unallocatedQuotaResources += (required - allocated);
  }

  // Determine how many resources we may allocate during the next stage.
  //
  // NOTE: Resources for quota allocations are already accounted in
  // `remainingClusterResources`.
  remainingClusterResources -= unallocatedQuotaResources;

  // To ensure we do not over-allocate resources during the second stage
  // with all frameworks, we use 2 stopping criteria:
  //   * No available resources for the second stage left, i.e.
  //     `remainingClusterResources` - `allocatedStage2` is empty.
  //   * A potential offer will force the second stage to use more resources
  //     than available, i.e. `remainingClusterResources` does not contain
  //     (`allocatedStage2` + potential offer). In this case we skip this
  //     agent and continue to the next one.
  //
  // NOTE: Like `remainingClusterResources`, `allocatedStage2` omits
  // information about dynamic reservations and persistent volumes for
  // performance reasons. This invariant is preserved because we only add
  // resources to it that have also had this metadata stripped from them
  // (typically by using `Resources::createStrippedScalarQuantity`).
  Resources allocatedStage2;

  // At this point resources for quotas are allocated or accounted for.
  // Proceed with allocating the remaining free pool.
  foreach (const SlaveID& slaveId, slaveIds) {
    // If there are no resources available for the second stage, stop.
    if (!allocatable(remainingClusterResources - allocatedStage2)) {
      break;
    }

    foreach (const string& role, roleSorter->sort()) {
      foreach (const string& frameworkId_,
               frameworkSorters[role]->sort()) {
        FrameworkID frameworkId;
        frameworkId.set_value(frameworkId_);

        // If the framework has suppressed offers, ignore.
        if (frameworks[frameworkId].suppressed) {
          continue;
        }

        // Calculate the currently available resources on the slave.
        Resources available = slaves[slaveId].total - slaves[slaveId].allocated;

        // The resources we offer are the unreserved resources as well as the
        // reserved resources for this particular role. This is necessary to
        // ensure that we don't offer resources that are reserved for another
        // role.
        //
        // NOTE: Currently, frameworks are allowed to have '*' role.
        // Calling reserved('*') returns an empty Resources object.
        //
        // NOTE: We do not offer roles with quota any more non-revocable
        // resources once their quota is satisfied. However, note that this is
        // not strictly true due to the coarse-grained nature (per agent) of the
        // allocation algorithm in stage 1.
        //
        // TODO(mpark): Offer unreserved resources as revocable beyond quota.
        Resources resources = available.reserved(role);
        if (!quotas.contains(role)) {
          resources += available.unreserved();
        }

        // Remove revocable resources if the framework has not opted
        // for them.
        if (!frameworks[frameworkId].revocable) {
          resources = resources.nonRevocable();
        }

        // If the resources are not allocatable, ignore.
        if (!allocatable(resources)) {
          continue;
        }

        // If the framework filters these resources, ignore.
        if (isFiltered(frameworkId, slaveId, resources)) {
          continue;
        }

        // If the offer generated by `resources` would force the second
        // stage to use more than `remainingClusterResources`, move along.
        // We do not terminate early, as offers generated further in the
        // loop may be small enough to fit within `remainingClusterResources`.
        const Resources scalarQuantity =
          resources.createStrippedScalarQuantity();

        if (!remainingClusterResources.contains(
                allocatedStage2 + scalarQuantity)) {
          continue;
        }

        VLOG(2) << "Allocating " << resources << " on slave " << slaveId
                << " to framework " << frameworkId;

        // NOTE: We perform "coarse-grained" allocation, meaning that we always
        // allocate the entire remaining slave resources to a single framework.
        //
        // NOTE: We may have already allocated some resources on the current
        // agent as part of quota.
        
        //主要是在这里！！！
        offerable[frameworkId][slaveId] += resources;
        allocatedStage2 += scalarQuantity;
        slaves[slaveId].allocated += resources;

        frameworkSorters[role]->add(slaveId, resources);
        frameworkSorters[role]->allocated(frameworkId_, slaveId, resources);
        roleSorter->allocated(role, slaveId, resources);

        if (quotas.contains(role)) {
          // See comment at `quotaRoleSorter` declaration regarding
          // non-revocable.
          quotaRoleSorter->allocated(role, slaveId, resources.nonRevocable());
        }
      }
    }
  }

  if (offerable.empty()) {
    VLOG(1) << "No resources available to allocate!";
  } else {
    // Now offer the resources to each framework.
    foreachkey (const FrameworkID& frameworkId, offerable) {
    ///逻辑主要在这里！！
      offerCallback(frameworkId, offerable[frameworkId]);
    }
  }

  // NOTE: For now, we implement maintenance inverse offers within the
  // allocator. We leverage the existing timer/cycle of offers to also do any
  // "deallocation" (inverse offers) necessary to satisfy maintenance needs.
  deallocate(slaveIds_);
}

~~~
注意上面分配到了资源之后，会进行回调，那么这个回调函数是在哪里设置的呢？
在Master::initialize里面：

~~~cpp
void Master::initialize()
{
  // Initialize the allocator.
  allocator->initialize(
      flags.allocation_interval,
      defer(self(), &Master::offer, lambda::_1, lambda::_2),
      defer(self(), &Master::inverseOffer, lambda::_1, lambda::_2),
      weights);
}
~~~

分配到资源之后调用 Master::offer


~~~cpp
void Master::offer(const FrameworkID& frameworkId,
                   const hashmap<SlaveID, Resources>& resources)
{
  if (!frameworks.registered.contains(frameworkId) ||
      !frameworks.registered[frameworkId]->active) {
    LOG(WARNING) << "Master returning resources offered to framework "
                 << frameworkId << " because the framework"
                 << " has terminated or is inactive";

    foreachpair (const SlaveID& slaveId, const Resources& offered, resources) {
      allocator->recoverResources(frameworkId, slaveId, offered, None());
    }
    return;
  }

  // Create an offer for each slave and add it to the message.
  ResourceOffersMessage message;

  Framework* framework = CHECK_NOTNULL(frameworks.registered[frameworkId]);
  foreachpair (const SlaveID& slaveId, const Resources& offered, resources) {
    if (!slaves.registered.contains(slaveId)) {
      LOG(WARNING)
        << "Master returning resources offered to framework " << *framework
        << " because slave " << slaveId << " is not valid";

      allocator->recoverResources(frameworkId, slaveId, offered, None());
      continue;
    }

    Slave* slave = slaves.registered.get(slaveId);
    CHECK_NOTNULL(slave);

    // This could happen if the allocator dispatched 'Master::offer' before
    // the slave was deactivated in the allocator.
    if (!slave->active) {
      LOG(WARNING)
        << "Master returning resources offered because slave " << *slave
        << " is " << (slave->connected ? "deactivated" : "disconnected");

      allocator->recoverResources(frameworkId, slaveId, offered, None());
      continue;
    }

#ifdef WITH_NETWORK_ISOLATOR
    // TODO(dhamon): This flag is required as the static allocation of
    // ephemeral ports leads to a maximum number of containers that can
    // be created on each slave. Once MESOS-1654 is fixed and ephemeral
    // ports are a first class resource, this can be removed.
    if (flags.max_executors_per_slave.isSome()) {
      // Check that we haven't hit the executor limit.
      size_t numExecutors = 0;
      foreachkey (const FrameworkID& frameworkId, slave->executors) {
        numExecutors += slave->executors[frameworkId].keys().size();
      }

      if (numExecutors >= flags.max_executors_per_slave.get()) {
        LOG(WARNING) << "Master returning resources offered because slave "
                     << *slave << " has reached the maximum number of "
                     << "executors";
        // Pass a default filter to avoid getting this same offer immediately
        // from the allocator.
        allocator->recoverResources(frameworkId, slaveId, offered, Filters());
        continue;
      }
    }
#endif // WITH_NETWORK_ISOLATOR

    // TODO(vinod): Split regular and revocable resources into
    // separate offers, so that rescinding offers with revocable
    // resources does not affect offers with regular resources.

    // TODO(bmahler): Set "https" if only "https" is supported.
    mesos::URL url;
    url.set_scheme("http");
    url.mutable_address()->set_hostname(slave->info.hostname());
    url.mutable_address()->set_ip(stringify(slave->pid.address.ip));
    url.mutable_address()->set_port(slave->pid.address.port);
    url.set_path("/" + slave->pid.id);

    Offer* offer = new Offer();
    offer->mutable_id()->MergeFrom(newOfferId());
    offer->mutable_framework_id()->MergeFrom(framework->id());
    offer->mutable_slave_id()->MergeFrom(slave->id);
    offer->set_hostname(slave->info.hostname());
    offer->mutable_url()->MergeFrom(url);
    offer->mutable_resources()->MergeFrom(offered);
    offer->mutable_attributes()->MergeFrom(slave->info.attributes());

    // Add all framework's executors running on this slave.
    if (slave->executors.contains(framework->id())) {
      const hashmap<ExecutorID, ExecutorInfo>& executors =
        slave->executors[framework->id()];
      foreachkey (const ExecutorID& executorId, executors) {
        offer->add_executor_ids()->MergeFrom(executorId);
      }
    }

    // If the slave in this offer is planned to be unavailable due to
    // maintenance in the future, then set the Unavailability.
    CHECK(machines.contains(slave->machineId));
    if (machines[slave->machineId].info.has_unavailability()) {
      offer->mutable_unavailability()->CopyFrom(
          machines[slave->machineId].info.unavailability());
    }

    offers[offer->id()] = offer;

    framework->addOffer(offer);
    slave->addOffer(offer);

    if (flags.offer_timeout.isSome()) {
      // Rescind the offer after the timeout elapses.
      offerTimers[offer->id()] =
        delay(flags.offer_timeout.get(),
              self(),
              &Self::offerTimeout,
              offer->id());
    }

    // TODO(jieyu): For now, we strip 'ephemeral_ports' resource from
    // offers so that frameworks do not see this resource. This is a
    // short term workaround. Revisit this once we resolve MESOS-1654.
    Offer offer_ = *offer;
    offer_.clear_resources();

    foreach (const Resource& resource, offered) {
      if (resource.name() != "ephemeral_ports") {
        offer_.add_resources()->CopyFrom(resource);
      }
    }

    // Add the offer *AND* the corresponding slave's PID.
    message.add_offers()->MergeFrom(offer_);
    message.add_pids(slave->pid);
  }

  if (message.offers().size() == 0) {
    return;
  }

  LOG(INFO) << "Sending " << message.offers().size()
            << " offers to framework " << *framework;

  framework->send(message);
}

~~~

上面这段代码是给Framework发送ResourceOffersMessage消息，下面我们看看
SchedulerProcess::resourceOffers是怎么处理这个消息的：

~~~cpp

 void resourceOffers(
      const UPID& from,
      const vector<Offer>& offers,
      const vector<string>& pids)
  {
    ...
    scheduler->resourceOffers(driver, offers);
    ...
 }

~~~
这样就走到了我们的自定义函数里面去了。

看一下客户端的逻辑：

~~~cpp

virtual void resourceOffers(SchedulerDriver* driver,
                              const vector<Offer>& offers)
  {
    for (size_t i = 0; i < offers.size(); i++) {
      const Offer& offer = offers[i];
      Resources remaining = offer.resources();

      static Resources TASK_RESOURCES = Resources::parse(
          "cpus:" + stringify<float>(CPUS_PER_TASK) +
          ";mem:" + stringify<size_t>(MEM_PER_TASK)).get();

      size_t maxTasks = 0;
      while (remaining.flatten().contains(TASK_RESOURCES)) {
        maxTasks++;
        remaining -= TASK_RESOURCES;
      }

      // Launch tasks.
      vector<TaskInfo> tasks;
      for (size_t i = 0; i < maxTasks / 2 && crawlQueue.size() > 0; i++) {
        string url = crawlQueue.front();
        crawlQueue.pop();
        string urlId = "C" + stringify<size_t>(processed[url]);
        TaskInfo task;
        task.set_name("Crawler " + urlId);
        task.mutable_task_id()->set_value(urlId);
        task.mutable_slave_id()->MergeFrom(offer.slave_id());
        task.mutable_executor()->MergeFrom(crawler);
        task.mutable_resources()->MergeFrom(TASK_RESOURCES);
        task.set_data(url);
        tasks.push_back(task);
        tasksLaunched++;
        cout << "Crawler " << urlId << " " << url << endl;
      }
      for (size_t i = maxTasks/2; i < maxTasks && renderQueue.size() > 0; i++) {
        string url = renderQueue.front();
        renderQueue.pop();
        string urlId = "R" + stringify<size_t>(processed[url]);
        TaskInfo task;
        task.set_name("Renderer " + urlId);
        task.mutable_task_id()->set_value(urlId);
        task.mutable_slave_id()->MergeFrom(offer.slave_id());
        task.mutable_executor()->MergeFrom(renderer);
        task.mutable_resources()->MergeFrom(TASK_RESOURCES);
        task.set_data(url);
        tasks.push_back(task);
        tasksLaunched++;
        cout << "Renderer " << urlId << " " << url << endl;
      }

      driver->launchTasks(offer.id(), tasks);
    }
  }

~~~
一般是在收到ResourceOffer以后会自己组装Task信息，然后通知driver启动launchTasks。
下面我们看看launchTasks是做什么的。


~~~cpp
Status MesosSchedulerDriver::launchTasks(
    const OfferID& offerId,
    const vector<TaskInfo>& tasks,
    const Filters& filters)
{
  vector<OfferID> offerIds;
  offerIds.push_back(offerId);

  return launchTasks(offerIds, tasks, filters);
}


Status MesosSchedulerDriver::launchTasks(
    const vector<OfferID>& offerIds,
    const vector<TaskInfo>& tasks,
    const Filters& filters)
{
  synchronized (mutex) {
    if (status != DRIVER_RUNNING) {
      return status;
    }

    CHECK(process != NULL);

    dispatch(process, &SchedulerProcess::launchTasks, offerIds, tasks, filters);

    return status;
  }
}


class SchedulerProcess
{
  ...
  void launchTasks(const vector<OfferID>& offerIds,
                   const vector<TaskInfo>& tasks,
                   const Filters& filters)
  {
    Offer::Operation operation;
    operation.set_type(Offer::Operation::LAUNCH);

    Offer::Operation::Launch* launch = operation.mutable_launch();
    foreach (const TaskInfo& task, tasks) {
      launch->add_task_infos()->CopyFrom(task);
    }

    acceptOffers(offerIds, {operation}, filters);
  }
 }
 
 
 注意connected这个变量是在detected里面设置的，会不断的监控这个变量的。
  void acceptOffers(
      const vector<OfferID>& offerIds,
      const vector<Offer::Operation>& operations,
      const Filters& filters)
  {
    // TODO(jieyu): Move all driver side verification to master since
    // we are moving towards supporting pure launguage scheduler.

    if (!connected) {
      VLOG(1) << "Ignoring accept offers message as master is disconnected";

      // NOTE: Reply to the framework with TASK_LOST messages for each
      // task launch. See details from notes in launchTasks.
      foreach (const Offer::Operation& operation, operations) {
        if (operation.type() != Offer::Operation::LAUNCH) {
          continue;
        }

        foreach (const TaskInfo& task, operation.launch().task_infos()) {
          StatusUpdate update = protobuf::createStatusUpdate(
              framework.id(),
              None(),
              task.task_id(),
              TASK_LOST,
              TaskStatus::SOURCE_MASTER,
              None(),
              "Master disconnected",
              TaskStatus::REASON_MASTER_DISCONNECTED);

          statusUpdate(UPID(), update, UPID());
        }
      }
      return;
    }

    Call call;
    CHECK(framework.has_id());
    call.mutable_framework_id()->CopyFrom(framework.id());
    call.set_type(Call::ACCEPT);

    Call::Accept* accept = call.mutable_accept();

    // Setting accept.operations.
    foreach (const Offer::Operation& _operation, operations) {
      Offer::Operation* operation = accept->add_operations();
      operation->CopyFrom(_operation);
    }

    // Setting accept.offer_ids.
    foreach (const OfferID& offerId, offerIds) {
      accept->add_offer_ids()->CopyFrom(offerId);

      if (!savedOffers.contains(offerId)) {
        // TODO(jieyu): A duplicated offer ID could also cause this
        // warning being printed. Consider refine this message here
        // and in launchTasks as well.
        LOG(WARNING) << "Attempting to accept an unknown offer " << offerId;
      } else {
        // Keep only the slave PIDs where we run tasks so we can send
        // framework messages directly.
        foreach (const Offer::Operation& operation, operations) {
          if (operation.type() != Offer::Operation::LAUNCH) {
            continue;
          }

          foreach (const TaskInfo& task, operation.launch().task_infos()) {
            const SlaveID& slaveId = task.slave_id();

            if (savedOffers[offerId].contains(slaveId)) {
              savedSlavePids[slaveId] = savedOffers[offerId][slaveId];
            } else {
              LOG(WARNING) << "Attempting to launch task " << task.task_id()
                           << " with the wrong slave id " << slaveId;
            }
          }
        }
      }

      // Remove the offer since we saved all the PIDs we might use.
      savedOffers.erase(offerId);
    }

    // Setting accept.filters.
    accept->mutable_filters()->CopyFrom(filters);

    CHECK_SOME(master);
    send(master.get().pid(), call);
  }
~~~

最主要的是发送了一个Call::ACCEPT的Call给Master。

~~~cpp

void Master::receive(
    const UPID& from,
    const scheduler::Call& call)
{
  ...
  switch (call.type()) {
    case scheduler::Call::TEARDOWN:
      teardown(framework);
      break;

    case scheduler::Call::ACCEPT:
      accept(framework, call.accept());
      break;
      
  ...
 }


void Master::accept(
    Framework* framework,
    const scheduler::Call::Accept& accept)
{
  CHECK_NOTNULL(framework);

  foreach (const Offer::Operation& operation, accept.operations()) {
    if (operation.type() == Offer::Operation::LAUNCH) {
      if (operation.launch().task_infos().size() > 0) {
        ++metrics->messages_launch_tasks;
      } else {
        ++metrics->messages_decline_offers;
      }
    }

    // TODO(jieyu): Add metrics for non launch operations.
  }

  // TODO(bmahler): We currently only support using multiple offers
  // for a single slave.
  Resources offeredResources;
  Option<SlaveID> slaveId = None();
  Option<Error> error = None();

  if (accept.offer_ids().size() == 0) {
    error = Error("No offers specified");
  } else {
    // Validate the offers.
    error = validation::offer::validate(accept.offer_ids(), this, framework);

    // Compute offered resources and remove the offers. If the
    // validation failed, return resources to the allocator.
    foreach (const OfferID& offerId, accept.offer_ids()) {
      Offer* offer = getOffer(offerId);

      // Since we re-use `OfferID`s, it is possible to arrive here with either
      // a resource offer, or an inverse offer. We first try as a resource offer
      // and if that fails, then we assume it is an inverse offer. This is
      // correct as those are currently the only 2 ways to get an `OfferID`.
      if (offer != NULL) {
        slaveId = offer->slave_id();
        offeredResources += offer->resources();

        if (error.isSome()) {
          allocator->recoverResources(
              offer->framework_id(),
              offer->slave_id(),
              offer->resources(),
              None());
        }
        removeOffer(offer);
        continue;
      }

      // Try it as an inverse offer. If this fails then the offer is no longer
      // valid.
      InverseOffer* inverseOffer = getInverseOffer(offerId);
      if (inverseOffer != NULL) {
        mesos::master::InverseOfferStatus status;
        status.set_status(mesos::master::InverseOfferStatus::ACCEPT);
        status.mutable_framework_id()->CopyFrom(inverseOffer->framework_id());
        status.mutable_timestamp()->CopyFrom(protobuf::getCurrentTime());

        allocator->updateInverseOffer(
            inverseOffer->slave_id(),
            inverseOffer->framework_id(),
            UnavailableResources{
                inverseOffer->resources(),
                inverseOffer->unavailability()},
            status,
            accept.filters());

        removeInverseOffer(inverseOffer);
        continue;
      }

      // If the offer was neither in our offer or inverse offer sets, then this
      // offer is no longer valid.
      LOG(WARNING) << "Ignoring accept of offer " << offerId
                   << " since it is no longer valid";
    }
  }

  // If invalid, send TASK_LOST for the launch attempts.
  // TODO(jieyu): Consider adding a 'drop' overload for ACCEPT call to
  // consistently handle message dropping. It would be ideal if the
  // 'drop' overload can handle both resource recovery and lost task
  // notifications.
  if (error.isSome()) {
    LOG(WARNING) << "ACCEPT call used invalid offers '" << accept.offer_ids()
                 << "': " << error.get().message;

    foreach (const Offer::Operation& operation, accept.operations()) {
      if (operation.type() != Offer::Operation::LAUNCH) {
        continue;
      }

      foreach (const TaskInfo& task, operation.launch().task_infos()) {
        const StatusUpdate& update = protobuf::createStatusUpdate(
            framework->id(),
            task.slave_id(),
            task.task_id(),
            TASK_LOST,
            TaskStatus::SOURCE_MASTER,
            None(),
            "Task launched with invalid offers: " + error.get().message,
            TaskStatus::REASON_INVALID_OFFERS);

        metrics->tasks_lost++;

        metrics->incrementTasksStates(
            TASK_LOST,
            TaskStatus::SOURCE_MASTER,
            TaskStatus::REASON_INVALID_OFFERS);

        forward(update, UPID(), framework);
      }
    }

    return;
  }

  CHECK_SOME(slaveId);
  Slave* slave = slaves.registered.get(slaveId.get());
  CHECK_NOTNULL(slave);

  LOG(INFO) << "Processing ACCEPT call for offers: " << accept.offer_ids()
            << " on slave " << *slave << " for framework " << *framework;

  list<Future<bool>> futures;
  foreach (const Offer::Operation& operation, accept.operations()) {
    switch (operation.type()) {
      case Offer::Operation::LAUNCH: {
        // Authorize the tasks. A task is in 'framework->pendingTasks'
        // before it is authorized.
        foreach (const TaskInfo& task, operation.launch().task_infos()) {
          futures.push_back(authorizeTask(task, framework));

          // Add to pending tasks.
          //
          // NOTE: The task ID here hasn't been validated yet, but it
          // doesn't matter. If the task ID is not valid, the task won't
          // be launched anyway. If two tasks have the same ID, the second
          // one will not be put into 'framework->pendingTasks', therefore
          // will not be launched.
          if (!framework->pendingTasks.contains(task.task_id())) {
            framework->pendingTasks[task.task_id()] = task;
          }
        }
        break;
      }

      // NOTE: When handling RESERVE and UNRESERVE operations, authorization
      // will proceed even if no principal is specified, although currently
      // resources cannot be reserved or unreserved unless a principal is
      // provided. Any RESERVE/UNRESERVE operation with no associated principal
      // will be found invalid when `validate()` is called in `_accept()` below.

      // The RESERVE operation allows a principal to reserve resources.
      case Offer::Operation::RESERVE: {
        Option<string> principal = framework->info.has_principal()
          ? framework->info.principal()
          : Option<string>::none();

        futures.push_back(
            authorizeReserveResources(
                operation.reserve(), principal));

        break;
      }

      // The UNRESERVE operation allows a principal to unreserve resources.
      case Offer::Operation::UNRESERVE: {
        Option<string> principal = framework->info.has_principal()
          ? framework->info.principal()
          : Option<string>::none();

        futures.push_back(
            authorizeUnreserveResources(
                operation.unreserve(), principal));

        break;
      }

      // The CREATE operation allows the creation of a persistent volume.
      case Offer::Operation::CREATE: {
        Option<string> principal = framework->info.has_principal()
          ? framework->info.principal()
          : Option<string>::none();

        futures.push_back(
            authorizeCreateVolume(
                operation.create(), principal));

        break;
      }

      // The DESTROY operation allows the destruction of a persistent volume.
      case Offer::Operation::DESTROY: {
        Option<string> principal = framework->info.has_principal()
          ? framework->info.principal()
          : Option<string>::none();

        futures.push_back(
            authorizeDestroyVolume(
                operation.destroy(), principal));

        break;
      }
    }
  }

  // Wait for all the tasks to be authorized.
  await(futures)
    .onAny(defer(self(),
                 &Master::_accept,
                 framework->id(),
                 slaveId.get(),
                 offeredResources,
                 accept,
                 lambda::_1));
}
~~~

验证了之后会调用

~~~
void Master::_accept(
    const FrameworkID& frameworkId,
    const SlaveID& slaveId,
    const Resources& offeredResources,
    const scheduler::Call::Accept& accept,
    const Future<list<Future<bool>>>& _authorizations)
{
  Framework* framework = getFramework(frameworkId);

  // TODO(jieyu): Consider using the 'drop' overload mentioned in
  // 'accept' to consistently handle dropping ACCEPT calls.
  if (framework == NULL) {
    LOG(WARNING)
      << "Ignoring ACCEPT call for framework " << frameworkId
      << " because the framework cannot be found";

    // Tell the allocator about the recovered resources.
    allocator->recoverResources(
        frameworkId,
        slaveId,
        offeredResources,
        None());

    return;
  }

  Slave* slave = slaves.registered.get(slaveId);

  if (slave == NULL || !slave->connected) {
    foreach (const Offer::Operation& operation, accept.operations()) {
      if (operation.type() != Offer::Operation::LAUNCH) {
        continue;
      }

      foreach (const TaskInfo& task, operation.launch().task_infos()) {
        const TaskStatus::Reason reason =
            slave == NULL ? TaskStatus::REASON_SLAVE_REMOVED
                          : TaskStatus::REASON_SLAVE_DISCONNECTED;
        const StatusUpdate& update = protobuf::createStatusUpdate(
            framework->id(),
            task.slave_id(),
            task.task_id(),
            TASK_LOST,
            TaskStatus::SOURCE_MASTER,
            None(),
            slave == NULL ? "Slave removed" : "Slave disconnected",
            reason);

        metrics->tasks_lost++;

        metrics->incrementTasksStates(
            TASK_LOST,
            TaskStatus::SOURCE_MASTER,
            reason);

        forward(update, UPID(), framework);
      }
    }

    // Tell the allocator about the recovered resources.
    allocator->recoverResources(
        frameworkId,
        slaveId,
        offeredResources,
        None());

    return;
  }

  // Some offer operations update the offered resources. We keep
  // updated offered resources here. When a task is successfully
  // launched, we remove its resource from offered resources.
  Resources _offeredResources = offeredResources;

  // The order of `authorizations` must match the order of the operations in
  // `accept.operations()`, as they are iterated through simultaneously.
  CHECK_READY(_authorizations);
  list<Future<bool>> authorizations = _authorizations.get();

  foreach (const Offer::Operation& operation, accept.operations()) {
    switch (operation.type()) {
      // The RESERVE operation allows a principal to reserve resources.
      case Offer::Operation::RESERVE: {
        Future<bool> authorization = authorizations.front();
        authorizations.pop_front();

        CHECK(!authorization.isDiscarded());

        if (authorization.isFailed()) {
          // TODO(greggomann): We may want to retry this failed authorization
          // request rather than dropping it immediately.
          drop(framework,
               operation,
               "Authorization of principal '" + framework->info.principal() +
               "' to reserve resources failed: " +
               authorization.failure());

          continue;
        } else if (!authorization.get()) {
          drop(framework,
               operation,
               "Not authorized to reserve resources as '" +
                 framework->info.principal() + "'");

          continue;
        }

        Option<string> principal = framework->info.has_principal()
          ? framework->info.principal()
          : Option<string>::none();

        // Make sure this reserve operation is valid.
        Option<Error> error = validation::operation::validate(
            operation.reserve(), principal);

        if (error.isSome()) {
          drop(framework, operation, error.get().message);
          continue;
        }

        // Test the given operation on the included resources.
        Try<Resources> resources = _offeredResources.apply(operation);
        if (resources.isError()) {
          drop(framework, operation, resources.error());
          continue;
        }

        _offeredResources = resources.get();

        LOG(INFO) << "Applying RESERVE operation for resources "
                  << operation.reserve().resources() << " from framework "
                  << *framework << " to slave " << *slave;

        apply(framework, slave, operation);
        break;
      }

      // The UNRESERVE operation allows a principal to unreserve resources.
      case Offer::Operation::UNRESERVE: {
        Future<bool> authorization = authorizations.front();
        authorizations.pop_front();

        CHECK(!authorization.isDiscarded());

        if (authorization.isFailed()) {
          // TODO(greggomann): We may want to retry this failed authorization
          // request rather than dropping it immediately.
          drop(framework,
               operation,
               "Authorization of principal '" + framework->info.principal() +
               "' to unreserve resources failed: " +
               authorization.failure());

          continue;
        } else if (!authorization.get()) {
          drop(framework,
               operation,
               "Not authorized to unreserve resources as '" +
                 framework->info.principal() + "'");

          continue;
        }

        // Make sure this unreserve operation is valid.
        Option<Error> error = validation::operation::validate(
            operation.unreserve());

        if (error.isSome()) {
          drop(framework, operation, error.get().message);
          continue;
        }

        // Test the given operation on the included resources.
        Try<Resources> resources = _offeredResources.apply(operation);
        if (resources.isError()) {
          drop(framework, operation, resources.error());
          continue;
        }

        _offeredResources = resources.get();

        LOG(INFO) << "Applying UNRESERVE operation for resources "
                  << operation.unreserve().resources() << " from framework "
                  << *framework << " to slave " << *slave;

        apply(framework, slave, operation);
        break;
      }

      case Offer::Operation::CREATE: {
        Future<bool> authorization = authorizations.front();
        authorizations.pop_front();

        CHECK(!authorization.isDiscarded());

        if (authorization.isFailed()) {
          // TODO(greggomann): We may want to retry this failed authorization
          // request rather than dropping it immediately.
          drop(framework,
               operation,
               "Authorization of principal '" + framework->info.principal() +
               "' to create persistent volumes failed: " +
               authorization.failure());

          continue;
        } else if (!authorization.get()) {
          drop(framework,
               operation,
               "Not authorized to create persistent volumes as '" +
                 framework->info.principal() + "'");

          continue;
        }

        // Make sure this create operation is valid.
        Option<Error> error = validation::operation::validate(
            operation.create(), slave->checkpointedResources);

        if (error.isSome()) {
          drop(framework, operation, error.get().message);
          continue;
        }

        Try<Resources> resources = _offeredResources.apply(operation);
        if (resources.isError()) {
          drop(framework, operation, resources.error());
          continue;
        }

        _offeredResources = resources.get();

        LOG(INFO) << "Applying CREATE operation for volumes "
                  << operation.create().volumes() << " from framework "
                  << *framework << " to slave " << *slave;

        apply(framework, slave, operation);
        break;
      }

      case Offer::Operation::DESTROY: {
        Future<bool> authorization = authorizations.front();
        authorizations.pop_front();

        CHECK(!authorization.isDiscarded());

        if (authorization.isFailed()) {
          // TODO(greggomann): We may want to retry this failed authorization
          // request rather than dropping it immediately.
          drop(framework,
               operation,
               "Authorization of principal '" + framework->info.principal() +
               "' to destroy persistent volumes failed: " +
               authorization.failure());

          continue;
        } else if (!authorization.get()) {
          drop(framework,
               operation,
               "Not authorized to destroy persistent volumes as '" +
                 framework->info.principal() + "'");

          continue;
        }

        // Make sure this destroy operation is valid.
        Option<Error> error = validation::operation::validate(
            operation.destroy(), slave->checkpointedResources);

        if (error.isSome()) {
          drop(framework, operation, error.get().message);
          continue;
        }

        Try<Resources> resources = _offeredResources.apply(operation);
        if (resources.isError()) {
          drop(framework, operation, resources.error());
          continue;
        }

        _offeredResources = resources.get();

        LOG(INFO) << "Applying DESTROY operation for volumes "
                  << operation.create().volumes() << " from framework "
                  << *framework << " to slave " << *slave;

        apply(framework, slave, operation);
        break;
      }

      case Offer::Operation::LAUNCH: {
        foreach (const TaskInfo& task, operation.launch().task_infos()) {
          Future<bool> authorization = authorizations.front();
          authorizations.pop_front();

          // NOTE: The task will not be in 'pendingTasks' if
          // 'killTask()' for the task was called before we are here.
          // No need to launch the task if it's no longer pending.
          // However, we still need to check the authorization result
          // and do the validation so that we can send status update
          // in case the task has duplicated ID.
          bool pending = framework->pendingTasks.contains(task.task_id());

          // Remove from pending tasks.
          framework->pendingTasks.erase(task.task_id());

          CHECK(!authorization.isDiscarded());

          if (authorization.isFailed() || !authorization.get()) {
            string user = framework->info.user(); // Default user.
            if (task.has_command() && task.command().has_user()) {
              user = task.command().user();
            } else if (task.has_executor() &&
                       task.executor().command().has_user()) {
              user = task.executor().command().user();
            }

            const StatusUpdate& update = protobuf::createStatusUpdate(
                framework->id(),
                task.slave_id(),
                task.task_id(),
                TASK_ERROR,
                TaskStatus::SOURCE_MASTER,
                None(),
                authorization.isFailed() ?
                    "Authorization failure: " + authorization.failure() :
                    "Not authorized to launch as user '" + user + "'",
                TaskStatus::REASON_TASK_UNAUTHORIZED);

            metrics->tasks_error++;

            metrics->incrementTasksStates(
                TASK_ERROR,
                TaskStatus::SOURCE_MASTER,
                TaskStatus::REASON_TASK_UNAUTHORIZED);

            forward(update, UPID(), framework);

            continue;
          }

          // Validate the task.

          // Make a copy of the original task so that we can
          // fill the missing `framework_id` in ExecutorInfo
          // if needed. This field was added to the API later
          // and thus was made optional.
          TaskInfo task_(task);
          if (task.has_executor() && !task.executor().has_framework_id()) {
            task_.mutable_executor()
                ->mutable_framework_id()->CopyFrom(framework->id());
          }

          const Option<Error>& validationError = validation::task::validate(
              task_,
              framework,
              slave,
              _offeredResources);

          if (validationError.isSome()) {
            const StatusUpdate& update = protobuf::createStatusUpdate(
                framework->id(),
                task_.slave_id(),
                task_.task_id(),
                TASK_ERROR,
                TaskStatus::SOURCE_MASTER,
                None(),
                validationError.get().message,
                TaskStatus::REASON_TASK_INVALID);

            metrics->tasks_error++;

            metrics->incrementTasksStates(
                TASK_ERROR,
                TaskStatus::SOURCE_MASTER,
                TaskStatus::REASON_TASK_INVALID);

            forward(update, UPID(), framework);

            continue;
          }

          // Add task.
          if (pending) {
            _offeredResources -= addTask(task_, framework, slave);

            // TODO(bmahler): Consider updating this log message to
            // indicate when the executor is also being launched.
            LOG(INFO) << "Launching task " << task_.task_id()
                      << " of framework " << *framework
                      << " with resources " << task_.resources()
                      << " on slave " << *slave;

            RunTaskMessage message;
            message.mutable_framework()->MergeFrom(framework->info);

            // TODO(anand): We set 'pid' to UPID() for http frameworks
            // as 'pid' was made optional in 0.24.0. In 0.25.0, we
            // no longer have to set pid here for http frameworks.
            message.set_pid(framework->pid.getOrElse(UPID()));
            message.mutable_task()->MergeFrom(task_);

            if (HookManager::hooksAvailable()) {
              // Set labels retrieved from label-decorator hooks.
              message.mutable_task()->mutable_labels()->CopyFrom(
                  HookManager::masterLaunchTaskLabelDecorator(
                      task_,
                      framework->info,
                      slave->info));
            }

            send(slave->pid, message);
          }
        }
        break;
      }

      default:
        LOG(ERROR) << "Unsupported offer operation " << operation.type();
        break;
    }
  }

  if (!_offeredResources.empty()) {
    // Tell the allocator about the unused (e.g., refused) resources.
    allocator->recoverResources(
        frameworkId,
        slaveId,
        _offeredResources,
        accept.filters());
  }
}

~~~

其实就是挨个给Slave发送RunTaskMessage。


然后我们看看Slave是怎么处理这个消息的。


~~~
// TODO(vinod): Instead of crashing the slave on checkpoint errors,
// send TASK_LOST to the framework.
void Slave::runTask(
    const UPID& from,
    const FrameworkInfo& frameworkInfo,
    const FrameworkID& frameworkId_,
    const UPID& pid,
    TaskInfo task)
{
  if (master != from) {
    LOG(WARNING) << "Ignoring run task message from " << from
                 << " because it is not the expected master: "
                 << (master.isSome() ? stringify(master.get()) : "None");
    return;
  }

  if (!frameworkInfo.has_id()) {
    LOG(ERROR) << "Ignoring run task message from " << from
               << " because it does not have a framework ID";
    return;
  }

  // Create frameworkId alias to use in the rest of the function.
  const FrameworkID frameworkId = frameworkInfo.id();

  LOG(INFO) << "Got assigned task " << task.task_id()
            << " for framework " << frameworkId;

  if (!(task.slave_id() == info.id())) {
    LOG(WARNING)
      << "Slave " << info.id() << " ignoring task " << task.task_id()
      << " because it was intended for old slave " << task.slave_id();
    return;
  }

  CHECK(state == RECOVERING || state == DISCONNECTED ||
        state == RUNNING || state == TERMINATING)
    << state;

  // TODO(bmahler): Also ignore if we're DISCONNECTED.
  if (state == RECOVERING || state == TERMINATING) {
    LOG(WARNING) << "Ignoring task " << task.task_id()
                 << " because the slave is " << state;
    // TODO(vinod): Consider sending a TASK_LOST here.
    // Currently it is tricky because 'statusUpdate()'
    // ignores updates for unknown frameworks.
    return;
  }

  Future<bool> unschedule = true;

  // If we are about to create a new framework, unschedule the work
  // and meta directories from getting gc'ed.
  Framework* framework = getFramework(frameworkId);
  if (framework == NULL) {
    // Unschedule framework work directory.
    string path = paths::getFrameworkPath(
        flags.work_dir, info.id(), frameworkId);

    if (os::exists(path)) {
      unschedule = unschedule.then(defer(self(), &Self::unschedule, path));
    }

    // Unschedule framework meta directory.
    path = paths::getFrameworkPath(metaDir, info.id(), frameworkId);
    if (os::exists(path)) {
      unschedule = unschedule.then(defer(self(), &Self::unschedule, path));
    }

    Option<UPID> frameworkPid = None();

    if (pid != UPID()) {
      frameworkPid = pid;
    }

    framework = new Framework(this, frameworkInfo, frameworkPid);
    frameworks[frameworkId] = framework;
    if (frameworkInfo.checkpoint()) {
      framework->checkpointFramework();
    }

    // Is this same framework in completedFrameworks? If so, move the completed
    // executors to this framework and remove it from that list.
    // TODO(brenden): Consider using stout/cache.hpp instead of boost
    // circular_buffer.
    for (auto it = completedFrameworks.begin(), end = completedFrameworks.end();
         it != end;
         ++it) {
      if ((*it)->id() == frameworkId) {
        framework->completedExecutors = (*it)->completedExecutors;
        completedFrameworks.erase(it);
        break;
      }
    }
  }

  const ExecutorInfo executorInfo = getExecutorInfo(frameworkInfo, task);
  const ExecutorID& executorId = executorInfo.executor_id();

  if (HookManager::hooksAvailable()) {
    // Set task labels from run task label decorator.
    task.mutable_labels()->CopyFrom(HookManager::slaveRunTaskLabelDecorator(
        task, executorInfo, frameworkInfo, info));
  }

  // We add the task to 'pending' to ensure the framework is not
  // removed and the framework and top level executor directories
  // are not scheduled for deletion before '_runTask()' is called.
  CHECK_NOTNULL(framework);
  framework->pending[executorId][task.task_id()] = task;

  // If we are about to create a new executor, unschedule the top
  // level work and meta directories from getting gc'ed.
  Executor* executor = framework->getExecutor(executorId);
  if (executor == NULL) {
    // Unschedule executor work directory.
    string path = paths::getExecutorPath(
        flags.work_dir, info.id(), frameworkId, executorId);

    if (os::exists(path)) {
      unschedule = unschedule.then(defer(self(), &Self::unschedule, path));
    }

    // Unschedule executor meta directory.
    path = paths::getExecutorPath(metaDir, info.id(), frameworkId, executorId);

    if (os::exists(path)) {
      unschedule = unschedule.then(defer(self(), &Self::unschedule, path));
    }
  }

  // Run the task after the unschedules are done.
  unschedule.onAny(
      defer(self(), &Self::_runTask, lambda::_1, frameworkInfo, task));
}


void Slave::_runTask(
    const Future<bool>& future,
    const FrameworkInfo& frameworkInfo,
    const TaskInfo& task)
{
  const FrameworkID frameworkId = frameworkInfo.id();

  LOG(INFO) << "Launching task " << task.task_id()
            << " for framework " << frameworkId;

  Framework* framework = getFramework(frameworkId);
  if (framework == NULL) {
    LOG(WARNING) << "Ignoring run task " << task.task_id()
                 << " because the framework " << frameworkId
                 << " does not exist";
    return;
  }

  const ExecutorInfo executorInfo = getExecutorInfo(frameworkInfo, task);
  const ExecutorID& executorId = executorInfo.executor_id();

  if (framework->pending.contains(executorId) &&
      framework->pending[executorId].contains(task.task_id())) {
    framework->pending[executorId].erase(task.task_id());
    if (framework->pending[executorId].empty()) {
      framework->pending.erase(executorId);
      // NOTE: Ideally we would perform the following check here:
      //
      //   if (framework->executors.empty() &&
      //       framework->pending.empty()) {
      //     removeFramework(framework);
      //   }
      //
      // However, we need 'framework' to stay valid for the rest of
      // this function. As such, we perform the check before each of
      // the 'return' statements below.
    }
  } else {
    LOG(WARNING) << "Ignoring run task " << task.task_id()
                 << " of framework " << frameworkId
                 << " because the task has been killed in the meantime";
    return;
  }

  // We don't send a status update here because a terminating
  // framework cannot send acknowledgements.
  if (framework->state == Framework::TERMINATING) {
    LOG(WARNING) << "Ignoring run task " << task.task_id()
                 << " of framework " << frameworkId
                 << " because the framework is terminating";

    // Refer to the comment after 'framework->pending.erase' above
    // for why we need this.
    if (framework->executors.empty() && framework->pending.empty()) {
      removeFramework(framework);
    }

    return;
  }

  if (!future.isReady()) {
    LOG(ERROR) << "Failed to unschedule directories scheduled for gc: "
               << (future.isFailed() ? future.failure() : "future discarded");

    const StatusUpdate update = protobuf::createStatusUpdate(
        frameworkId,
        info.id(),
        task.task_id(),
        TASK_LOST,
        TaskStatus::SOURCE_SLAVE,
        UUID::random(),
        "Could not launch the task because we failed to unschedule directories"
        " scheduled for gc",
        TaskStatus::REASON_GC_ERROR);

    // TODO(vinod): Ensure that the status update manager reliably
    // delivers this update. Currently, we don't guarantee this
    // because removal of the framework causes the status update
    // manager to stop retrying for its un-acked updates.
    statusUpdate(update, UPID());

    // Refer to the comment after 'framework->pending.erase' above
    // for why we need this.
    if (framework->executors.empty() && framework->pending.empty()) {
      removeFramework(framework);
    }

    return;
  }

  // NOTE: If the task or executor uses resources that are
  // checkpointed on the slave (e.g. persistent volumes), we should
  // already know about it. If the slave doesn't know about them (e.g.
  // CheckpointResourcesMessage was dropped or came out of order),
  // we send TASK_LOST status updates here since restarting the task
  // may succeed in the event that CheckpointResourcesMessage arrives
  // out of order.
  Resources checkpointedTaskResources =
    Resources(task.resources()).filter(needCheckpointing);

  foreach (const Resource& resource, checkpointedTaskResources) {
    if (!checkpointedResources.contains(resource)) {
      LOG(WARNING) << "Unknown checkpointed resource " << resource
                   << " for task " << task.task_id()
                   << " of framework " << frameworkId;

      const StatusUpdate update = protobuf::createStatusUpdate(
          frameworkId,
          info.id(),
          task.task_id(),
          TASK_LOST,
          TaskStatus::SOURCE_SLAVE,
          UUID::random(),
          "The checkpointed resources being used by the task are unknown to "
          "the slave",
          TaskStatus::REASON_RESOURCES_UNKNOWN);

      statusUpdate(update, UPID());

      // Refer to the comment after 'framework->pending.erase' above
      // for why we need this.
      if (framework->executors.empty() && framework->pending.empty()) {
        removeFramework(framework);
      }

      return;
    }
  }

  if (task.has_executor()) {
    Resources checkpointedExecutorResources =
      Resources(task.executor().resources()).filter(needCheckpointing);

    foreach (const Resource& resource, checkpointedExecutorResources) {
      if (!checkpointedResources.contains(resource)) {
        LOG(WARNING) << "Unknown checkpointed resource " << resource
                     << " for executor '" << task.executor().executor_id()
                     << "' of framework " << frameworkId;

        const StatusUpdate update = protobuf::createStatusUpdate(
            frameworkId,
            info.id(),
            task.task_id(),
            TASK_LOST,
            TaskStatus::SOURCE_SLAVE,
            UUID::random(),
            "The checkpointed resources being used by the executor are unknown "
            "to the slave",
            TaskStatus::REASON_RESOURCES_UNKNOWN,
            task.executor().executor_id());

        statusUpdate(update, UPID());

        // Refer to the comment after 'framework->pending.erase' above
        // for why we need this.
        if (framework->executors.empty() && framework->pending.empty()) {
          removeFramework(framework);
        }

        return;
      }
    }
  }

  // NOTE: The slave cannot be in 'RECOVERING' because the task would
  // have been rejected in 'runTask()' in that case.
  CHECK(state == DISCONNECTED || state == RUNNING || state == TERMINATING)
    << state;

  if (state == TERMINATING) {
    LOG(WARNING) << "Ignoring run task " << task.task_id()
                 << " of framework " << frameworkId
                 << " because the slave is terminating";

    // Refer to the comment after 'framework->pending.erase' above
    // for why we need this.
    if (framework->executors.empty() && framework->pending.empty()) {
      removeFramework(framework);
    }

    // We don't send a TASK_LOST here because the slave is
    // terminating.
    return;
  }

  CHECK(framework->state == Framework::RUNNING) << framework->state;

  // Either send the task to an executor or start a new executor
  // and queue the task until the executor has started.
  Executor* executor = framework->getExecutor(executorId);
~~~

~~~cpp
  if (executor == NULL) {
    executor = framework->launchExecutor(executorInfo, task);
  }
~~~

~~~cpp

  CHECK_NOTNULL(executor);

  switch (executor->state) {
    case Executor::TERMINATING:
    case Executor::TERMINATED: {
      LOG(WARNING) << "Asked to run task '" << task.task_id()
                   << "' for framework " << frameworkId
                   << " with executor '" << executorId
                   << "' which is terminating/terminated";

      const StatusUpdate update = protobuf::createStatusUpdate(
          frameworkId,
          info.id(),
          task.task_id(),
          TASK_LOST,
          TaskStatus::SOURCE_SLAVE,
          UUID::random(),
          "Executor terminating/terminated",
          TaskStatus::REASON_EXECUTOR_TERMINATED);

      statusUpdate(update, UPID());
      break;
    }
    case Executor::REGISTERING:
      // Checkpoint the task before we do anything else.
      if (executor->checkpoint) {
        executor->checkpointTask(task);
      }

      // Queue task if the executor has not yet registered.
      LOG(INFO) << "Queuing task '" << task.task_id()
                << "' for executor " << *executor;

      executor->queuedTasks[task.task_id()] = task;
      break;
    case Executor::RUNNING: {
      // Checkpoint the task before we do anything else.
      if (executor->checkpoint) {
        executor->checkpointTask(task);
      }

      // Queue task until the containerizer is updated with new
      // resource limits (MESOS-998).
      LOG(INFO) << "Queuing task '" << task.task_id()
                << "' for executor " << *executor;

      executor->queuedTasks[task.task_id()] = task;

      // Update the resource limits for the container. Note that the
      // resource limits include the currently queued tasks because we
      // want the container to have enough resources to hold the
      // upcoming tasks.
      Resources resources = executor->resources;

      // TODO(jieyu): Use foreachvalue instead once LinkedHashmap
      // supports it.
      foreach (const TaskInfo& task, executor->queuedTasks.values()) {
        resources += task.resources();
      }

      containerizer->update(executor->containerId, resources)
        .onAny(defer(self(),
                     &Self::runTasks,
                     lambda::_1,
                     frameworkId,
                     executorId,
                     executor->containerId,
                     list<TaskInfo>({task})));
      break;
    }
    default:
      LOG(FATAL) << "Executor " << *executor << " is in unexpected state "
                 << executor->state;
      break;
  }

  // We don't perform the checks for 'removeFramework' here since
  // we're guaranteed by 'launchExecutor' that 'framework->executors'
  // will be non-empty.
  CHECK(!framework->executors.empty());
}
~~~


~~~cpp
// Create and launch an executor.
Executor* Framework::launchExecutor(
    const ExecutorInfo& executorInfo,
    const TaskInfo& taskInfo)
{
  // Generate an ID for the executor's container.
  // TODO(idownes) This should be done by the containerizer but we
  // need the ContainerID to create the executor's directory. Fix
  // this when 'launchExecutor()' is handled asynchronously.
  ContainerID containerId;
  containerId.set_value(UUID::random().toString());

  Option<string> user = None();
#ifndef __WINDOWS__
  if (slave->flags.switch_user) {
    // The command (either in form of task or executor command) can
    // define a specific user to run as. If present, this precedes the
    // framework user value. The selected user will have been verified by
    // the master at this point through the active ACLs.
    // NOTE: The global invariant is that the executor info at this
    // point is (1) the user provided task.executor() or (2) a command
    // executor constructed by the slave from the task.command().
    // If this changes, we need to check the user in both
    // task.command() and task.executor().command() below.
    user = info.user();
    if (executorInfo.command().has_user()) {
      user = executorInfo.command().user();
    }
  }
#endif // __WINDOWS__

  // Create a directory for the executor.
  const string directory = paths::createExecutorDirectory(
      slave->flags.work_dir,
      slave->info.id(),
      id(),
      executorInfo.executor_id(),
      containerId,
      user);

  Executor* executor = new Executor(
      slave, id(), executorInfo, containerId, directory, info.checkpoint());

  if (executor->checkpoint) {
    executor->checkpointExecutor();
  }

  CHECK(!executors.contains(executorInfo.executor_id()))
    << "Unknown executor " << executorInfo.executor_id();

  executors[executorInfo.executor_id()] = executor;

  LOG(INFO) << "Launching executor " << executorInfo.executor_id()
            << " of framework " << id()
            << " with resources " << executorInfo.resources()
            << " in work directory '" << directory << "'";

  slave->files->attach(executor->directory, executor->directory)
    .onAny(defer(slave, &Slave::fileAttached, lambda::_1, executor->directory));

  // Tell the containerizer to launch the executor.
  // NOTE: We modify the ExecutorInfo to include the task's
  // resources when launching the executor so that the containerizer
  // has non-zero resources to work with when the executor has
  // no resources. This should be revisited after MESOS-600.
  ExecutorInfo executorInfo_ = executor->info;
  Resources resources = executorInfo_.resources();
  resources += taskInfo.resources();
  executorInfo_.mutable_resources()->CopyFrom(resources);

  // Launch the container.
  Future<bool> launch;
  if (!executor->isCommandExecutor()) {
    // If the executor is _not_ a command executor, this means that
    // the task will include the executor to run. The actual task to
    // run will be enqueued and subsequently handled by the executor
    // when it has registered to the slave.
    launch = slave->containerizer->launch(
        containerId,
        executorInfo_, // Modified to include the task's resources, see above.
        executor->directory,
        user,
        slave->info.id(),
        slave->self(),
        info.checkpoint());
  } else {
    // An executor has _not_ been provided by the task and will
    // instead define a command and/or container to run. Right now,
    // these tasks will require an executor anyway and the slave
    // creates a command executor. However, it is up to the
    // containerizer how to execute those tasks and the generated
    // executor info works as a placeholder.
    // TODO(nnielsen): Obsolete the requirement for executors to run
    // one-off tasks.
    launch = slave->containerizer->launch(
        containerId,
        taskInfo,
        executorInfo_, // Modified to include the task's resources, see above.
        executor->directory,
        user,
        slave->info.id(),
        slave->self(),
        info.checkpoint());
  }

  launch.onAny(defer(slave,
                     &Slave::executorLaunched,
                     id(),
                     executor->id,
                     containerId,
                     lambda::_1));

  // Make sure the executor registers within the given timeout.
  delay(slave->flags.executor_registration_timeout,
        slave,
        &Slave::registerExecutorTimeout,
        id(),
        executor->id,
        containerId);

  return executor;
}
~~~

这里实际上就是起了一个进程

~~~cpp

Future<bool> ExternalContainerizerProcess::launch(
    const ContainerID& containerId,
    const Option<TaskInfo>& taskInfo,
    const ExecutorInfo& executor,
    const string& directory,
    const Option<string>& user,
    const SlaveID& slaveId,
    const PID<Slave>& slavePid,
    bool checkpoint)
{
  LOG(INFO) << "Launching container '" << containerId << "'";

  if (actives.contains(containerId)) {
    return Failure("Cannot start already running container '" +
                   containerId.value() + "'");
  }

  map<string, string> environment = executorEnvironment(
      executor,
      directory,
      slaveId,
      slavePid,
      checkpoint,
      flags);

  // TODO(tillt): Consider moving this into
  // Containerizer::executorEnvironment.
  if (!flags.hadoop_home.empty()) {
    environment["HADOOP_HOME"] = flags.hadoop_home;
  }

  if (flags.default_container_image.isSome()) {
    environment["MESOS_DEFAULT_CONTAINER_IMAGE"] =
      flags.default_container_image.get();
  }

  containerizer::Launch launch;
  launch.mutable_container_id()->CopyFrom(containerId);
  if (taskInfo.isSome()) {
    launch.mutable_task_info()->CopyFrom(taskInfo.get());
  }
  launch.mutable_executor_info()->CopyFrom(executor);
  launch.set_directory(directory);
  if (user.isSome()) {
    launch.set_user(user.get());
  }
  launch.mutable_slave_id()->CopyFrom(slaveId);
  launch.set_slave_pid(slavePid);
  launch.set_checkpoint(checkpoint);

  Sandbox sandbox(directory, user);

  Try<Subprocess> invoked = invoke(
      "launch",
      launch,
      sandbox,
      environment);

  if (invoked.isError()) {
    return Failure("Launch of container '" + containerId.value() +
                   "' failed: " + invoked.error());
  }

  // Checkpoint the executor's pid if requested.
  // NOTE: Containerizer(s) currently rely on their state being
  // persisted in the slave. However, that responsibility should have
  // been delegated to the containerizer.
  // To work around the mandatory forked pid recovery, we need to
  // checkpoint one. See MESOS-1328 and MESOS-923.
  // TODO(tillt): Remove this entirely as soon as MESOS-923 is fixed.
  if (checkpoint) {
    const string& path = slave::paths::getForkedPidPath(
        slave::paths::getMetaRootDir(flags.work_dir),
        slaveId,
        executor.framework_id(),
        executor.executor_id(),
        containerId);

    LOG(INFO) << "Checkpointing executor's forked pid " << invoked.get().pid()
              << " to '" << path <<  "'";

    Try<Nothing> checkpointed =
      slave::state::checkpoint(path, stringify(invoked.get().pid()));

    if (checkpointed.isError()) {
      LOG(ERROR) << "Failed to checkpoint executor's forked pid to '"
                 << path << "': " << checkpointed.error();

      return Failure("Could not checkpoint executor's pid");
    }
  }

  // Record the container launch intend.
  actives.put(containerId, Owned<Container>(new Container(sandbox)));

  return invoked.get().status()
    .then(defer(
        PID<ExternalContainerizerProcess>(this),
        &ExternalContainerizerProcess::_launch,
        containerId,
        lambda::_1))
    .onAny(defer(
        PID<ExternalContainerizerProcess>(this),
        &ExternalContainerizerProcess::__launch,
        containerId,
        lambda::_1));
}


Future<bool> ExternalContainerizerProcess::_launch(
    const ContainerID& containerId,
    const Future<Option<int>>& future)
{
  VLOG(1) << "Launch validation callback triggered on container '"
          << containerId << "'";

  Option<Error> error = validate(future);
  if (error.isSome()) {
    return Failure("Could not launch container '" +
                   containerId.value() + "': " + error.get().message);
  }

  VLOG(1) << "Launch finishing up for container '" << containerId << "'";

  // Launch is done, we can now process all other commands that might
  // have gotten chained up.
  actives[containerId]->launched.set(Nothing());

  return true;
}


void ExternalContainerizerProcess::__launch(
    const ContainerID& containerId,
    const Future<bool>& future)
{
  VLOG(1) << "Launch confirmation callback triggered on container '"
          << containerId << "'";

  // We need to cleanup whenever this callback was invoked due to a
  // failure or discarded future.
  if (!future.isReady()) {
    cleanup(containerId);
  }
}


~~~


~~~cpp
slave launched以后就等待进程结束，实际上就是containerizer->wait的效果。

void Slave::executorLaunched(
    const FrameworkID& frameworkId,
    const ExecutorID& executorId,
    const ContainerID& containerId,
    const Future<bool>& future)
{
  // Set up callback for executor termination. Note that we do this
  // regardless of whether or not we have successfully launched the
  // executor because even if we failed to launch the executor the
  // result of calling 'wait' will make sure everything gets properly
  // cleaned up. Note that we do this here instead of where we do
  // Containerizer::launch because we want to guarantee the contract
  // with the Containerizer that we won't call 'wait' until after the
  // launch has completed.
  containerizer->wait(containerId)
    .onAny(defer(self(),
                 &Self::executorTerminated,
                 frameworkId,
                 executorId,
                 lambda::_1));

  if (!future.isReady()) {
    LOG(ERROR) << "Container '" << containerId
               << "' for executor '" << executorId
               << "' of framework " << frameworkId
               << " failed to start: "
               << (future.isFailed() ? future.failure() : " future discarded");

    ++metrics.container_launch_errors;

    containerizer->destroy(containerId);

    Executor* executor = getExecutor(frameworkId, executorId);
    if (executor != NULL) {
      containerizer::Termination termination;
      termination.set_state(TASK_FAILED);
      termination.add_reasons(TaskStatus::REASON_CONTAINER_LAUNCH_FAILED);
      termination.set_message(
          "Failed to launch container: " +
          (future.isFailed() ? future.failure() : "discarded"));

      executor->pendingTermination = termination;

      // TODO(jieyu): Set executor->state to be TERMINATING.
    }

    return;
  } else if (!future.get()) {
    LOG(ERROR) << "Container '" << containerId
               << "' for executor '" << executorId
               << "' of framework " << frameworkId
               << " failed to start: None of the enabled containerizers ("
               << flags.containerizers << ") could create a container for the "
               << "provided TaskInfo/ExecutorInfo message";

    ++metrics.container_launch_errors;
    return;
  }

  Framework* framework = getFramework(frameworkId);
  if (framework == NULL) {
    LOG(WARNING) << "Framework '" << frameworkId
                 << "' for executor '" << executorId
                 << "' is no longer valid";
    return;
  }

  CHECK(framework->state == Framework::RUNNING ||
        framework->state == Framework::TERMINATING)
    << framework->state;

  if (framework->state == Framework::TERMINATING) {
    LOG(WARNING) << "Killing executor '" << executorId
                 << "' of framework " << frameworkId
                 << " because the framework is terminating";
    containerizer->destroy(containerId);
    return;
  }

  Executor* executor = framework->getExecutor(executorId);
  if (executor == NULL) {
    LOG(WARNING) << "Killing unknown executor '" << executorId
                 << "' of framework " << frameworkId;
    containerizer->destroy(containerId);
    return;
  }

  switch (executor->state) {
    case Executor::TERMINATING:
      LOG(WARNING) << "Killing executor " << *executor
                   << " because the executor is terminating";

      containerizer->destroy(containerId);
      break;
    case Executor::REGISTERING:
    case Executor::RUNNING:
      break;
    case Executor::TERMINATED:
    default:
      LOG(FATAL) << "Executor " << *executor << " is in an unexpected state "
                 << executor->state;

      break;
  }
}
~~~



以上就是Framework的注册和运行任务的机制，下面看看executor是怎么启动起来的。


注意这个函数设置了executor所需的所有的环境变量：

~~~cpp

map<string, string> executorEnvironment(
    const ExecutorInfo& executorInfo,
    const string& directory,
    const SlaveID& slaveId,
    const PID<Slave>& slavePid,
    bool checkpoint,
    const Flags& flags,
    bool includeOsEnvironment)
{
  map<string, string> environment;

  // In cases where DNS is not available on the slave, the absence of
  // LIBPROCESS_IP in the executor's environment will cause an error when the
  // new executor process attempts a hostname lookup. Thus, we pass the slave's
  // LIBPROCESS_IP through here, even if the executor environment is specified
  // explicitly. Note that a LIBPROCESS_IP present in the provided flags will
  // override this value.
  Option<string> libprocessIP = os::getenv("LIBPROCESS_IP");
  if (libprocessIP.isSome()) {
    environment["LIBPROCESS_IP"] = libprocessIP.get();
  }

  if (flags.executor_environment_variables.isSome()) {
    foreachpair (const string& key,
                 const JSON::Value& value,
                 flags.executor_environment_variables.get().values) {
      // See slave/flags.cpp where we validate each value is a string.
      CHECK(value.is<JSON::String>());
      environment[key] = value.as<JSON::String>().value;
    }
  } else if (includeOsEnvironment) {
    environment = os::environment();
  }

  // Set LIBPROCESS_PORT so that we bind to a random free port (since
  // this might have been set via --port option). We do this before
  // the environment variables below in case it is included.
  environment["LIBPROCESS_PORT"] = "0";

  // Also add MESOS_NATIVE_JAVA_LIBRARY if it's not already present (and
  // like above, we do this before the environment variables below in
  // case the framework wants to override).
  // TODO(tillt): Adapt library towards JNI specific name once libmesos
  // has been split.
  if (environment.count("MESOS_NATIVE_JAVA_LIBRARY") == 0) {
    string path =
#ifdef __APPLE__
      LIBDIR "/libmesos-" VERSION ".dylib";
#else
      LIBDIR "/libmesos-" VERSION ".so";
#endif
    if (os::exists(path)) {
      environment["MESOS_NATIVE_JAVA_LIBRARY"] = path;
    }
  }

  // Also add MESOS_NATIVE_LIBRARY if it's not already present.
  // This environment variable is kept for offering non JVM-based
  // frameworks a more compact and JNI independent library.
  if (environment.count("MESOS_NATIVE_LIBRARY") == 0) {
    string path =
#ifdef __APPLE__
      LIBDIR "/libmesos-" VERSION ".dylib";
#else
      LIBDIR "/libmesos-" VERSION ".so";
#endif
    if (os::exists(path)) {
      environment["MESOS_NATIVE_LIBRARY"] = path;
    }
  }

  environment["MESOS_FRAMEWORK_ID"] = executorInfo.framework_id().value();
  environment["MESOS_EXECUTOR_ID"] = executorInfo.executor_id().value();
  environment["MESOS_DIRECTORY"] = directory;
  environment["MESOS_SLAVE_ID"] = slaveId.value();
  environment["MESOS_SLAVE_PID"] = stringify(slavePid);
  environment["MESOS_AGENT_ENDPOINT"] = stringify(slavePid.address);
  environment["MESOS_CHECKPOINT"] = checkpoint ? "1" : "0";
  environment["MESOS_EXECUTOR_SHUTDOWN_GRACE_PERIOD"] =
    stringify(flags.executor_shutdown_grace_period);

  if (checkpoint) {
    environment["MESOS_RECOVERY_TIMEOUT"] = stringify(flags.recovery_timeout);

    // The maximum backoff duration to be used by an executor between two
    // retries when disconnected.
    environment["MESOS_SUBSCRIPTION_BACKOFF_MAX"] =
      stringify(EXECUTOR_REREGISTER_TIMEOUT);
  }

  if (HookManager::hooksAvailable()) {
    // Include any environment variables from Hooks.
    // TODO(karya): Call environment decorator hook _after_ putting all
    // variables from executorInfo into 'env'. This would prevent the
    // ones provided by hooks from being overwritten by the ones in
    // executorInfo in case of a conflict. The overwriting takes places
    // at the callsites of executorEnvironment (e.g., ___launch function
    // in src/slave/containerizer/docker.cpp)
    // TODO(karya): Provide a mechanism to pass the new environment
    // variables created above (MESOS_*) on to the hook modules.
    const Environment& hooksEnvironment =
      HookManager::slaveExecutorEnvironmentDecorator(executorInfo);

    foreach (const Environment::Variable& variable,
             hooksEnvironment.variables()) {
      environment[variable.name()] = variable.value();
    }
  }

  return environment;
}

~~~
