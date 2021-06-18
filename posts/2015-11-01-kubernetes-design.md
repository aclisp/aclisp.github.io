# Kubernetes 设计思想

这篇笔记是我对协议、网络、分布式设计理解集大成者。也是持续思考、持续改进的最佳实践。

* TOC
{:toc}

# 背景

基础平台架构里，会有这样的数据中心

![What a datacenter looks like](/assets/datacenter.jpg)

考虑到地震这种小概率事件，一个数据中心不够。下面是一个跨洋数据中心示意图，所谓的 Geographic Redundancy。

![Geographic redundancy datacenters](/assets/geored_datacenter_1.png)

注意，专线（Private Connection）不一定有。因为我厂是慢慢起家的，最初仅仅租用几个数据中心里的数十台服务器，后来自购服务器托管到数据中心里，直到现在也没有自建数据中心。因此，专线就是锦上添花的事情。

没有专线的问题是，节点（Hardware Nodes）之间交互要通过上游路由器（Uplink Routers）去往网络运营商线路，

1. 没有上游路由器的控制权，基本无法改动网络拓扑结构
1. 跨网络运营商线路的交互，在时延、抖动等网络质量指标上无法保证

我的设计就是在这样一个受限的条件下开展。

![Limited geographic redundancy datacenters](/assets/geored_datacenter_2.png)

# 网络配置

## Docker 默认网络

Docker 装上之后，默认的网络是与外界隔离的。

![Default docker networks](/assets/docker_isolated_network.png)

以上是一个经典的网络示意图，几个细节很少有人注意到

1. 容器内的 `eth0` 获得 `docker0` 子网 `172.17.42.1/16` 下的任一 IP
1. `docker0` 并没有挂上主机的 `eth0` 或者 `eth1`

上面两点怎么理解？简单的，把 `Virtual Ethernet Bridge` 类比你家的宽带路由器。

每启动一个容器，就相当于你家的一个设备，比如手机或者电脑，连上去了（无论是通过 Wi-Fi 还是网线）。你的设备当然会获得一个 IP 地址。这台宽带路由器的 WAN 口还没有接上入户线，所以还不能上网。

别忘了我们有 iptables。Docker 采用了 IP Masquerade 的技术使容器能够访问外网。外网访问容器则使用 NAT。

![IP Masquerade ](/assets/ip_masq.gif)

## L2 网络

这种方式，是最自然的跨主机容器通信的方案。从网络的角度，容器就是一台独立的机器。

![Docker networks](/assets/docker_network_1.png)

优点：

1. 不需要引入 SDN 使整个系统的稳定和性能都有保证

缺点：

1. 主机网络的 DHCP 无法为容器分配 IP
1. 必须为每个主机保留一段 IP，造成 IP 地址资源浪费
1. 主机网络是一张 L2 大网。也即需要有跨数据中心专线

## SDN 网络

这里的缺点就是上面的优点。

![Docker SDN networks](/assets/docker_network_2.png)

## 使用 `--net=host` 启动容器

放弃使用容器的网络隔离特性。并且需要注意同一主机上容器的端口冲突。

# 设计思想

在设计系统时，我们会假定部署环境满足一定条件的时延、带宽和可用性。这也意味着，所有主机必须在一个数据中心，或者地域接近。

把物理位置接近的数据中心组织起来，形成一个地域（Zone）是一个好办法。

![Zones](/assets/geored_zones.jpg)

但一个地域内，也是有多个数据中心的。这种跨数据中心交互，由软件设计来保证可靠性。

## Overview

逻辑结构图

![Logical diagram](/assets/k8s_architecture.png)

物理部署图

![Deploy diagram](/assets/k8s_deploy.png)

## Kubernetes 与 Borg

> Borg 是 Google 的内部容器管理系统。早在十几年前，Google 就已经部署 Borg 系统对来自于几千个应用程序所提交的 job 进行接收、调试、启动、停止、重启和监控，实现资源管理的自动化以及跨多个数据中心的资源利用率最大化。Kubernetes 项目的创始人 Brendan Burns 曾表示，Kubernetes 项目的目的就是将 Borg 最精华的部分提取出来，使现在的开发者能够更简单、直接地应用。Kubernetes 以 Borg 为灵感，但又没那么复杂和功能全面，更强调了模块性和可理解性。

> Kubernetes 与 Borg 最主要不同就是 API 。Borg 的高层是描述性的，但是在 Borg 真正实现的组件之间，实际上是命令性的 API。而我们在最初设计 Kubernetes 时就坚持使用描述性的 API。

> 因为使用了描述性 API，所以 Kubernetes 的内部实现不需要有非常复杂的状态机，我们使用了一个比较简单的调和控制回路。所谓「描述性」就是形容你想要的是什么状态，最终要的是什么结果，然后你的调和控制回路（也就是控制系统）知道了这个目标，它会根据现有状况进行调整，一直驱动达到理想的状态。比如你在调度的时候，需要考虑有多少个 job 在运行，job 是在什么情况下运行，有多少个 copy 在运行，少了一个 copy 你就多加一个，多了一个你就杀死一个。这两点就是主要贯彻在整个 Kubernetes 设计中的原则。

## 组件关系

为了背书这种设计思想，一个简化后的组件关系图如下：

![Simplified diagram](/assets/k8s_simplified.png)

* __API Server 与 Storage__ 这两者共同维护一个由 `runtime.Object` 组成的数据结构
* __Agent__ 注册节点，上报事件，同步状态
* __Controller__ 处理上报的事件
* __Scheduler__ 只负责调度算法

这是一种全新的思路，没有模块之间复杂的交互时序图。每个模块只做好自己的事：

* Agent 不断 Watch 是否有新状态分配给自己；
* Controller 不断 Watch 集群状态，作相应处理并更新状态；
* Scheduler 不断 Watch 新容器状态，调度完毕后分配给相应节点并更新状态。

得益于 Etcd 提供的 Watch API，一切井然有序的工作着；不像命令性的 API，需要考虑失败重传，成功确认。

## 使用 `--net=host` 要注意的地方

各种配置不能直接拿来用，需要小心规划端口。

---

参考：

1. [Etcd API](https://github.com/coreos/etcd/blob/master/Documentation/api.md)
1. [Considerations for running multiple clusters](http://kubernetes.io/v1.0/docs/admin/multi-cluster.html)
1. [使用 Kubernetes 1.0 构建 CaaS](http://mp.weixin.qq.com/s?__biz=MzA4OTMxODQwNA==&mid=400232239&idx=1&sn=754226d2ba259b8d47367bc9addbd78f&scene=2&srcid=1025Ho1IJwPEQHEmUqTio7jU&from=timeline&isappinstalled=0&key=b410d3164f5f798e04d76c1a92e630430852bf90e50038756969bb5393e7862900907db3f8c63d0fc62f3a8be36dacb5&ascene=1&uin=MjY5ODI0MjgyMg%3D%3D&devicetype=webwx&version=70000001&pass_ticket=wx9WwacUJKh5Jycgnr1y6BZVVXlJHFMby1FKW2aAzwXEDqjgJgrCvB%2FdK8Z6gMy1)
1. [Google云平台负责人：开源是唯一的路](http://www.leiphone.com/news/201509/THxjnrCWZMkCAniC.html)


