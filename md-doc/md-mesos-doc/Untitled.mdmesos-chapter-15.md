#Executor

~~~cpp

class ExecutorProcess : public ProtobufProcess<ExecutorProcess>
{
public:
  virtual void initialize()
  {
    VLOG(1) << "Executor started at: " << self()
            << " with pid " << getpid();

    link(slave);

    // Register with slave.
    RegisterExecutorMessage message;
    message.mutable_framework_id()->MergeFrom(frameworkId);
    message.mutable_executor_id()->MergeFrom(executorId);
    send(slave, message);
  }
 
 }
 ~~~
 上来就给slave发送RegisterExecutorMessage ，Slave注册成功后后回送ExecutorRegisteredMessage
 然后回调executor的接口。
 ~~~cpp
 void registered(const ExecutorInfo& executorInfo,
                  const FrameworkID& frameworkId,
                  const FrameworkInfo& frameworkInfo,
                  const SlaveID& slaveId,
                  const SlaveInfo& slaveInfo)
  {
    if (aborted.load()) {
      VLOG(1) << "Ignoring registered message from slave " << slaveId
              << " because the driver is aborted!";
      return;
    }

    LOG(INFO) << "Executor registered on slave " << slaveId;

    connected = true;
    connection = UUID::random();

    Stopwatch stopwatch;
    if (FLAGS_v >= 1) {
      stopwatch.start();
    }

    executor->registered(driver, executorInfo, frameworkInfo, slaveInfo);

    VLOG(1) << "Executor::registered took " << stopwatch.elapsed();
  }

~~~

slave里面有一行

void Slave::registerExecutor(
    const UPID& from,
    const FrameworkID& frameworkId,
    const ExecutorID& executorId)
{
  switch (executor->state) {
   
   case Executor::REGISTERING: {
      executor->state = Executor::RUNNING;

      // Save the pid for the executor.
      // 建立连接
      executor->pid = from;
      link(from);

     // Tell executor it's registered and give it any queued tasks.
      ExecutorRegisteredMessage message;
      message.mutable_executor_info()->MergeFrom(executor->info);
      message.mutable_framework_id()->MergeFrom(framework->id());
      message.mutable_framework_info()->MergeFrom(framework->info);
      message.mutable_slave_id()->MergeFrom(info.id());
      message.mutable_slave_info()->MergeFrom(info);
      executor->send(message);

      containerizer->update(executor->containerId, resources)
        .onAny(defer(self(),
                     &Self::runTasks,
                     lambda::_1,
                     frameworkId,
                     executorId,
                     executor->containerId,
                     executor->queuedTasks.values()));

}

void Slave::runTasks(
    const Future<Nothing>& future,
    const FrameworkID& frameworkId,
    const ExecutorID& executorId,
    const ContainerID& containerId,
    const list<TaskInfo>& tasks)
{
...
    // Add the task and send it to the executor.
    executor->addTask(task);

    LOG(INFO) << "Sending queued task '" << task.task_id()
              << "' to executor " << *executor;

    RunTaskMessage message;
    message.mutable_framework()->MergeFrom(framework->info);
    message.mutable_task()->MergeFrom(task);

    // Note that 0.23.x executors require the 'pid' to be set
    // to decode the message, but do not use the field.
    message.set_pid(framework->pid.getOrElse(UPID()));

    executor->send(message);
  }
}


ExecutorProcess:
void runTask(const TaskInfo& task)
  {
    if (aborted.load()) {
      VLOG(1) << "Ignoring run task message for task " << task.task_id()
              << " because the driver is aborted!";
      return;
    }

    CHECK(!tasks.contains(task.task_id()))
      << "Unexpected duplicate task " << task.task_id();

    tasks[task.task_id()] = task;

    VLOG(1) << "Executor asked to run task '" << task.task_id() << "'";

    Stopwatch stopwatch;
    if (FLAGS_v >= 1) {
      stopwatch.start();
    }

    executor->launchTask(driver, task);

    VLOG(1) << "Executor::launchTask took " << stopwatch.elapsed();
  }

这里就回到了我们自己的任务逻辑那里。

看一下这个消息：
FrameworkToExecutorMessage

