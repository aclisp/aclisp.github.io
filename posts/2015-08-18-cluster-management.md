# 集群管理

# 什么是集群管理系统

[集群管理系统](https://en.wikipedia.org/wiki/Cluster_manager)是现在后端的热门话题。前有[京东618：Docker扛大旗，弹性伸缩成重点](http://www.infoq.com/cn/news/2015/06/jd-618-docker/)，后有[Voidbox – Docker on YARN](http://tech.hulu.com/blog/2015/08/06/voidbox-docker-on-yarn/)。其实，现在各大互联网公司都有自己的集群管理系统，如腾讯Gaia，百度Matrix，Google Borg/Omega；没有的也正在研发的路上。

那么，什么是集群管理系统呢？

集群管理系统也叫集群操作系统（Cluster Operating System）。OS的出现让应用（Process）同时共享一台计算机，而Cluster OS的出现让应用（如Hadoop Jobs）同时共享一个集群。

> Good ideas today mirror good ideas of yesteryear.

可以这样比较两者的一些概念：

|          | 单机操作系统 Linux | 集群操作系统 Mesos|
| -------- | :-----: | :-----: |
| 隔离机制 | 进程 Process 提供地址空间和文件描述符的隔离 | 容器 LXC 提供CPU, Memory, FileSystem等的隔离 |
| 共享机制 | FileSystem, IPC | Distributed-FS, Network |
| 调度机制 | Process Scheduler 保证多个进程共享几个CPU | 同样有机制保证多个集群应用的每个只占用一小块数据中心资源 |
| 抽象机制 | read(), write(), open(), bind(), connect() 这些系统调用抽象了不同的硬件和网络 | launchTask(), killTask(), statusUpdate() 这些API抽象了部署位置和远程通信，不再直接跟socket和pid打交道 |
| 包管理机制 | apt-get, yum | Docker |

# 为什么要有集群管理系统

* 资源共享，降低成本，像使用一台服务器一样，使用整个集群的资源。
* 动态伸缩，根据业务压力按需分配资源。
* 容灾容错，应用自动重启，自动迁移。
* 更加便捷、更加可靠的升级、回滚方式。

# 如何设计实现一个集群管理系统

我司正在研发的路上，摆在面前的有两种选择：

* 从零开始，实现包括节点管理、任务调度等一系列功能。可参考的资料有 [Brog](http://research.google.com/pubs/pub43438.html)。非常的有挑战性。
* 借助开源，让专业人士来解决复杂的调度问题。集中精力做好跟业务对接的用户体验。

> Building custom framework and leveraging open source.

## 调度策略的难题

就像Linux Kernel书首先会讲进程调度，集群管理系统首先要研究的也是调度策略。

业务一般有两大类：服务（long-running services, 如 [RDS](https://aws.amazon.com/rds/mysql/)）和任务（batch jobs, 如 Hadoop）。它们有不同的特征，前者一般是[IO密集型](http://en.wikipedia.org/wiki/I/O_bound)，后者一般是[CPU密集型](http://en.wikipedia.org/wiki/CPU-bound)。对计算节点的性能容量要求各不相同。因此，对调度算法的要求也会不同。

我们所谈到的最大好处，共享、伸缩、迁移，要解决的难题其实都是一个：在哪个节点上共享，往哪里扩展，往哪儿迁移。如果做不到，根本就谈不上集群管理，充其量也就是远程运行容器。

## 有哪些调度策略

*   Monolithic
    所有的任务用同一种算法调度。YARN，Borg和Kubernetes都用这种。

*   Statically partitioned
    固定分配，专机专用。基本谈不上调度了。例如Hadoop早期版本（[YARN](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html)出现之前）。

*   Two-level ([Mesos](http://mesos.apache.org/documentation/latest/mesos-architecture/))
	中心管理器收集可用资源作为Offer提供给用户，再由用户自定义任务的分配策略。

*   Shared-state ([Omega](http://research.google.com/pubs/pub41684.html))
    竞争式乐观锁分配。只是Omega的设想，好不好用谁也不知道。

## 为什么要用 Docker

通过深度定制 OpenStack，其实基于 VM 的分配策略已经很完善了。想用 Docker 是因为 VM 的 OS 虚拟层损耗让共享分配得不偿失。

## 除了调度策略，还有哪些坑

*   搭建 Docker Registry
*   构建业务的 Docker Image
*   规划 Docker 的网络和存储
*   服务发现和负载均衡
*   业务接入、扩容、缩容、升级、下线的工作流
*   跟 OpenStack 集成
*   支持大规模集群
*   性能和时延是否满足业务要求





---

参考：

1.  [Mesos: The Operating System for your Cluster](https://www.youtube.com/watch?v=gVGZHzRjvo0)
2.  [Borg: The Predecessor to Kubernetes](http://blog.kubernetes.io/2015/04/borg-predecessor-to-kubernetes.html)
