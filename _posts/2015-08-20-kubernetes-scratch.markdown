---
layout: post
title:  "安装 Kubernetes 二三事"
date:   2015-08-20
categories: blog
---

本文探讨如何在 Ubuntu 14.04 上安装一套 Kubernetes。所有 Node 都用 Ubuntu 14.04。

> Kubernetes can run anywhere! However, initial development was done on GCE and so our instructions and scripts are built around that.

这就是为什么装一套 Kubernetes 那么难...

# Build from source

Kubernetes是在Docker容器里构建的。真是可重复构建 (REPEATABLE BUILDS）被应用的典范。不用纠结于用什么版本的GCC，又缺了哪个版本的动态链接库等等龟毛的问题。只要装了Docker就成。构建环境也推荐使用 Ubuntu 14.04。

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
	kubectl config set-cluster local-server --server=http://localhost:8080 --insecure-skip-tls-verify=true
	kubectl config set-context default-context --cluster=local-server --user=myself
	kubectl config use-context default-context
	kubectl config set contexts.default-context.namespace the-right-prefix
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

| 文件名 | 运行在容器外 | 运行在容器内 | Kubernetes推荐 |
| -------- | :-----: | :-----: | :-----: |
| docker | X | - | Run as system daemon on every node |
| kubelet | X | - | Run as system daemon on every node |
| kube-proxy | X | - | Run as system daemon on every node |
| etcd | ? | X | Use gcr.io/google_containers/hyperkube:$TAG | 
| kube-apiserver | ? | X | Use gcr.io/google_containers/hyperkube:$TAG |
| kube-controller-manager | ? | X | Use gcr.io/google_containers/hyperkube:$TAG |
| kube-scheduler | ? | X | Use gcr.io/google_containers/hyperkube:$TAG |

## 安装软件到 Node

每个 Node 要跑三个服务。注意每个 Node 不必是同等配置，可以有随意的 Core 和内存。

* docker
* kubelet
* kube-proxy

### docker 

使用官方最新稳定版。但是需要为Kubernetes做特别配置。

* 清空 NAT

		iptables -t nat -F

* 停止并删除 docker0

		ifconfig docker0 down
		brctl delbr docker0

与用什么样的网络规划有关，需要设置下面这些默认的 docker options，一般是在 `/etc/default/docker` 里改。

* create your own bridge for the per-node CIDR ranges, call it `cbr0`, and set `--bridge=cbr0` option on docker.
* set `--iptables=false` so docker will not manipulate iptables for host-ports (too coarse on older docker versions, may be fixed in newer versions) so that kube-proxy can manage iptables instead of docker.
* `--ip-masq=false` ??? TODO
* `--mtu=` may be required when using Flannel, because of the extra packet size due to udp encapsulation
* `--insecure-registry $CLUSTER_SUBNET` to connect to a private registry, if you set one up, without using SSL.
* `DOCKER_NOFILE=1000000`

阅读材料 [Docker Networking](https://docs.docker.com/articles/networking/) 一定要细读。

### kubelet

需要考虑的参数：

* `--kubeconfig=/var/lib/kubelet/kubeconfig`
* `--api-servers=http://$MASTER_IP`
* `--config=/etc/kubernetes/manifests` ???
* `--configure-cbr0=` (described above)
* `--register-node` (参见 [Kubernetes Node 扫盲](TODO))
	
### kube-proxy

需要考虑的参数：

* `--kubeconfig=/var/lib/kube-proxy/kubeconfig`
* `--api-servers=http://$MASTER_IP`


## Node 的网络规划 

TODO：非常有技术含量的问题，会另起一篇再讲。

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

累了，下次再写。

---

参考：

1. [Getting started from Scratch](https://github.com/kubernetes/kubernetes/blob/master/docs/getting-started-guides/scratch.md)
2. [Docker Networking](https://docs.docker.com/articles/networking/)
3. [Kubernetes Networking](https://github.com/kubernetes/kubernetes/blob/master/docs/admin/networking.md)
4. [Four ways to connect a docker container to a local network](http://blog.oddbit.com/2014/08/11/four-ways-to-connect-a-docker/)