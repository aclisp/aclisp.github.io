---
layout: post
title:  "安装 Kubernetes 二三事"
date:   2015-08-20
categories: blog
---

* TOC
{:toc}

本文探讨如何在 Ubuntu 14.04 上安装一套 Kubernetes。所有 Node 都用 Ubuntu 14.04。

> Kubernetes can run anywhere! However, initial development was done on GCE and so our instructions and scripts are built around that.

这就是为什么装一套 Kubernetes 那么难...

# Build from source

Kubernetes是在Docker容器里构建的。真是可重复构建（REPEATABLE BUILDS）被应用的典范。不用纠结于用什么版本的GCC，又缺了哪个版本的动态链接库等等龟毛问题。只要装了Docker就成。构建环境也推荐使用 Ubuntu 14.04。

构建过程需要至少3G内存，并会下载一个500M+的Image。需自备梯子。

	git clone git@github.com:kubernetes/kubernetes.git
	rsync -avz /media/sf_kubernetes .
	cd sf_kubernetes/
	cd build/
	./run.sh hack/build-go.sh
	
Build失败竟然会有Stack trace。怎么做到的？写bash也可以到这种程度吗？看来对牛人来说，语言真的不成问题。

由于 Golang 和 Docker Hub 在墙外，遇到超时就在Dockerfile里加上Proxy。**每个Dockerfile都要加！**

	ENV http_proxy http://192.168.56.1:1080
	ENV https_proxy http://192.168.56.1:1080
	ENV no_proxy 127.0.0.1
 
构建成功之后，在 `../_output/dockerized/bin/linux/amd64/` 下出现一堆可执行文件。

# Configuring kubectl

建议先安装控制端 `kubectl`。

	cd ..
	cd _output
	cp dockerized/bin/linux/amd64/kubectl /usr/local/bin/
	kubectl cluster-info

By default, kubectl configuration lives at `~/.kube/config`.

## 安全问题

RESTful调用容易被忽视的问题，就是如何进行认证。这里采用最简单的配置：

* 使用HTTP
* 基于用户名/密码认证
* kubelets, kube-proxy和kubectl用同一用户 

如下命令创建 `~/.kube/config` 配置文件

	kubectl config set preferences.colors true
	kubectl config set-credentials myself --username=admin --password=secret
	kubectl config set-cluster kubernetes --server=http://localhost:8080 --insecure-skip-tls-verify=true
	kubectl config set-context default-context --cluster=kubernetes --user=myself
	kubectl config use-context default-context
	kubectl config set contexts.default-context.namespace default
	kubectl config view

测试一下效果：

	kubectl cluster-info
	error: couldn't read version from server: Get http://localhost:8080/api: dial tcp 127.0.0.1:8080: connection refused

这就对了，因为 cluster 还没起呢。随便改改配置文件里的端口，发现是有效的。

# Configuring etcd

先起一个节点。HA问题以后再说。 1Core/1G 内存足够10个 Node 用。

	cp etcd-v2.1.1-linux-amd64/etcd /usr/local/bin/
	mkdir /var/lib/etcd
	etcd --data-dir /var/lib/etcd
	
**etcd推荐用Kubernetes指定的版本**，因为经过了充分的测试。

# 准备好这些可执行文件

| 文件名 | 运行在容器外 | 运行在容器内 | 推荐运行方式 |
| -------- | :-----: | :-----: | :-----: |
| docker | X | - | 系统服务 |
| kubelet | X | - | 系统服务 |
| kube-proxy | X | - | 系统服务 |
| etcd | - | X | Image+Pod | 
| kube-apiserver | - | X | Image+Pod |
| kube-controller-manager | - | X | Image+Pod |
| kube-scheduler | - | X | Image+Pod |

## 安装软件到 Node

每个 Node 要跑三个服务。注意每个 Node 不必是同等配置，可以有随意的 Core 和内存。

| 服务名 | 主要功能 | 如何保证高可用性 | 特别注意 |
| ----- | ------- | -------- | --------|
| docker | 本机容器管理 | systemd/upstart | 不用 `docker0`，不用 iptables |
| kubelet | Kubernetes管理 | systemd/upstart | 创建 `cbr0` 给 docker 用 |
| kube-proxy (可选) | 服务发现和负载均衡 | systemd/upstart | 接管 iptables |

### docker 

使用官方最新稳定版。但是需要为Kubernetes做特别配置。

* 清空 NAT

		iptables -t nat -F

* 停止并删除 docker0

		ifconfig docker0 down
		brctl delbr docker0

与用什么样的网络规划有关，需要设置下面这些 docker options，一般是在 `/etc/default/docker` 里改。

* `--bridge=cbr0` 由 kubelet 创建的 `cbr0`。
* `--iptables=false` iptables 将由 kube-proxy 接管。
* `--ip-masq=false` 
* `--mtu=` may be required when using Flannel, because of the extra packet size due to udp encapsulation.
* `--insecure-registry $CLUSTER_SUBNET` to connect to a private registry, if you set one up, without using SSL.
* `DOCKER_NOFILE=1000000`

### kubelet

**Kubernetes 核心组件，将会集成到 CoreOS 中！**

需要考虑的参数：

* `--config=/etc/kubernetes/manifests` Pod template 都放入这里。
* `--configure-cbr0=true` 创建 Node 时，让 kubelet 根据 `Node.Spec.PodCIDR` 配置 `cbr0`。kubelet 会等到 NodeController 设置了 `Node.Spec.PodCIDR` 之后才配置 `cbr0`. 
* `--register-node=false` 不采用自注册本机 Node，手工通过 apiserver 创建 Node。手工创建的 Node 由 NodeController 做 health checking，一旦失联则 Pod 不会被调度到其之上。

只有 `--register-node=true` 时，才要再考虑下列参数：

* `--kubeconfig=/var/lib/kubelet/kubeconfig` tells kubelet where to find credentials to authenticate itself to the apiserver. (笔者注：就是用 `kubectl config` 创建的文件吧？)
* `--api-servers=http://$MASTER_IP` tells the kubelet the location of the apiserver.
* `--cloud-provider=` tells the kubelet how to talk to a cloud provider to read metadata about itself.

### kube-proxy

需要考虑的参数：

* `--kubeconfig=/var/lib/kube-proxy/kubeconfig`
* `--api-servers=http://$MASTER_IP`

## Node 的网络规划 

Kubernetes 必须用（某种意义上的）扁平网络，但是我们的托管主机只有一个出口，主机之间没有公共的 router/switch。这就需要做虚拟交换。简单先用着 [Flannel](https://github.com/coreos/flannel)。

| Scope   | 解释                                            | 最大数量 |
| :-----: | ----------------------------------------------- | :---: |
| Cluster | 每个`10.x`(x=0-255)都是一个cluster，我们只用`10.10`    | 256 |
| Node    | `10.10.0.0/24` 至 `10.10.255.0/24` 每个都是一个node   | 256 |
| Pod     | `10.10.x.2/32` 至 `10.10.x.254/32` 位于第 x 个node   | 253 | 
| cbr0    | 一般 `Node.Spec.PodCIDR` 的第一个 IP 给 bridge        | - |

例如：

* 通过 apiserver 创建第一个 Node，对应的 `Node.Spec.PodCIDR` 则为 `10.10.0.0/24`
* 通过 apiserver 创建第二个 Node，对应的 `Node.Spec.PodCIDR` 则为 `10.10.1.0/24`
* 以此类推……

剩下的问题是 Node_1 与 Node_2 之间如何互通。考虑这样一种状况：

* `Node_1_cbr0` 获得 IP `10.10.0.1/24`
* `Node_1_Pod_1` 获得 IP `10.10.0.2/24`
* `Node_2_cbr0` 获得 IP `10.10.1.1/24`
* `Node_2_Pod_1` 获得 IP `10.10.1.2/24`

`Node_1_Pod_1` 发包到 `Node_2_Pod_1` 经历这样一个过程：

1. `Node_1_Pod_1` -> `Node_1_cbr0` -> `Node_1_flannel0` -> `Node_1_flanneld` -> `Node_1_eth0`
2. `Node_1_eth0` -> Internet -> `Node_2_eth0` (UDP TUNNEL)
3. `Node_2_Pod_1` <- `Node_2_cbr0` <- `Node_2_flannel0` <- `Node_2_flanneld` <- `Node_2_eth0`

请自行脑补L2/L3/ARP/IP等底层协议栈的详细过程。

## Node 的调度算法

* kube-scheduler 尝试在“最佳” Node 上创建 Pod；如果一个 Node 都找不到，则等待直到有这样的 Node。
* “最佳” Node 如何定义呢？
	1. Node 和 Pod 的 Label 要匹配
	2. 已用资源与请求资源之和不大于 Node 容量
	3. 如果多个 Node 满足前二，进入优先级考查
* 优先级如何考查呢？scheduler 对满足条件的所有 Node 做分级：
	- 已用资源最少的 Node 最优先

调度算法已经被设计成 plugin，不爽可以修改定制。

## Node 其他

* 如果需要，配置Node的OS自动升级。
* 防止服务日志把磁盘写爆 (如利用 logrotate).
* Setup liveness-monitoring (e.g. using monit).

# Cluster 启动

写到这里才发现，

* etcd
* kube-apiserver
* kube-controller-manager
* kube-scheduler

这几个用 Kubernetes 自己进行配置和管理最方便。

* their options are specified in a Pod spec (yaml or json) rather than an /etc/init.d file or systemd unit.
* they are kept running by Kubernetes rather than by init.

因此，要先把它们变成 Image。

等等，这里有个“先有鸡还是先有蛋”的问题：Kubernetes 调度 apiserver 需要知道 Node，而注册 Node 又需要 apiserver 可达。尼玛……还是直接启动好了。

---

参考：

1. [Getting started from Scratch](https://github.com/kubernetes/kubernetes/blob/master/docs/getting-started-guides/scratch.md)
2. [Docker Networking](https://docs.docker.com/articles/networking/)
3. [Kubernetes Networking](https://github.com/kubernetes/kubernetes/blob/master/docs/admin/networking.md)
4. [Four ways to connect a docker container to a local network](http://blog.oddbit.com/2014/08/11/four-ways-to-connect-a-docker/)
5. [How to understand Linux bridge](http://unix.stackexchange.com/questions/191174/how-to-understand-virtual-switch-in-linux)
6. [How does Kubernetes' scheduler work](http://stackoverflow.com/questions/28857993/how-does-kubernetes-scheduler-work)
