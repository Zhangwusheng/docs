#第1章：Mesos初探
本章我们介绍如何从源码安装mesos，并且使用一个实际可以运行的例子来演示mesos的基本原理，看看mesos是如何部署和运行的以及如何基于mesos进行开发。

注：软件的安装看个人喜好，本人喜欢将自己安装的软件安装在/usr/local下面。后续均以此为准。

##1.1 源代码下载
* export MESOS_INSTALL=~/soft

* cd $MESOS_INSTALL

* wget http://www-eu.apache.org/dist/mesos/0.27.2/mesos-0.27.2.tar.gz

* tar zxcv mesos-0.27.2.tar.gz

##1.2 Ubuntu源码编译

* 操作系统版本：Ubuntu 14.04

* 编译器：g++ 4.9.2 

因为mesos源代码里面用到了大量的C++11的内容，所以编译器要支持才行。

###1.2.1 安装前准备
确保如下包已经被安装

* make
* cmake
* git
* autoconf
* libunwind-dev
* libcurl4-nss-dev
* libapr1-dev
* libsvn-dev
* build-essential
* libsasl2-dev
* python-dev
* zlib1g-dev
* libsasl2-modules
* python-boto
* libboost-all-dev

以上包均可以通过如下命令进行安装：

apt-get -y install 包名

另外，建议手工安装配置Java环境和Maven环境：

1. 建议安装OracleJDK jdk-7u76-linux-x64.tar.gz，可以从Oracle官网下载
2. 安装Maven,本人安装的是apache-maven-3.3.9.tar.gz
3. 安装Zookeeper。我使用的是zookeeper-3.4.6.tar.gz

安装好后，把PATH环境变量设置好。使用如下面命令验证：

* java -version
* mvn -version

如果都能正常运行，说明安装无误，有问题请自行搜索解决方案。

###1.2.2 编译源码
* cd $MESOS_INSTALL/mesos-0.27.2
* mkdir build
* cd build
* ../configure --prefix=/usr/local/mesos
* make -j 8
* make install

经过漫长的等待之后，程序编译完成。安装到了/usr/local/mesos下面。


##1.2.3 安装phantomjs
由于我们的示例中使用了phantomjs来实现executor的功能，因此需要首先安转这个软件。此处对phantomjs做一个简要介绍：

PhantomJS是一个基于WebKit的服务器端JavaScript API，它基于 BSD开源协议发布。PhantomJS无需浏览器的支持即可实现对Web的支持，且原生支持各种Web标准，如DOM 处理、JavaScript、CSS选择器、JSON、Canvas和可缩放矢量图形SVG。
其他功能请自行了解，此处我们使用phantomjs来实现Web页面的屏幕捕获。

* wget -c https://bitbucket.org/ariya/phantomjs/downloads/phantomjs-2.1.1-linux-x86_64.tar.bz2
* tar jxvf phantomjs-2.1.1-linux-x86_64.tar.bz2
* mv phantomjs-2.1.1-linux-x86_64 /usr/local
* ln -fs /usr/local/phantomjs-2.1.1-linux-x86_64 /usr/local/phantomjs
* cat <<EOF > /etc/profile.d/phantomjs.sh

    export PATH=\$PATH:/usr/local/phantomjs/bin
EOF
* source /etc/profile.d/phantomjs.sh

    
##1.2.4 编译RENDLER

RENDLER是运行于mesos集群的一个示例Framework，主要功能是实现一个简单的分布式爬虫。
   
* cd $MESOS_INSTALL
* wget https://github.com/mesosphere/RENDLER/archive/master.zip
* unzip master.zip
* apt-get -y install libgoogle-glog-dev
* apt-get -y install libprotobuf-dev 
* apt-get -y install fontconfig
* apt-get -y install libboost-regex-dev
* echo '/usr/local/mesos/lib' > /etc/ld.so.conf.d/mesos.conf
* ldconfig
* cd RENDLER-master/cpp;make

编译完毕后,会产生三个可执行文件：

1. crawl_executor
2. render_executor
3. rendler

其中前两个是mesos里面的executor，第三个是mesos的Framework。

##1.2.5 启动集群：

* 启动master（一行放不下，另行写）：

		/usr/local/mesos/sbin/mesos-master 
			--work_dir=/disk1/data/mesos/master/ 
			--zk=zk://192.168.1.193:5181/mesos 
			--quorum=2 --log_dir=/disk1/data/mesos/logs

* 启动slave：

		/usr/local/mesos/sbin/mesos-slave 
			--work_dir=/disk1/data/mesos/slave 
			--master=zk://192.168.1.193:5181/mesos  
			--log_dir=/disk1/data/mesos/logs 
			--no-systemd_enable_support
			
	* 如果需要使用hadoop，使mesos自动从hadoop下载executor的jar等，可以加上这个参数(具体代码可以参见launcher/fetcher.cpp)：
	    
	    --hadoop_home=/usr/local/hadoop
		
* 提交rendler

		cd RENDLER-master/cpp;
		mkdir rendler-work-dir

		./rendler --master zk://192.168.1.193:5181/mesos  
			--seedUrl=YourURL

***注：如果运行出错的话，建议修改render_executor.cpp:57:***

***string cmd = "/usr/local/phantomjs/bin/phantomjs  "+ renderJSPath + " " + url + " " + filename;***

***将phantomjs绝对路径写进去***

运行后，你会看到rendler-work-dir目录下面有生成的网页的截图。
恭喜你，你已经成功运行了第一个运行于mesos之上的Framework！

##1.3 MAC源码编译
由于本人比较喜欢在Xcode下面阅读C++源代码，因此此处介绍一下如何在MAC下面编译mesos代码：


* xcode-select --install
* brew install wget git autoconf 
* brew install automake libtool subversion
* brew install apr

以本人习惯，建议手工安装java和maven,配置环境变量PATH。

* java -version
* mvn -version

验证无误后，运行如下命令：

* cd $MESOS_INSTALL/mesos-0.27.2
* ../configure --with-apr=/usr/local/Cellar/apr/1.5.2/libexec/ --disable-python --prefix=/usr/local/mesos

***注：configure时可能会报找不到apr-1的错误，因此本人加上了--with-apr参数。由于只是为了在mac上阅读和调试代码，因此省却了python的编译。***

##1.4 RENDLER源代码一览

示例代码有三个源文件：
后续大家会看到，我们要编写的mesos Framework主要是编写自己的Scheduler和Executor。

###rendler.cpp：

####1. main函数
~~~cpp

int main(int argc, char** argv)
{
  string seedUrl, master;
  shift;
  while (true) {
    string s = argc>0 ? argv[0] : "--help";
    if (argc > 1 && s == "--seedUrl") {
      seedUrl = argv[1];
      shift; shift;
    } else if (argc > 1 && s == "--master") {
      master = argv[1];
      shift; shift;
    } else {
      break;
    }
  }

  if (master.length() == 0 || seedUrl.length() == 0) {
    printf("Usage: rendler --seedUrl <URL> --master <ip>:<port>\n");
    exit(1);
  }
~~~
  
以上代码主要是命令行参数解析，不详述。
  

  
~~~cpp
  // Find this executable's directory to locate executor.
  string path = realpath(dirname(argv[0]), NULL);
  string crawlerURI = path + "/crawl_executor";
  string rendererURI = path + "/render_executor";
  cout << crawlerURI << endl;
  cout << rendererURI << endl;
~~~
因为三个程序在同一个目录，我们运行的是rendler，但是需要获取另外两个程序(crawl_executor及render_executor)，所以这里把程序自己的绝对目录读出来。主要是为了设置下面两个executor的绝对路径。

~~~cpp
  ExecutorInfo crawler;            [1]
  crawler.mutable_executor_id()->set_value("Crawler");
  crawler.mutable_command()->set_value(crawlerURI);
  crawler.set_name("Crawl Executor (C++)");
  crawler.set_source("cpp");

  ExecutorInfo renderer;
  renderer.mutable_executor_id()->set_value("Renderer");
  renderer.mutable_command()->set_value(rendererURI);
  renderer.set_name("Render Executor (C++)");
  renderer.set_source("cpp");

  Rendler scheduler(crawler, renderer, seedUrl);  [2]
  //设置FrameworkInfo。
  FrameworkInfo framework;        
  framework.set_user(""); // Have Mesos fill in the current user.
  framework.set_name("Rendler Framework (C++)");
  //framework.set_role(role);
  framework.set_principal("rendler-cpp");
~~~

设置了两个ExecutorInfo对象，ExecutorInfo是ProtoBuffer消息。此处主要是设置Executor对应的程序路径。

~~~cpp
  schedulerDriver = new MesosSchedulerDriver(&scheduler, framework, master);
~~~
这是最主要的一行，MesosSchedulerDriver负责与Mesos Master/Slave通信，在合适时机调用scheduler的回调函数。

~~~cpp  

  int status = schedulerDriver->run() == DRIVER_STOPPED ? 0 : 1;

  // Ensure that the driver process terminates.
  schedulerDriver->stop();

  shutdown();

  delete schedulerDriver;
  return status;
}

~~~
启动MesosSchedulerDriver，driver一直处于接收消息->处理消息的循环中，然后等待程序返回。

####2. 全局变量列表
示例程序为了简单起见，一些必要的信息都用全局变量来保存。

~~~cpp
const float CPUS_PER_TASK = 0.2;  //mesos CPU资源
const int32_t MEM_PER_TASK = 32;  //mesos Memory资源

static queue<string> crawlQueue;  
static queue<string> renderQueue;
static map<string, vector<string> > crawlResults;
static map<string, string> renderResults;
static map<string, size_t> processed;
static size_t nextUrlId = 0;

MesosSchedulerDriver* schedulerDriver;
~~~
####2. Rendler
I
~~~cpp
class Rendler : public Scheduler
{
...
~~~

当这个Framework被分配到资源的时候，会回调这个函数

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
~~~
上面主要是设置每个任务需要的资源配置。

~~~cpp
      size_t maxTasks = 0;
      while (remaining.flatten().contains(TASK_RESOURCES)) {
        maxTasks++;
        remaining -= TASK_RESOURCES;
      }
~~~
上面计算可以启动多少个任务。mesos提供了Resource和Resources类代表资源，也实现了很多算数操作符的重载，所以上面能够出现remaining -= TASK_RESOURCES;这样的写法。

~~~cpp
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
      
~~~

~~~cpp      
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
  ...
  
~~~



###运行自己的Framework
java版本的必须要设置一下这个变量：

export MESOS\_NATIVE\_JAVA_LIBRARY=/usr/local/mesos/lib/libmesos.so

java -cp ./cronserver-1.0-jar-with-dependencies.jar org.apache.mesos.chronos.Main /disk1/soft/cronserver/acs-config.properties 

ssh -p 33222 -NCPf zhangwusheng@219.135.99.170 -i /Users/zhangwusheng/Documents/SecureCRT-Config/Identity_zhangwusheng -L 9998:192.168.1.194:9998



java -cp /disk1/soft/cronserver/cronserver-1.0-jar-with-dependencies.jar -Xdebug -Xrunjdwp:transport=dt_socket,server=y,address=9998  org.apache.mesos.chronos.Main /disk1/soft/cronserver/acs-config.properties 


###

###IOS爱拍Git地址

