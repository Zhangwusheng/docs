

#第二章： Mesos架构

**Mesos Architecture**

Mesos has an architecture that is composed of master and slave daemons, and frameworks. Here is a quick breakdown of these components, and some relevant terms:

* Master daemon: runs on a master node and manages slave daemons
* Slave daemon: runs on a master node and runs tasks that belong to frameworks
* Framework: also known as a Mesos application, is composed of a scheduler, which registers with the master to receive resource offers, and one or more executors, which launches tasks on slaves. Examples of Mesos frameworks include Marathon, Chronos, and Hadoop
* Offer: a list of a slave node's available CPU and memory resources. All slave nodes send offers to the master, and the master provides offers to registered frameworks
* Task: a unit of work that is scheduled by a framework, and is executed on a slave node. A task can be anything from a bash command or script, to an SQL query, to a Hadoop job
* Apache ZooKeeper: software that is used to coordinate the master nodes


![MacDown Screenshot](file:///Users/zhangwusheng/Documents/mesos-doc/mesos_architecture.png)

**Mesos Architecture-2**

![MacDown Screenshot](file:///Users/zhangwusheng/Documents/mesos-doc/mesos-architecture-2.png)

##主要组件以及特性

**1. Basic components of a Mesos Cluster**

* Many Mesos Slaves 
* Schedulers Also called frameworks 
* Executors

**2. All communication done via HTTP**

**3. Communication flow between components**

##Example


**Slave** running at 192.168.0.7:5051


1. **Discovery between Master to Slaves**
  Slaves announce themselves to the Master。
  
   Master pings slave:
        Connection: Keep-Alive
        
  
        User-Agent: libprocess/  slave(1)@192.168.0.7:5051 
        Connection: Keep-Alive
        
2. Scheduler starts and Registers to the master

        Host: 192.168.0.7:5050
        Libprocess-From:    scheduler(1)@192.168.0.7:59508     
        Accept-Encoding: gzip

3. Master ACKs the registering to the scheduler

        User-Agent: libprocess/master@192.168.0.7:5050
4. Then Master starts giving resources to the Scheduler

        User-Agent: libprocess/master@192.168.0.7:5050
5. Scheduler accumulates offerings and launches tasks to the Master

   The Master will give an Slave resource to run the job.
   
        POST /master/mesos.internal.LaunchTasksMessage HTTP/1.1 
        Host: 192.168.0.7:5050
        Libprocess-From: scheduler(1)@192.168.0.7:59508 
        Accept-Encoding: gzip
6. Master submits job from scheduler to the Slave

        POST /slave(1)/mesos.internal.RunTaskMessage HTTP/1.0 
        User-Agent: libprocess/master@192.168.0.7:5050   
        Connection: Keep-Alive
        
7. Executor is started and registers back to the Slave

        POST /slave(1)/mesos.internal.RegisterExecutorMessage HTTP/1.0 
        User-Agent: libprocess/executor(1)@192.168.0.7:58006 
        Connection: Keep-Alive
   
8. Slave ACKs to the executor that it is aware of it

        POST /executor(1)/mesos.internal.ExecutorRegisteredMessage HTTP/1.0 
        User-Agent: libprocess/slave(1)@192.168.0.7:5051
        
9. Then Slave submits a job to the Executor

        POST /executor(1)/mesos.internal.RunTaskMessage HTTP/1.0 
        User-Agent: libprocess/slave(1)@192.168.0.7:5051 
        Connection: Keep-Alive
10. Executor will constantly be sharing status to the slave

        POST /slave(1)/mesos.internal.StatusUpdateMessage HTTP/1.0 
        User-Agent: libprocess/executor(1)@192.168.0.7:58006 
        Connection: Keep-Alive
11. Then the Slave will escalate the status to the Master
        
        POST /master/mesos.internal.StatusUpdateMessage HTTP/1.0 
        User-Agent: libprocess/slave(1)@192.168.0.7:5051 
        Connection: Keep-Alive
        
 And so on, and so on...


**Responsibilities of the Scheduler and Executor**



* Run tasks
 
