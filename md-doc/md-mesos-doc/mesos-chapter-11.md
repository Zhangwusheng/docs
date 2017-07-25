#Recover

除了各个层级之外，他还记录个最近的一个任务的运行，叫runs
可以首先看一下目录组织，然后就知道怎么一层一层的恢复数据了。

##各种目录
slave/paths.cpp

~~~cpp
const char TASK_UPDATES_FILE[] = "task.updates";

const char SLAVES_DIR[] = "slaves";
const char FRAMEWORKS_DIR[] = "frameworks";
const char EXECUTORS_DIR[] = "executors";
const char CONTAINERS_DIR[] = "runs";
const char RESOURCES_INFO_FILE[] = "resources.info";


string getSlavePath(
    const string& rootDir,
    const SlaveID& slaveId)
{
  return path::join(rootDir, SLAVES_DIR, stringify(slaveId));
}


string getFrameworkPath(
    const string& rootDir,
    const SlaveID& slaveId,
    const FrameworkID& frameworkId)
{
  return path::join(
      getSlavePath(rootDir, slaveId), FRAMEWORKS_DIR, stringify(frameworkId));
}


string getExecutorPath(
    const string& rootDir,
    const SlaveID& slaveId,
    const FrameworkID& frameworkId,
    const ExecutorID& executorId)
{
  return path::join(
        getFrameworkPath(rootDir, slaveId, frameworkId),
        EXECUTORS_DIR,
        stringify(executorId));
}


string getExecutorRunPath(
    const string& rootDir,
    const SlaveID& slaveId,
    const FrameworkID& frameworkId,
    const ExecutorID& executorId,
    const ContainerID& containerId)
{
  return path::join(
      getExecutorPath(rootDir, slaveId, frameworkId, executorId),
      CONTAINERS_DIR,
      stringify(containerId));
}



string getTaskPath(
    const string& rootDir,
    const SlaveID& slaveId,
    const FrameworkID& frameworkId,
    const ExecutorID& executorId,
    const ContainerID& containerId,
    const TaskID& taskId)
{
  return path::join(
      getExecutorRunPath(
          rootDir,
          slaveId,
          frameworkId,
          executorId,
          containerId),
      "tasks",
      stringify(taskId));
}


string getTaskUpdatesPath(
    const string& rootDir,
    const SlaveID& slaveId,
    const FrameworkID& frameworkId,
    const ExecutorID& executorId,
    const ContainerID& containerId,
    const TaskID& taskId)
{
  return path::join(
      getTaskPath(
          rootDir,
          slaveId,
          frameworkId,
          executorId,
          containerId,
          taskId),
      TASK_UPDATES_FILE);
}

string getMetaRootDir(const string& rootDir)
{
  return path::join(rootDir, "meta");
}

string getResourcesInfoPath(
    const string& rootDir)
{
  return path::join(rootDir, "resources", RESOURCES_INFO_FILE);
}

~~~

* SLAVE的目录是：SLAVE=$ROOT/slaves/${slaveId}
* Framework的目录是： $SLAVE/frameworks/${frameworkId}
* Executor 的目录是：$Framework/executors/${executorId}
* ExecutorRun的目录是：$Executor/runs/${containerId}
* Task的目录是：$ExecutorRun/tasks/${taskId}
* TaskUpdatesPath的目录是：$Task/task.updates
* ResourcesInfoPath的目录是：$ROOT/resources/resources.info




~~~cpp

Slave::Slave()
    metaDir(paths::getMetaRootDir(flags.work_dir)),

~~~

首先构造函数里面把metaDir设置为：$work_dir/meta/,然后在Slave::initialize里面调用

~~~cpp 
  // Do recovery.
  async(&state::recover, metaDir, flags.strict)
    .then(defer(self(), &Slave::recover, lambda::_1))
    .then(defer(self(), &Slave::_recover))
    .onAny(defer(self(), &Slave::__recover, lambda::_1));
~~~

这里用到了async函数。

recover函数执行了之后，所有的checkpoint的状态已经读取到了内存了。然后就会调用
Slave::recover函数

~~~cpp
applyCheckpointedResources在common/resource_utils.cpp里面

这个要好好看看，看不懂呢。
Try<Resources> applyCheckpointedResources(
    const Resources& resources,
    const Resources& checkpointedResources)
{
  Resources totalResources = resources;

  foreach (const Resource& resource, checkpointedResources) {
    if (!needCheckpointing(resource)) {
      return Error("Unexpected checkpointed resources " + stringify(resource));
    }

    Resource stripped = resource;

    if (Resources::isDynamicallyReserved(resource)) {
      stripped.set_role("*");
      stripped.clear_reservation();
    }

    // Strip persistence and volume from the disk info so that we can
    // check whether it is contained in the `totalResources`.
    if (Resources::isPersistentVolume(resource)) {
      if (stripped.disk().has_source()) {
        stripped.mutable_disk()->clear_persistence();
        stripped.mutable_disk()->clear_volume();
      } else {
        stripped.clear_disk();
      }
    }

    if (!totalResources.contains(stripped)) {
      return Error(
          "Incompatible slave resources: " + stringify(totalResources) +
          " does not contain " + stringify(stripped));
    }

    totalResources -= stripped;
    totalResources += resource;
  }

  return totalResources;
}
~~~

~~~cpp
Future<Nothing> Slave::recover(const Result<state::State>& state)
{
  if (state.isError()) {
    return Failure(state.error());
  }

  Option<ResourcesState> resourcesState;
  Option<SlaveState> slaveState;
  if (state.isSome()) {
    resourcesState = state.get().resources;
    slaveState = state.get().slave;
  }

  // Recover checkpointed resources.
  // NOTE: 'resourcesState' is None if the slave rootDir does not
  // exist or the resources checkpoint file cannot be found.
  if (resourcesState.isSome()) {
    if (resourcesState.get().errors > 0) {
      LOG(WARNING) << "Errors encountered during resources recovery: "
                   << resourcesState.get().errors;

      metrics.recovery_errors += resourcesState.get().errors;
    }

    // This is to verify that the checkpointed resources are
    // compatible with the slave resources specified through the
    // '--resources' command line flag.
    Try<Resources> totalResources = applyCheckpointedResources(
        info.resources(),
        resourcesState.get().resources);

    if (totalResources.isError()) {
      return Failure(
          "Checkpointed resources " +
          stringify(resourcesState.get().resources) +
          " are incompatible with slave resources " +
          stringify(info.resources()) + ": " +
          totalResources.error());
    }

    checkpointedResources = resourcesState.get().resources;
  }

  if (slaveState.isSome() && slaveState.get().info.isSome()) {
    // Check for SlaveInfo compatibility.
    // TODO(vinod): Also check for version compatibility.
    // NOTE: We set the 'id' field in 'info' from the recovered slave,
    // as a hack to compare the info created from options/flags with
    // the recovered info.
    info.mutable_id()->CopyFrom(slaveState.get().id);
    if (flags.recover == "reconnect" &&
        !(info == slaveState.get().info.get())) {
      return Failure(strings::join(
          "\n",
          "Incompatible slave info detected.",
          "------------------------------------------------------------",
          "Old slave info:\n" + stringify(slaveState.get().info.get()),
          "------------------------------------------------------------",
          "New slave info:\n" + stringify(info),
          "------------------------------------------------------------"));
    }

    info = slaveState.get().info.get(); // Recover the slave info.

    if (slaveState.get().errors > 0) {
      LOG(WARNING) << "Errors encountered during slave recovery: "
                   << slaveState.get().errors;

      metrics.recovery_errors += slaveState.get().errors;
    }

    // TODO(bernd-mesos): Make this an instance method call, see comment
    // in "fetcher.hpp"".
    Try<Nothing> recovered = Fetcher::recover(slaveState.get().id, flags);
    if (recovered.isError()) {
      return Failure(recovered.error());
    }

    // Recover the frameworks.
    foreachvalue (const FrameworkState& frameworkState,
                  slaveState.get().frameworks) {
      recoverFramework(frameworkState);
    }
  }

  return statusUpdateManager->recover(metaDir, slaveState)
    .then(defer(self(), &Slave::_recoverContainerizer, slaveState));
}
~~~
~~~cpp
Try<Nothing> Fetcher::recover(const SlaveID& slaveId, const Flags& flags)
{
  // Good enough for now, simple, least-effort recovery.
  VLOG(1) << "Clearing fetcher cache";

  string cacheDirectory = paths::getSlavePath(flags.fetcher_cache_dir, slaveId);
  Result<string> path = os::realpath(cacheDirectory);
  if (path.isError()) {
    LOG(ERROR) << "Malformed fetcher cache directory path '" << cacheDirectory
               << "', error: " + path.error();

    return Error(path.error());
  }

  if (path.isSome() && os::exists(path.get())) {
    Try<Nothing> rmdir = os::rmdir(path.get(), true);
    if (rmdir.isError()) {
      LOG(ERROR) << "Could not delete fetcher cache directory '"
                 << cacheDirectory << "', error: " + rmdir.error();

      return rmdir;
    }
  }

  return Nothing();
}
~~~

Fetcher::recover仅仅是删除Slave的缓存目录。继续看recoverFramework

~~~cpp
void Slave::recoverFramework(const FrameworkState& state)
{
  LOG(INFO) << "Recovering framework " << state.id;

  if (state.executors.empty()) {
    // GC the framework work directory.
    garbageCollect(
        paths::getFrameworkPath(flags.work_dir, info.id(), state.id));

    // GC the framework meta directory.
    garbageCollect(
        paths::getFrameworkPath(metaDir, info.id(), state.id));

    return;
  }

  CHECK(!frameworks.contains(state.id));

  CHECK_SOME(state.info);
  FrameworkInfo frameworkInfo = state.info.get();

  // Mesos 0.22 and earlier didn't write the FrameworkID into the FrameworkInfo.
  // In this case, we we update FrameworkInfo.framework_id from directory name,
  // and rewrite the new format when we are done.
  bool recheckpoint = false;
  if (!frameworkInfo.has_id()) {
    frameworkInfo.mutable_id()->CopyFrom(state.id);
    recheckpoint = true;
  }

  CHECK(frameworkInfo.has_id());
  CHECK(frameworkInfo.checkpoint());

  // In 0.24.0, HTTP schedulers are supported and these do not
  // have a 'pid'. In this case, the slave will checkpoint UPID().
  CHECK_SOME(state.pid);

  Option<UPID> pid = state.pid.get();

  if (pid.get() == UPID()) {
    pid = None();
  }

  Framework* framework = new Framework(this, frameworkInfo, pid);
  frameworks[framework->id()] = framework;

  if (recheckpoint) {
    framework->checkpointFramework();
  }

  // Now recover the executors for this framework.
  foreachvalue (const ExecutorState& executorState, state.executors) {
    framework->recoverExecutor(executorState);
  }

  // Remove the framework in case we didn't recover any executors.
  if (framework->executors.empty()) {
    removeFramework(framework);
  }
}

~~~


~~~cpp

void Framework::recoverExecutor(const ExecutorState& state)
{
  LOG(INFO) << "Recovering executor '" << state.id
            << "' of framework " << id();

  CHECK_NOTNULL(slave);

  if (state.runs.empty() || state.latest.isNone() || state.info.isNone()) {
    LOG(WARNING) << "Skipping recovery of executor '" << state.id
                 << "' of framework " << id()
                 << " because its latest run or executor info"
                 << " cannot be recovered";

    // GC the top level executor work directory.
    slave->garbageCollect(paths::getExecutorPath(
        slave->flags.work_dir, slave->info.id(), id(), state.id));

    // GC the top level executor meta directory.
    slave->garbageCollect(paths::getExecutorPath(
        slave->metaDir, slave->info.id(), id(), state.id));

    return;
  }

  // We are only interested in the latest run of the executor!
  // So, we GC all the old runs.
  // NOTE: We don't schedule the top level executor work and meta
  // directories for GC here, because they will be scheduled when
  // the latest executor run terminates.
  const ContainerID& latest = state.latest.get();
  foreachvalue (const RunState& run, state.runs) {
    CHECK_SOME(run.id);
    const ContainerID& runId = run.id.get();
    if (latest != runId) {
      // GC the executor run's work directory.
      // TODO(vinod): Expose this directory to webui by recovering the
      // tasks and doing a 'files->attach()'.
      slave->garbageCollect(paths::getExecutorRunPath(
          slave->flags.work_dir, slave->info.id(), id(), state.id, runId));

      // GC the executor run's meta directory.
      slave->garbageCollect(paths::getExecutorRunPath(
          slave->metaDir, slave->info.id(), id(), state.id, runId));
    }
  }

  Option<RunState> run = state.runs.get(latest);
  CHECK_SOME(run)
      << "Cannot find latest run " << latest << " for executor " << state.id
      << " of framework " << id();

  // Create executor.
  const string directory = paths::getExecutorRunPath(
      slave->flags.work_dir, slave->info.id(), id(), state.id, latest);

  Executor* executor = new Executor(
      slave, id(), state.info.get(), latest, directory, info.checkpoint());

  // Recover the libprocess PID if possible for PID based executors.
  if (run.get().http.isSome()) {
    if (!run.get().http.get()) {
      // When recovering in non-strict mode, the assumption is that the
      // slave can die after checkpointing the forked pid but before the
      // libprocess pid. So, it is not possible for the libprocess pid
      // to exist but not the forked pid. If so, it is a really bad
      // situation (e.g., disk corruption).
      CHECK_SOME(run.get().forkedPid)
        << "Failed to get forked pid for executor " << state.id
        << " of framework " << id();

      executor->pid = run.get().libprocessPid.get();
    } else {
      // We set the PID to None() to signify that this is a HTTP based
      // executor.
      executor->pid = None();
    }
  } else {
    // We set the PID to UPID() to signify that the connection type for this
    // executor is unknown.
    executor->pid = UPID();
  }

  // And finally recover all the executor's tasks.
  foreachvalue (const TaskState& taskState, run.get().tasks) {
    executor->recoverTask(taskState);
  }

  // Expose the executor's files.
  // 这里的文件主要是可以通过HTTP访问，可以不用关心。
  slave->files->attach(executor->directory, executor->directory)
    .onAny(defer(slave, &Slave::fileAttached, lambda::_1, executor->directory));

  // Add the executor to the framework.
  executors[executor->id] = executor;

  // If the latest run of the executor was completed (i.e., terminated
  // and all updates are acknowledged) in the previous run, we
  // transition its state to 'TERMINATED' and gc the directories.
  if (run.get().completed) {
    ++slave->metrics.executors_terminated;

    executor->state = Executor::TERMINATED;

    CHECK_SOME(run.get().id);
    const ContainerID& runId = run.get().id.get();

    // GC the executor run's work directory.
    const string path = paths::getExecutorRunPath(
        slave->flags.work_dir, slave->info.id(), id(), state.id, runId);

    slave->garbageCollect(path)
       .then(defer(slave, &Slave::detachFile, path));

    // GC the executor run's meta directory.
    slave->garbageCollect(paths::getExecutorRunPath(
        slave->metaDir, slave->info.id(), id(), state.id, runId));

    // GC the top level executor work directory.
    slave->garbageCollect(paths::getExecutorPath(
        slave->flags.work_dir, slave->info.id(), id(), state.id));

    // GC the top level executor meta directory.
    slave->garbageCollect(paths::getExecutorPath(
        slave->metaDir, slave->info.id(), id(), state.id));

    // Move the executor to 'completedExecutors'.
    destroyExecutor(executor->id);
  }

  return;
}

void Executor::recoverTask(const TaskState& state)
{
  if (state.info.isNone()) {
    LOG(WARNING) << "Skipping recovery of task " << state.id
                 << " because its info cannot be recovered";
    return;
  }

  launchedTasks[state.id] = new Task(state.info.get());

  // NOTE: Since some tasks might have been terminated when the
  // slave was down, the executor resources we capture here is an
  // upper-bound. The actual resources needed (for live tasks) by
  // the isolator will be calculated when the executor re-registers.
  resources += state.info.get().resources();

  // Read updates to get the latest state of the task.
  foreach (const StatusUpdate& update, state.updates) {
    updateTaskState(update.status());

    // Terminate the task if it received a terminal update.
    // We ignore duplicate terminal updates by checking if
    // the task is present in launchedTasks.
    // TODO(vinod): Revisit these semantics when we disallow duplicate
    // terminal updates (e.g., when slave recovery is always enabled).
    if (protobuf::isTerminalState(update.status().state()) &&
        launchedTasks.contains(state.id)) {
      terminateTask(state.id, update.status());

      CHECK(update.has_uuid())
        << "Expecting updates without 'uuid' to have been rejected";

      // If the terminal update has been acknowledged, remove it.
      if (state.acks.contains(UUID::fromBytes(update.uuid()))) {
        completeTask(state.id);
      }
      break;
    }
  }
}

这里主要是任务的更新状态重新更新回去

void Executor::updateTaskState(const TaskStatus& status)
{
  if (launchedTasks.contains(status.task_id())) {
    Task* task = launchedTasks[status.task_id()];
    // TODO(brenden): Consider wiping the `data` and `message` fields?
    if (task->statuses_size() > 0 &&
        task->statuses(task->statuses_size() - 1).state() == status.state()) {
      task->mutable_statuses()->RemoveLast();
    }
    task->add_statuses()->CopyFrom(status);
    task->set_state(status.state());
  }
}

~~~

~~~cpp
files/files.cpp


Future<Nothing> FilesProcess::attach(const string& path, const string& name)
{
  Result<string> result = os::realpath(path);

  if (!result.isSome()) {
    return Failure(
        "Failed to get realpath of '" + path + "': " +
        (result.isError()
         ? result.error()
         : "No such file or directory"));
  }

  // Make sure we have permissions to read the file/dir.
  Try<bool> access = os::access(result.get(), R_OK);

  if (access.isError() || !access.get()) {
    return Failure("Failed to access '" + path + "': " +
                   (access.isError() ? access.error() : "Access denied"));
  }

  // To simplify the read/browse logic, strip any trailing / from the name.
  string cleanedName = strings::remove(name, "/", strings::SUFFIX);

  // TODO(bmahler): Do we want to always wipe out the previous path?
  paths[cleanedName] = result.get();

  return Nothing();
}
~~~
上来首先调用garbageCollect回收两个目录，如果这个Slave下面的Framework为空，那么这个目录页应该删除。这两个目录的删除也比较有意思：

~~~cpp
slave/gc.cpp

Future<Nothing> GarbageCollectorProcess::schedule(
    const Duration& d,
    const string& path)
{
  LOG(INFO) << "Scheduling '" << path << "' for gc " << d << " in the future";

  // If there's an existing schedule for this path, we must remove
  // it here in order to reschedule.
  if (timeouts.contains(path)) {
    CHECK(unschedule(path));
  }

  Owned<Promise<Nothing> > promise(new Promise<Nothing>());

  Timeout removalTime = Timeout::in(d);

  timeouts[path] = removalTime;
  paths.put(removalTime, PathInfo(path, promise));

  // If the timer is not yet initialized or the timeout is sooner than
  // the currently active timer, update it.
  if (timer.timeout().remaining() == Seconds(0) ||
      removalTime < timer.timeout()) {
    reset(); // Schedule the timer for next event.
  }

  return promise->future();
}

超时了就调用reset。主要逻辑还是在reset里面。
void GarbageCollectorProcess::reset()
{
  Clock::cancel(timer); // Cancel the existing timer, if any.
  if (!paths.empty()) {
    Timeout removalTime = (*paths.begin()).first; // Get the first entry.

    timer = delay(removalTime.remaining(), self(), &Self::remove, removalTime);
  } else {
    timer = Timer(); // Reset the timer.
  }
}

这里计算好了延迟之后，调用delay，分发的还是GarbageCollectorProcess::remove函数

void GarbageCollectorProcess::remove(const Timeout& removalTime)
{
  // TODO(bmahler): Other dispatches can block waiting for a removal
  // operation. To fix this, the removal operation can be done
  // asynchronously in another thread.
  if (paths.count(removalTime) > 0) {
    foreach (const PathInfo& info, paths.get(removalTime)) {
      LOG(INFO) << "Deleting " << info.path;

      Try<Nothing> rmdir = os::rmdir(info.path);

      if (rmdir.isError()) {
        LOG(WARNING) << "Failed to delete '" << info.path << "': "
                     << rmdir.error();
        info.promise->fail(rmdir.error());
      } else {
        LOG(INFO) << "Deleted '" << info.path << "'";
        info.promise->set(rmdir.get());
      }

      timeouts.erase(info.path);
    }

    paths.remove(removalTime);
  } else {
    // This occurs when either:
    //   1. The path(s) has already been removed (e.g. by prune()).
    //   2. All paths under the removal time were unscheduled.
    LOG(INFO) << "Ignoring gc event at " << removalTime.remaining()
              << " as the paths were already removed, or were unscheduled";
  }

  reset(); // Schedule the timer for next event.
}

主要还是调用rmdir ：）


~~~

##目录结构为：

~~~cpp
// The slave leverages the file system for a number of purposes:
//
//   (1) For executor sandboxes and disk volumes.
//
//   (2) For checkpointing metadata in order to support "slave
//       recovery". That is, allow the slave to restart without
//       affecting the tasks / executors.
//
//   (3) To detect reboots (by checkpointing the boot id), in which
//       case everything has died so no recovery should be attempted.
//
//   (4) For checkpointing resources that persist across slaves.
//       This includes things like persistent volumes and dynamic
//       reservations.
//
//   (5) For provisioning root filesystems for containers.
//
// The file system layout is as follows:
//
//   root ('--work_dir' flag)
//   |-- slaves
//   |   |-- latest (symlink)
//   |   |-- <slave_id>
//   |       |-- frameworks
//   |           |-- <framework_id>
//   |               |-- executors
//   |                   |-- <executor_id>
//   |                       |-- runs
//   |                           |-- latest (symlink)
//   |                           |-- <container_id> (sandbox)
//   |-- meta
//   |   |-- slaves
//   |       |-- latest (symlink)
//   |       |-- <slave_id>
//   |           |-- slave.info
//   |           |-- frameworks
//   |               |-- <framework_id>
//   |                   |-- framework.info
//   |                   |-- framework.pid
//   |                   |-- executors
//   |                       |-- <executor_id>
//   |                           |-- executor.info
//   |                           |-- runs
//   |                               |-- latest (symlink)
//   |                               |-- <container_id> (sandbox)
//   |                                   |-- executor.sentinel (if completed)
//   |                                   |-- pids
//   |                                   |   |-- forked.pid
//   |                                   |   |-- libprocess.pid
//   |                                   |-- tasks
//   |                                       |-- <task_id>
//   |                                           |-- task.info
//   |                                           |-- task.updates
//   |-- boot_id
//   |-- resources
//   |   |-- resources.info
//   |-- volumes
//   |   |-- roles
//   |       |-- <role>
//   |           |-- <persistence_id> (persistent volume)
//   |-- provisioner

~~~


##State(s)

~~~cpp
struct ExecutorState
{
  ExecutorState () : errors(0) {}

  static Try<ExecutorState> recover(
      const std::string& rootDir,
      const SlaveID& slaveId,
      const FrameworkID& frameworkId,
      const ExecutorID& executorId,
      bool strict);

  ExecutorID id;
  Option<ExecutorInfo> info;
  Option<ContainerID> latest;
  hashmap<ContainerID, RunState> runs;
  unsigned int errors;
};


struct FrameworkState
{
  FrameworkState () : errors(0) {}

  static Try<FrameworkState> recover(
      const std::string& rootDir,
      const SlaveID& slaveId,
      const FrameworkID& frameworkId,
      bool strict);

  FrameworkID id;
  Option<FrameworkInfo> info;

  // Note that HTTP frameworks (supported in 0.24.0) do not have a
  // PID, in which case 'pid' is Some(UPID()) rather than None().
  Option<process::UPID> pid;

  hashmap<ExecutorID, ExecutorState> executors;
  unsigned int errors;
};


struct ResourcesState
{
  ResourcesState() : errors(0) {}

  static Try<ResourcesState> recover(
      const std::string& rootDir,
      bool strict);

  Resources resources;
  unsigned int errors;
};


struct SlaveState
{
  SlaveState () : errors(0) {}

  static Try<SlaveState> recover(
      const std::string& rootDir,
      const SlaveID& slaveId,
      bool strict);

  SlaveID id;
  Option<SlaveInfo> info;
  hashmap<FrameworkID, FrameworkState> frameworks;
  unsigned int errors;
};


// The top level state. The members are child nodes in the tree. Each
// child node (recursively) recovers the checkpointed state.
struct State
{
  State() : errors(0) {}

  Option<ResourcesState> resources;
  Option<SlaveState> slave;

  // TODO(jieyu): Consider using a vector of Option<Error> here so
  // that we can print all the errors. This also applies to all the
  // State structs below.
  unsigned int errors;
};


~~~

~~~cpp

const char BOOT_ID_FILE[] = "boot_id";

string getBootIdPath(const string& rootDir)
{
  return path::join(rootDir, BOOT_ID_FILE);
}

string getLatestSlavePath(const string& rootDir)
{
  return path::join(rootDir, SLAVES_DIR, LATEST_SYMLINK);
}

~~~

### ::recover

recover也是一级一级的向下恢复的。最根部的是State，这里面包含了Resource和Slave。

~~~cpp
// The top level state. The members are child nodes in the tree. Each
// child node (recursively) recovers the checkpointed state.
struct State
{
  State() : errors(0) {}

  Option<ResourcesState> resources;
  Option<SlaveState> slave;

  // TODO(jieyu): Consider using a vector of Option<Error> here so
  // that we can print all the errors. This also applies to all the
  // State structs below.
  unsigned int errors;
};

Result<State> recover(const string& rootDir, bool strict)
{
  LOG(INFO) << "Recovering state from '" << rootDir << "'";

  // We consider the absence of 'rootDir' to mean that this is either
  // the first time this slave was started or this slave was started after
  // an upgrade (--recover=cleanup).
  if (!os::exists(rootDir)) {
    return None();
  }

  // Now, start to recover state from 'rootDir'.
  State state;

  // Recover resources regardless whether the host has rebooted.
  Try<ResourcesState> resources = ResourcesState::recover(rootDir, strict);
  if (resources.isError()) {
    return Error(resources.error());
  }

  // TODO(jieyu): Do not set 'state.resources' if we cannot find the
  // resources checkpoint file.
  state.resources = resources.get();

  // Did the machine reboot? No need to recover slave state if the
  // machine has rebooted.
  if (os::exists(paths::getBootIdPath(rootDir))) {
    Try<string> read = os::read(paths::getBootIdPath(rootDir));
    if (read.isSome()) {
      Try<string> id = os::bootId();
      CHECK_SOME(id);

      if (id.get() != strings::trim(read.get())) {
        LOG(INFO) << "Slave host rebooted";
        return state;
      }
    }
  }

  const string& latest = paths::getLatestSlavePath(rootDir);

  // Check if the "latest" symlink to a slave directory exists.
  if (!os::exists(latest)) {
    // The slave was asked to shutdown or died before it registered
    // and had a chance to create the "latest" symlink.
    LOG(INFO) << "Failed to find the latest slave from '" << rootDir << "'";
    return state;
  }

  // Get the latest slave id.
  Result<string> directory = os::realpath(latest);
  if (!directory.isSome()) {
    return Error("Failed to find latest slave: " +
                 (directory.isError()
                  ? directory.error()
                  : "No such file or directory"));
  }

  SlaveID slaveId;
  slaveId.set_value(Path(directory.get()).basename());

  Try<SlaveState> slave = SlaveState::recover(rootDir, slaveId, strict);
  if (slave.isError()) {
    return Error(slave.error());
  }

  state.slave = slave.get();

  return state;
}
~~~

Slave只恢复最近的一个，latest对应的目录里面的内容才是我们需要恢复的。

###ResourcesState恢复

~~~cpp

struct ResourcesState
{
  ResourcesState() : errors(0) {}

  static Try<ResourcesState> recover(
      const std::string& rootDir,
      bool strict);

  Resources resources;
  unsigned int errors;
};


Try<ResourcesState> ResourcesState::recover(
    const string& rootDir,
    bool strict)
{
  ResourcesState state;

  const string& path = paths::getResourcesInfoPath(rootDir);
  if (!os::exists(path)) {
    LOG(INFO) << "No checkpointed resources found at '" << path << "'";
    return state;
  }

  Try<int> fd = os::open(path, O_RDWR | O_CLOEXEC);
  if (fd.isError()) {
    string message =
      "Failed to open resources file '" + path + "': " + fd.error();

    if (strict) {
      return Error(message);
    } else {
      LOG(WARNING) << message;
      state.errors++;
      return state;
    }
  }

  Result<Resource> resource = None();
  while (true) {
    // Ignore errors due to partial protobuf read and enable undoing
    // failed reads by reverting to the previous seek position.
    resource = ::protobuf::read<Resource>(fd.get(), true, true);
    if (!resource.isSome()) {
      break;
    }

    state.resources += resource.get();
  }

  off_t offset = lseek(fd.get(), 0, SEEK_CUR);
  if (offset < 0) {
    os::close(fd.get());
    return ErrnoError("Failed to lseek resources file '" + path + "'");
  }

  // Always truncate the file to contain only valid resources.
  // NOTE: This is safe even though we ignore partial protobuf read
  // errors above, because the 'fd' is properly set to the end of the
  // last valid resource by 'protobuf::read()'.
  Try<Nothing> truncated = os::ftruncate(fd.get(), offset);

  if (truncated.isError()) {
    os::close(fd.get());
    return Error(
      "Failed to truncate resources file '" + path +
      "': " + truncated.error());
  }

  // After reading a non-corrupted resources file, 'record' should be
  // 'none'.
  if (resource.isError()) {
    string message =
      "Failed to read resources file  '" + path + "': " + resource.error();

    os::close(fd.get());

    if (strict) {
      return Error(message);
    } else {
      LOG(WARNING) << message;
      state.errors++;
      return state;
    }
  }

  os::close(fd.get());

  return state;
}


~~~

###SlaveState恢复

~~~cpp
Try<list<string>> getFrameworkPaths(
    const string& rootDir,
    const SlaveID& slaveId)
{
  return fs::list(
      path::join(getSlavePath(rootDir, slaveId), FRAMEWORKS_DIR, "*"));
}

Try<SlaveState> SlaveState::recover(
    const string& rootDir,
    const SlaveID& slaveId,
    bool strict)
{
  SlaveState state;
  state.id = slaveId;

  // Read the slave info.
  const string& path = paths::getSlaveInfoPath(rootDir, slaveId);
  if (!os::exists(path)) {
    // This could happen if the slave died before it registered with
    // the master.
    LOG(WARNING) << "Failed to find slave info file '" << path << "'";
    return state;
  }

  const Result<SlaveInfo>& slaveInfo = ::protobuf::read<SlaveInfo>(path);

  if (slaveInfo.isError()) {
    const string& message = "Failed to read slave info from '" + path + "': " +
                            slaveInfo.error();
    if (strict) {
      return Error(message);
    } else {
      LOG(WARNING) << message;
      state.errors++;
      return state;
    }
  }

  if (slaveInfo.isNone()) {
    // This could happen if the slave died after opening the file for
    // writing but before it checkpointed anything.
    LOG(WARNING) << "Found empty slave info file '" << path << "'";
    return state;
  }

  state.info = slaveInfo.get();

  // Find the frameworks.
  Try<list<string> > frameworks = paths::getFrameworkPaths(rootDir, slaveId);

  if (frameworks.isError()) {
    return Error("Failed to find frameworks for slave " + slaveId.value() +
                 ": " + frameworks.error());
  }

  // Recover each of the frameworks.
  foreach (const string& path, frameworks.get()) {
    FrameworkID frameworkId;
    frameworkId.set_value(Path(path).basename());

    Try<FrameworkState> framework =
      FrameworkState::recover(rootDir, slaveId, frameworkId, strict);

    if (framework.isError()) {
      return Error("Failed to recover framework " + frameworkId.value() +
                   ": " + framework.error());
    }

    state.frameworks[frameworkId] = framework.get();
    state.errors += framework.get().errors;
  }

  return state;
}

~~~

###FrameworkState

~~~cpp
Try<FrameworkState> FrameworkState::recover(
    const string& rootDir,
    const SlaveID& slaveId,
    const FrameworkID& frameworkId,
    bool strict)
{
  FrameworkState state;
  state.id = frameworkId;
  string message;

  // Read the framework info.
  string path = paths::getFrameworkInfoPath(rootDir, slaveId, frameworkId);
  if (!os::exists(path)) {
    // This could happen if the slave died after creating the
    // framework directory but before it checkpointed the framework
    // info.
    LOG(WARNING) << "Failed to find framework info file '" << path << "'";
    return state;
  }

  const Result<FrameworkInfo>& frameworkInfo =
    ::protobuf::read<FrameworkInfo>(path);

  if (frameworkInfo.isError()) {
    message = "Failed to read framework info from '" + path + "': " +
              frameworkInfo.error();

    if (strict) {
      return Error(message);
    } else {
      LOG(WARNING) << message;
      state.errors++;
      return state;
    }
  }

  if (frameworkInfo.isNone()) {
    // This could happen if the slave died after opening the file for
    // writing but before it checkpointed anything.
    LOG(WARNING) << "Found empty framework info file '" << path << "'";
    return state;
  }

  state.info = frameworkInfo.get();

  // Read the framework pid.
  path = paths::getFrameworkPidPath(rootDir, slaveId, frameworkId);
  if (!os::exists(path)) {
    // This could happen if the slave died after creating the
    // framework info but before it checkpointed the framework pid.
    LOG(WARNING) << "Failed to framework pid file '" << path << "'";
    return state;
  }

  Try<string> pid = os::read(path);

  if (pid.isError()) {
    message =
      "Failed to read framework pid from '" + path + "': " + pid.error();

    if (strict) {
      return Error(message);
    } else {
      LOG(WARNING) << message;
      state.errors++;
      return state;
    }
  }

  if (pid.get().empty()) {
    // This could happen if the slave died after opening the file for
    // writing but before it checkpointed anything.
    LOG(WARNING) << "Found empty framework pid file '" << path << "'";
    return state;
  }

  state.pid = process::UPID(pid.get());

  // Find the executors.
  Try<list<string> > executors =
    paths::getExecutorPaths(rootDir, slaveId, frameworkId);

  if (executors.isError()) {
    return Error(
        "Failed to find executors for framework " + frameworkId.value() +
        ": " + executors.error());
  }

  // Recover the executors.
  foreach (const string& path, executors.get()) {
    ExecutorID executorId;
    executorId.set_value(Path(path).basename());

    Try<ExecutorState> executor =
      ExecutorState::recover(rootDir, slaveId, frameworkId, executorId, strict);

    if (executor.isError()) {
      return Error("Failed to recover executor '" + executorId.value() +
                   "': " + executor.error());
    }

    state.executors[executorId] = executor.get();
    state.errors += executor.get().errors;
  }

  return state;
}
~~~
###ExecutorState恢复

~~~cpp
Try<ExecutorState> ExecutorState::recover(
    const string& rootDir,
    const SlaveID& slaveId,
    const FrameworkID& frameworkId,
    const ExecutorID& executorId,
    bool strict)
{
  ExecutorState state;
  state.id = executorId;
  string message;

  // Find the runs.
  Try<list<string> > runs = paths::getExecutorRunPaths(
      rootDir,
      slaveId,
      frameworkId,
      executorId);

  if (runs.isError()) {
    return Error("Failed to find runs for executor '" + executorId.value() +
                 "': " + runs.error());
  }

  // Recover the runs.
  foreach (const string& path, runs.get()) {
    if (Path(path).basename() == paths::LATEST_SYMLINK) {
      const Result<string>& latest = os::realpath(path);
      if (!latest.isSome()) {
        return Error(
            "Failed to find latest run of executor '" +
            executorId.value() + "': " +
            (latest.isError()
             ? latest.error()
             : "No such file or directory"));
      }

      // Store the ContainerID of the latest executor run.
      ContainerID containerId;
      containerId.set_value(Path(latest.get()).basename());
      state.latest = containerId;
    } else {
      ContainerID containerId;
      containerId.set_value(Path(path).basename());

      Try<RunState> run = RunState::recover(
          rootDir, slaveId, frameworkId, executorId, containerId, strict);

      if (run.isError()) {
        return Error(
            "Failed to recover run " + containerId.value() +
            " of executor '" + executorId.value() +
            "': " + run.error());
      }

      state.runs[containerId] = run.get();
      state.errors += run.get().errors;
    }
  }

  // Find the latest executor.
  // It is possible that we cannot find the "latest" executor if the
  // slave died before it created the "latest" symlink.
  if (state.latest.isNone()) {
    LOG(WARNING) << "Failed to find the latest run of executor '"
                 << executorId << "' of framework " << frameworkId;
    return state;
  }

  // Read the executor info.
  const string& path =
    paths::getExecutorInfoPath(rootDir, slaveId, frameworkId, executorId);
  if (!os::exists(path)) {
    // This could happen if the slave died after creating the executor
    // directory but before it checkpointed the executor info.
    LOG(WARNING) << "Failed to find executor info file '" << path << "'";
    return state;
  }

  const Result<ExecutorInfo>& executorInfo =
    ::protobuf::read<ExecutorInfo>(path);

  if (executorInfo.isError()) {
    message = "Failed to read executor info from '" + path + "': " +
              executorInfo.error();

    if (strict) {
      return Error(message);
    } else {
      LOG(WARNING) << message;
      state.errors++;
      return state;
    }
  }

  if (executorInfo.isNone()) {
    // This could happen if the slave died after opening the file for
    // writing but before it checkpointed anything.
    LOG(WARNING) << "Found empty executor info file '" << path << "'";
    return state;
  }

  state.info = executorInfo.get();

  return state;
}

~~~

###RunState恢复

~~~cpp
Try<RunState> RunState::recover(
    const string& rootDir,
    const SlaveID& slaveId,
    const FrameworkID& frameworkId,
    const ExecutorID& executorId,
    const ContainerID& containerId,
    bool strict)
{
  RunState state;
  state.id = containerId;
  string message;

  // See if the sentinel file exists. This is done first so it is
  // known even if partial state is returned, e.g., if the libprocess
  // pid file is not recovered. It indicates the slave removed the
  // executor.
  string path = paths::getExecutorSentinelPath(
      rootDir, slaveId, frameworkId, executorId, containerId);

  state.completed = os::exists(path);

  // Find the tasks.
  Try<list<string> > tasks = paths::getTaskPaths(
      rootDir,
      slaveId,
      frameworkId,
      executorId,
      containerId);

  if (tasks.isError()) {
    return Error(
        "Failed to find tasks for executor run " + containerId.value() +
        ": " + tasks.error());
  }

  // Recover tasks.
  foreach (const string& path, tasks.get()) {
    TaskID taskId;
    taskId.set_value(Path(path).basename());

    Try<TaskState> task = TaskState::recover(
        rootDir, slaveId, frameworkId, executorId, containerId, taskId, strict);

    if (task.isError()) {
      return Error(
          "Failed to recover task " + taskId.value() + ": " + task.error());
    }

    state.tasks[taskId] = task.get();
    state.errors += task.get().errors;
  }

  // Read the forked pid.
  path = paths::getForkedPidPath(
      rootDir, slaveId, frameworkId, executorId, containerId);
  if (!os::exists(path)) {
    // This could happen if the slave died before the isolator
    // checkpointed the forked pid.
    LOG(WARNING) << "Failed to find executor forked pid file '" << path << "'";
    return state;
  }

  Try<string> pid = os::read(path);

  if (pid.isError()) {
    message = "Failed to read executor forked pid from '" + path +
              "': " + pid.error();

    if (strict) {
      return Error(message);
    } else {
      LOG(WARNING) << message;
      state.errors++;
      return state;
    }
  }

  if (pid.get().empty()) {
    // This could happen if the slave died after opening the file for
    // writing but before it checkpointed anything.
    LOG(WARNING) << "Found empty executor forked pid file '" << path << "'";
    return state;
  }

  Try<pid_t> forkedPid = numify<pid_t>(pid.get());
  if (forkedPid.isError()) {
    return Error("Failed to parse forked pid " + pid.get() +
                 ": " + forkedPid.error());
  }

  state.forkedPid = forkedPid.get();

  // Read the libprocess pid.
  path = paths::getLibprocessPidPath(
      rootDir, slaveId, frameworkId, executorId, containerId);

  if (os::exists(path)) {
    pid = os::read(path);

    if (pid.isError()) {
      message = "Failed to read executor libprocess pid from '" + path +
                "': " + pid.error();

      if (strict) {
        return Error(message);
      } else {
        LOG(WARNING) << message;
        state.errors++;
        return state;
      }
    }

    if (pid.get().empty()) {
      // This could happen if the slave died after opening the file for
      // writing but before it checkpointed anything.
      LOG(WARNING) << "Found empty executor libprocess pid file '" << path
                   << "'";
      return state;
    }

    state.libprocessPid = process::UPID(pid.get());
    state.http = false;

    return state;
  }

  path = paths::getExecutorHttpMarkerPath(
      rootDir, slaveId, frameworkId, executorId, containerId);

  if (!os::exists(path)) {
    // This could happen if the slave died before the executor
    // registered with the slave.
    LOG(WARNING) << "Failed to find executor libprocess pid/http marker file";
    return state;
  }

  state.http = true;

  return state;
}

~~~


###TaskState
在这里终于看到 ***StatusUpdateRecord***的读取了。

~~~cpp
Try<TaskState> TaskState::recover(
    const string& rootDir,
    const SlaveID& slaveId,
    const FrameworkID& frameworkId,
    const ExecutorID& executorId,
    const ContainerID& containerId,
    const TaskID& taskId,
    bool strict)
{
  TaskState state;
  state.id = taskId;
  string message;

  // Read the task info.
  string path = paths::getTaskInfoPath(
      rootDir, slaveId, frameworkId, executorId, containerId, taskId);
  if (!os::exists(path)) {
    // This could happen if the slave died after creating the task
    // directory but before it checkpointed the task info.
    LOG(WARNING) << "Failed to find task info file '" << path << "'";
    return state;
  }

  const Result<Task>& task = ::protobuf::read<Task>(path);

  if (task.isError()) {
    message = "Failed to read task info from '" + path + "': " + task.error();

    if (strict) {
      return Error(message);
    } else {
      LOG(WARNING) << message;
      state.errors++;
      return state;
    }
  }

  if (task.isNone()) {
    // This could happen if the slave died after opening the file for
    // writing but before it checkpointed anything.
    LOG(WARNING) << "Found empty task info file '" << path << "'";
    return state;
  }

  state.info = task.get();

  // Read the status updates.
  path = paths::getTaskUpdatesPath(
      rootDir, slaveId, frameworkId, executorId, containerId, taskId);
  if (!os::exists(path)) {
    // This could happen if the slave died before it checkpointed any
    // status updates for this task.
    LOG(WARNING) << "Failed to find status updates file '" << path << "'";
    return state;
  }

  // Open the status updates file for reading and writing (for
  // truncating).
  Try<int> fd = os::open(path, O_RDWR | O_CLOEXEC);

  if (fd.isError()) {
    message = "Failed to open status updates file '" + path +
              "': " + fd.error();

    if (strict) {
      return Error(message);
    } else {
      LOG(WARNING) << message;
      state.errors++;
      return state;
    }
  }

  // Now, read the updates.
  Result<StatusUpdateRecord> record = None();
  while (true) {
    // Ignore errors due to partial protobuf read and enable undoing
    // failed reads by reverting to the previous seek position.
    record = ::protobuf::read<StatusUpdateRecord>(fd.get(), true, true);

    if (!record.isSome()) {
      break;
    }

    if (record.get().type() == StatusUpdateRecord::UPDATE) {
      state.updates.push_back(record.get().update());
    } else {
      state.acks.insert(UUID::fromBytes(record.get().uuid()));
    }
  }

  off_t offset = lseek(fd.get(), 0, SEEK_CUR);

  if (offset < 0) {
    os::close(fd.get());
    return ErrnoError("Failed to lseek status updates file '" + path + "'");
  }

  // Always truncate the file to contain only valid updates.
  // NOTE: This is safe even though we ignore partial protobuf read
  // errors above, because the 'fd' is properly set to the end of the
  // last valid update by 'protobuf::read()'.
  Try<Nothing> truncated = os::ftruncate(fd.get(), offset);

  if (truncated.isError()) {
    os::close(fd.get());
    return Error(
        "Failed to truncate status updates file '" + path +
        "': " + truncated.error());
  }

  // After reading a non-corrupted updates file, 'record' should be
  // 'none'.
  if (record.isError()) {
    message = "Failed to read status updates file  '" + path +
              "': " + record.error();

    os::close(fd.get());

    if (strict) {
      return Error(message);
    } else {
      LOG(WARNING) << message;
      state.errors++;
      return state;
    }
  }

  // Close the updates file.
  os::close(fd.get());

  return state;
}


##StatusUpdateManagerProcess

这里把恢复的SlaveState在重新播放一遍。然后事情就正常了。

~~~cpp
Future<Nothing> StatusUpdateManagerProcess::recover(
    const string& rootDir,
    const Option<SlaveState>& state)
{
  LOG(INFO) << "Recovering status update manager";

  if (state.isNone()) {
    return Nothing();
  }

  foreachvalue (const FrameworkState& framework, state.get().frameworks) {
    foreachvalue (const ExecutorState& executor, framework.executors) {
      LOG(INFO) << "Recovering executor '" << executor.id
                << "' of framework " << framework.id;

      if (executor.info.isNone()) {
        LOG(WARNING) << "Skipping recovering updates of"
                     << " executor '" << executor.id
                     << "' of framework " << framework.id
                     << " because its info cannot be recovered";
        continue;
      }

      if (executor.latest.isNone()) {
        LOG(WARNING) << "Skipping recovering updates of"
                     << " executor '" << executor.id
                     << "' of framework " << framework.id
                     << " because its latest run cannot be recovered";
        continue;
      }

      // We are only interested in the latest run of the executor!
      const ContainerID& latest = executor.latest.get();
      Option<RunState> run = executor.runs.get(latest);
      CHECK_SOME(run);

      if (run.get().completed) {
        VLOG(1) << "Skipping recovering updates of"
                << " executor '" << executor.id
                << "' of framework " << framework.id
                << " because its latest run " << latest.value()
                << " is completed";
        continue;
      }

      foreachvalue (const TaskState& task, run.get().tasks) {
        // No updates were ever received for this task!
        // This means either:
        // 1) the executor never received this task or
        // 2) executor launched it but the slave died before it got any updates.
        if (task.updates.empty()) {
          LOG(WARNING) << "No updates found for task " << task.id
                       << " of framework " << framework.id;
          continue;
        }

        // Create a new status update stream.
        StatusUpdateStream* stream = createStatusUpdateStream(
            task.id, framework.id, state.get().id, true, executor.id, latest);

        // Replay the stream.
        Try<Nothing> replay = stream->replay(task.updates, task.acks);
        if (replay.isError()) {
          return Failure(
              "Failed to replay status updates for task " + stringify(task.id) +
              " of framework " + stringify(framework.id) +
              ": " + replay.error());
        }

        // At the end of the replay, the stream is either terminated or
        // contains only unacknowledged, if any, pending updates. The
        // pending updates will be flushed after the slave
        // re-registers with the master.
        if (stream->terminated) {
          cleanupStatusUpdateStream(task.id, framework.id);
        }
      }
    }
  }

  return Nothing();
}
~~~


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


从这里的resume可以看出，前面恢复到pending的，就会继续forward到master去了：）

