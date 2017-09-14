
#PLAYING TRAFFIC COP: RESOURCE ALLOCATION IN APACHE MESOS

[http://cloudarchitectmusings.com/2015/04/08/playing-traffic-cop-resource-allocation-in-apache-mesos/](http://cloudarchitectmusings.com/2015/04/08/playing-traffic-cop-resource-allocation-in-apache-mesos/)

A key capability that makes Apache Mesos a viable uber resource manager for the data center is its ability to play traffic cop amongst a diversity of workloads.  The purpose of this post will be to dig into the internals of resource allocation in Mesos and how it balances fair resource sharing with customer workload needs.  Before going on, you may want to read the other posts in this series if you have not already done so.  An overview of Mesos can be found [here](http://cloudarchitectmusings.com/2015/03/23/apache-mesos-the-true-os-for-the-software-defined-data-center/), followed by an explanation of its two-level architecture [here](http://cloudarchitectmusings.com/2015/03/26/digging-deeper-into-apache-mesos/), and then a post [here](http://cloudarchitectmusings.com/2015/03/31/dealing-with-persistent-storage-and-fault-tolerance-in-apache-mesos/) on data storage and on fault tolerance.  I also write about what I see is [going right](http://wp.me/p2MZ5x-IK) in the Mesos project.  If you are interested in spinning up and trying out Mesos, I link to some resources in another [blog post](http://cloudarchitectmusings.com/2015/04/30/trying-out-apache-mesos/).


We will be walking through the role of the Mesos resource allocation module, how it decides what resources to offer to which frameworks, and how it reclaims resources when necessary.  But let’s start by reviewing the task scheduling process in Mesos:

![MacDown Screenshot](file:///Users/zhangwusheng/Documents//GitHub/docs/md-doc/mesos-framework-example1.jpg)



If you recall from my earlier architecture [post](http://cloudarchitectmusings.com/2015/03/26/digging-deeper-into-apache-mesos/), the Mesos master delegates the scheduling of tasks by collecting information about available resources from slave nodes and then offering those resources to registered frameworks in the form of resource offers.  Frameworks have the option of accepting resources offers or rejecting them if they do not meet task constraints.  Once a resource offer is accepted, a framework will coordinate with the master to schedule and to run tasks on appropriate slave nodes in the data center.


Responsibility for how resource offer decisions are made is delegated to the resource allocation module which lives with the master.  The resource allocation module determines the order in which frameworks are offered resources and has to do so while making sure that resources are fairly shared among inherently greedy frameworks.  In a homogenous environment, such as a Hadoop cluster, one of the most popular fair share allocation algorithms in use is max-min fairness.  [Max-min fairness](http://en.wikipedia.org/wiki/Max-min_fairness) maximizes the minimum allocation of resources offered to a user with the goal of ensuring that each user receives a fair share of resources needed to satisfy its requirements; for a simple example of how this might work, check out Example 1 on this [max-min fair share algorithm page](http://www.ece.rutgers.edu/~marsic/Teaching/CCN/minmax-fairsh.html).  This generally works well, as mentioned in a homogeneous environment where resource demands have little fluctuation as it pertains to the types of resources, e.g. CPU, RAM, network bandwidth, I/O.  Resource allocation becomes more difficult when you are scheduling resources across a data center with heterogeneous resource demands.  For example, what is a suitable fair share allocation policy if user A runs tasks that require (1 CPU, 4GB RAM) each and User B runs tasks that require (3 CPUs, 1GB RAM) each?  How do you fairly allocate a bundle of resources when User A tasks are RAM heavy and User B tasks are CPU heavy?

Since Mesos is specifically focused on managing resources in a heterogeneous environment, it implements a pluggable resource allocation module architecture that allows users to create allocation policies with algorithms that best fit a particular deployment.  For example, a user could implement a weighted max-min fairness algorithm that will give a specific framework a greater share of resources relative to other frameworks.  By default, Mesos includes a strict priority resource allocation module and a modified fair sharing resource allocation module.  The strict priority module implements an algorithm that gives a framework priority to always receive and accept resource offers sufficient to meet its task requirements.  It guarantees resources for critical workloads at the expense of restricting dynamic resource sharing in Mesos and potentially starving other frameworks.

For those reasons, most users default to Dominant Resource Fairness, a modified fair share algorithm in Mesos that is more suitable for heterogeneous environments.  Dominant Resource Fairness (DRF) was proposed by the same Berkeley AMPLab team that created Mesos and coded in as the default resource allocation policy in Mesos.  You can read the original papers on DRF [here](https://www.cs.berkeley.edu/~alig/papers/drf.pdf) and [here](http://www.eecs.berkeley.edu/Pubs/TechRpts/2010/EECS-2010-55.pdf).  For this blog post, I’ll attempt to summarize the main points and provide some examples that will hopefully make things more clear.  Time to get under the hood.



The goal of DRF is to ensure that each user, a framework in the case of Mesos, in a heterogeneous environment receives a fair share of the resource most needed by that user/framework.  To grasp DRF, you need to understand the concepts of dominant resource and dominant share.  A framework’s dominant resource is the resource type (CPU, RAM, etc.) that is most in demand by that framework, as a percentage of available resources presented to it by a given resource offer.  For example, a task that is computation-heavy will have CPU as its framework’s dominant resource while a task that relies on in-memory calculations may have RAM as its framework’s dominant resource.  As frameworks are allocated resources, DRF tracks the percentages of shares that each framework owns for a given resource type; the highest percentage of shares owned across all resource types for a given framework is that framework’s dominant share.  The DRF algorithm uses the dominant share of all registered frameworks to ensure that each framework receives a fair share of its dominant resource.

![MacDown Screenshot](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/md-mesos-doc/mesod_screenshot_20150407_at_949_36_pm.png）

Too abstract a concept?  Let’s illustrate with an example.  Assume you have a resource offer that consists of 9 CPUs and 18 GB of RAM.  Framework 1 runs tasks that require (1 CPU, 4 GB RAM) and framework 2 runs tasks that require (3 CPUs, 1 GB RAM).  Each framework 1 task will consume 1/9 of the total CPU and 2/9 of the total RAM, making RAM framework 1’s dominant resource.  Likewise, each framework 2 task will consume 1/3 of the total CPU and 1/18 of the total RAM, making CPU framework 2’s dominant resource.  DRF will attempt to give each framework an equal amount of their dominant resource, as represented by their dominant share.  In this example, DRF will work with the frameworks to allocate the following – three tasks to framework 1 for a total allocation of (3 CPUs, 12 GB RAM) and 2 tasks to framework 2 for a total allocation of (6 CPUs, 2 GB RAM).  In this case, each framework ends up with the same dominant share (2/3 or 67%) for each framework’s dominant resource (RAM for framework 1 and CPU for framework 2) before there is not enough available resources, in this particular offer, to run additional tasks.  Note that if framework 1, for example, only had 2 tasks that needed to be run, then framework 2 and any other registered frameworks would have received all left over resources.

![MacDown Screenshot](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/md-mesos-doc/mesod_screenshot_20150407_at_949_36_pm.png)

So how does DRF work to produce the outcome above?  As stated earlier, the DRF allocation module tracks the resources allocated to each framework and each framework’s dominant share.  At each step, DRF presents resource offers to the framework with the lowest dominant share among all frameworks with tasks to run.  The framework will accept the offer if there are enough available resources to run its task.  Using the example taken from the DRF papers cited earlier and modified for this blog post, I will walk through each step taken by the DRF algorithm.  To keep things simple, the example I use do not factor in resources that are released back to the pool after a short-running task is complete, I assume there is an infinite number of tasks to be run for each framework, and I assume that every resource offer is accepted.

If you recall from above, we assume you have a resource offer that consists of 9 CPUs and 18 GB of RAM.  Framework 1 runs tasks that require (1 CPU, 4 GB RAM) and framework 2 runs tasks that require (3 CPUs, 1 GB RAM).  Each framework 1 task will consume 1/9 of the total CPU and 2/9 of the total RAM, making RAM framework 1’s dominant resource.  Again, each framework 2 task will consume 1/3 of the total CPU and 1/18 of the total RAM, making CPU framework 2’s dominant resource.


![MacDown Screenshot](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/md-mesos-doc/mesos-drf1.jpg)

Each row provides the following information:

* Framework chosen – The framework that has been given the latest resource offer
* Resource Shares – Total resources accepted at a given time by a given framework by CPU and by RAM, expressed as fraction of total resources
* Dominant Share – Percentage of total shares accepted for a given framework’s dominant resource at a given time, expressed as fraction of total resources
* Dominant Share % – Percentage of total shares accepted for a given framework’s dominant resource at a given time, expressed as percentage of total resources
* CPU Total Allocation – Total CPU resources accepted by all frameworks at a given time
* RAM Total Allocation – Total RAM resources accepted by all frameworks at a given time
Note also that the lowest dominant share in each row can be found in bold.

Although Initially both frameworks have a dominant share of 0%, we’ll assume that DRF chooses framework 2 to offer resources to first though we could have assumed framework 1 and the final outcome would still be the same.

1. Framework 2 receives its shares to run a task causing the dominate share for its dominant resource (CPU) to be increased to 33%.
2. Since framework 1’s dominant share remained at 0%, it receives shares next to run a task and the dominant share for its dominant resource (RAM) is increased to 22%.
3. Since framework 1 continues to have the lower dominant share, it receives the next shares as well to run a task, increasing its dominant share to 44%.
4. DRF then offer resources to framework 2 since it now has the lower dominant share.
5. The process continues until it is not possible to run any more new tasks due to lack of available resources.  In this case, CPU resources have been saturated.
6. The process will then repeat with a new set of resource offers


Note that is possible to create a resource allocation module that uses weighted DRF to favor one framework or set of frameworks over another.  And as mentioned earlier, it is possible to create a number of other custom modules to provide organizational specific allocation policies.

Now under normal circumstances, most tasks are short-lived and Mesos is able to wait and reallocate resources when a task is finished.  However, it is possible that a cluster can be filled with long-running tasks due to a hung job or a badly behaving framework.  It is worth noting that in such a circumstance when resources are not being freed quickly enough, the resource allocation module has the ability to revoke tasks.  Mesos attempts to revoke a task by first requesting that an executor kill a given task and gives a grace period for clean up by the executor.  If the executor does not respond to the request, the allocation module then kills the executor and all its tasks.  An allocation policy can be implemented that prevents a given task from being revoked by providing the associated framework a guaranteed allocation.  If a framework is below its guaranteed allocation, Mesos will not be able to kill its tasks.

There is more to know about resource allocation in Mesos, but I’ll stop here.  Next week, I will do something a little different and blog about the Mesos community.  I believe that is an important topic to consider since open source is not just about the technology but also about the community.  After that, I will try to post some step-by-step tutorials on setting up Mesos and using and creating frameworks.

As always, I encourage readers to provide feedback, especially regarding if I am hitting the mark with these posts and if you see any errors that need to be corrected.  I am a learner and do not pretend to know all the answers; so correction and enlightenment is always welcomed.  I also respond on twitter at @kenhuiny.



中文版：
http://www.infoq.com/cn/articles/analyse-mesos-part-04
http://www.infoq.com/cn/articles/analyse-mesos-part-01
http://www.infoq.com/cn/articles/analyse-mesos-part-02
http://www.infoq.com/cn/articles/analyse-mesos-part-03


Apache Mesos能够成为最优秀的数据中心资源管理器的一个重要功能是面对各种类型的应用，它具备像交警一样的疏导能力。本文将深入Mesos的资源分配内部，探讨Mesos是如何根据客户应用需求，平衡公平资源共享的。在开始之前，如果读者还没有阅读这个系列的前序文章，建议首先阅读它们。第一篇是Mesos的概述，第二篇是两级架构的说明，第三篇是数据存储和容错。

我们将探讨Mesos的资源分配模块，看看它是如何确定将什么样的资源邀约发送给具体哪个Framework，以及在必要时如何回收资源。让我们先来回顾一下Mesos的任务调度过程：



从前面提到的两级架构的说明一文中我们知道，Mesos Master代理任务的调度首先从Slave节点收集有关可用资源的信息，然后以资源邀约的形式，将这些资源提供给注册其上的Framework。

Framework可以根据是否符合任务对资源的约束，选择接受或拒绝资源邀约。一旦资源邀约被接受，Framework将与Master协作调度任务，并在数据中心的相应Slave节点上运行任务。

如何作出资源邀约的决定是由资源分配模块实现的，该模块存在于Master之中。资源分配模块确定Framework接受资源邀约的顺序，与此同时，确保在本性贪婪的Framework之间公平地共享资源。在同质环境中，比如Hadoop集群，使用最多的公平份额分配算法之一是最大最小公平算法（max-min fairness）。最大最小公平算法算法将最小的资源分配最大化，并将其提供给用户，确保每个用户都能获得公平的资源份额，以满足其需求所需的资源；一个简单的例子能够说明其工作原理，请参考最大最小公平份额算法页面的示例1。如前所述，在同质环境下，这通常能够很好地运行。同质环境下的资源需求几乎没有波动，所涉及的资源类型包括CPU、内存、网络带宽和I/O。然而，在跨数据中心调度资源并且是异构的资源需求时，资源分配将会更加困难。例如，当用户A的每个任务需要1核CPU、4GB内存，而用户B的每个任务需要3核CPU、1GB内存时，如何提供合适的公平份额分配策略？当用户A的任务是内存密集型，而用户B的任务是CPU密集型时，如何公平地为其分配一揽子资源？

因为Mesos是专门管理异构环境中的资源，所以它实现了一个可插拔的资源分配模块架构，将特定部署最适合的分配策略和算法交给用户去实现。例如，用户可以实现加权的最大最小公平性算法，让指定的Framework相对于其它的Framework获得更多的资源。默认情况下，Mesos包括一个严格优先级的资源分配模块和一个改良的公平份额资源分配模块。严格优先级模块实现的算法给定Framework的优先级，使其总是接收并接受足以满足其任务要求的资源邀约。这保证了关键应用在Mesos中限制动态资源份额上的开销，但是会潜在其他Framework饥饿的情况。

由于这些原因，大多数用户默认使用DRF（主导资源公平算法 Dominant Resource Fairness），这是Mesos中更适合异质环境的改良公平份额算法。

DRF和Mesos一样出自Berkeley AMPLab团队，并且作为Mesos的默认资源分配策略实现编码。

读者可以从此处和此处阅读DRF的原始论文。在本文中，我将总结其中要点并提供一些例子，相信这样会更清晰地解读DRF。让我们开始揭秘之旅。

DRF的目标是确保每一个用户，即Mesos中的Framework，在异质环境中能够接收到其最需资源的公平份额。为了掌握DRF，我们需要了解主导资源（dominant resource）和主导份额（dominant share）的概念。Framework的主导资源是其最需的资源类型（CPU、内存等），在资源邀约中以可用资源百分比的形式展示。例如，对于计算密集型的任务，它的Framework的主导资源是CPU，而依赖于在内存中计算的任务，它的Framework的主导资源是内存。因为资源是分配给Framework的，所以DRF会跟踪每个Framework拥有的资源类型的份额百分比；Framework拥有的全部资源类型份额中占最高百分比的就是Framework的主导份额。DRF算法会使用所有已注册的Framework来计算主导份额，以确保每个Framework能接收到其主导资源的公平份额。

概念过于抽象了吧？让我们用一个例子来说明。假设我们有一个资源邀约，包含9核CPU和18GB的内存。Framework 1运行任务需要（1核CPU、4GB内存），Framework 2运行任务需要（3核CPU、1GB内存）
Framework 1的每个任务会消耗CPU总数的1/9、内存总数的2/9，因此Framework 1的主导资源是内存。同样，Framework 2的每个任务会CPU总数的1/3、内存总数的1/18，因此Framework 2的主导资源是CPU。DRF会尝试为每个Framework提供等量的主导资源，作为他们的主导份额。在这个例子中，DRF将协同Framework做如下分配：Framework 1有三个任务，总分配为（3核CPU、12GB内存），Framework 2有两个任务，总分配为（6核CPU、2GB内存）。

此时，每个Framework的主导资源（Framework 1的内存和Framework 2的CPU）最终得到相同的主导份额（2/3或67％），这样提供给两个Framework后，将没有足够的可用资源运行其他任务。需要注意的是，如果Framework 1中仅有两个任务需要被运行，那么Framework 2以及其他已注册的Framework将收到的所有剩余的资源。

![MacDown Screenshot](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/md-mesos-doc/mesod_screenshot_20150407_at_949_36_pm.png)

那么，DRF是怎样计算而产生上述结果的呢？如前所述，DRF分配模块跟踪分配给每个Framework的资源和每个框架的主导份额。每次，DRF以所有Framework中运行的任务中最低的主导份额作为资源邀约发送给Framework。如果有足够的可用资源来运行它的任务，Framework将接受这个邀约。通过前面引述的DRF论文中的示例，我们来贯穿DRF算法的每个步骤。为了简单起见，示例将不考虑短任务完成后，资源被释放回资源池中这一因素，我们假设每个Framework会有无限数量的任务要运行，并认为每个资源邀约都会被接受。

回顾上述示例，假设有一个资源邀约包含9核CPU和18GB内存。Framework 1运行的任务需要（1核CPU、4GB内存），Framework 2运行的任务需要（3核CPU、2GB内存）。Framework 1的任务会消耗CPU总数的1/9、内存总数的2/9，Framework 1的主导资源是内存。同样，Framework 2的每个任务会CPU总数的1/3、内存总数的1/18，Framework 2的主导资源是CPU。

![MacDown Screenshot](file:///Users/zhangwusheng/Documents/GitHub/docs/md-doc/md-mesos-doc/mesos-drf1.jpg)


上面表中的每一行提供了以下信息：

Framework chosen——收到最新资源邀约的Framework。
Resource Shares——给定时间内Framework接受的资源总数，包括CPU和内存，以占资源总量的比例表示。
Dominant Share（主导份额）——给定时间内Framework主导资源占总份额的比例，以占资源总量的比例表示。
Dominant Share %（主导份额百分比）——给定时间内Framework主导资源占总份额的百分比，以占资源总量的百分比表示。
CPU Total Allocation——给定时间内接受的所有Framework的总CPU资源。
RAM Total Allocation——给定时间内接受的所有Framework的总内存资源。
注意，每个行中的最低主导份额以粗体字显示，以便查找。

最初，两个Framework的主导份额是0％，我们假设DRF首先选择的是Framework 2，当然我们也可以假设Framework 1，但是最终的结果是一样的。

Framework 2接收份额并运行任务，使其主导资源成为CPU，主导份额增加至33％。
由于Framework 1的主导份额维持在0％，它接收共享并运行任务，主导份额的主导资源（内存）增加至22％。
由于Framework 1仍具有较低的主导份额，它接收下一个共享并运行任务，增加其主导份额至44％。
然后DRF将资源邀约发送给Framework 2，因为它现在拥有更低的主导份额。
该过程继续进行，直到由于缺乏可用资源，不能运行新的任务。在这种情况下，CPU资源已经饱和。
然后该过程将使用一组新的资源邀约重复进行。
需要注意的是，可以创建一个资源分配模块，使用加权的DRF使其偏向某个Framework或某组Framework。如前面所提到的，也可以创建一些自定义模块来提供组织特定的分配策略。

一般情况下，现在大多数的任务是短暂的，Mesos能够等待任务完成并重新分配资源。然而，集群上也可以跑长时间运行的任务，这些任务用于处理挂起作业或行为不当的Framework。

值得注意的是，在当资源释放的速度不够快的情况下，资源分配模块具有撤销任务的能力。Mesos尝试如此撤销任务：向执行器发送请求结束指定的任务，并给出一个宽限期让执行器清理该任务。如果执行器不响应请求，分配模块就结束该执行器及其上的所有任务。

分配策略可以实现为，通过提供与Framework相关的保证配置，来阻止对指定任务的撤销。如果Framework低于保证配置，Mesos将不能结束该Framework的任务。

我们还需了解更多关于Mesos资源分配的知识，但是我将戛然而止。接下来，我要说点不同的东西，是关于Mesos社区的。我相信这是一个值得考虑的重要话题，因为开源不仅包括技术，还包括社区。

说完社区，我将会写一些关于Mesos的安装和Framework的创建和使用的，逐步指导的教程。在一番实操教学的文章之后，我会回来做一些更深入的话题，比如Framework与Master是如何互动的，Mesos如何跨多个数据中心工作等。

与往常一样，我鼓励读者提供反馈，特别是关于如果我打标的地方，如果你发现哪里不对，请反馈给我。我非全知，虚心求教，所以非常期待读者的校正和启示。我们也可以在twitter上沟通，请关注 @hui_kenneth。




