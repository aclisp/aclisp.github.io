---
layout: post
title:  "启动 Kubernetes 集群"
date:   2015-08-25
categories: blog
---

* TOC
{:toc}

这次咱们直接开始启动一个 Kubernetes 集群。只记录过程，Know Why 参考上一篇。

# 本机 VirtualBox 环境

| Machine   | IP             | #Core | Mem (G) | Network | Role  |
| --------- | -------------- | :---: | :---: | :-------: | :---: |
| Desktop   | 192.168.56.1   | 4     | 8     | - | - |
| VM dev    | 192.168.56.101 | 4     | 1     | host-only | master, minion |
| VM node-1 | 192.168.56.102 | 2     | 0.5   | host-only | minion |
| VM node-2 | 192.168.56.103 | 2     | 0.5   | host-only | minion |

dev, node-1 和 node-2 作为 Kubernetes 集群的三个 Node，其 IP `192.168.56.0/24` 是本机唯一出口。建立 Kubernetes 网络规划如下：

| Hostname | PodCIDR      | 备注 |
| -------- | ------------ | --- |
| dev      | 10.10.x.0/24 | 由 flanneld 分配 |
| node-1   | 10.10.y.0/24 | 由 flanneld 分配 |
| node-2   | 10.10.z.0/24 | 由 flanneld 分配 |

# Boot _master_ Node

**以下步骤必须按顺序执行！** 所有进程都起在前台，以后再进行服务化。

    MASTER_IP=192.168.56.101

## 启动 etcd

    mkdir -p /var/lib/etcd
    etcd --data-dir=/var/lib/etcd --listen-client-urls=http://$MASTER_IP:4001 --advertise-client-urls=http://$MASTER_IP:4001

## 停止 docker

    service docker stop

    brctl show
    ip link set dev docker0 down
    brctl delbr docker0

    iptables -t nat -n -L
    iptables -t nat -F

Ubuntu 14.04

    echo manual > /etc/init/docker.override
    reboot

## 启动 flanneld

Push configuration JSON to etcd.

* `10.10.1.0/24` 预留给 service_cluster_ip_range
* `10.10.10.0/24` 至 `10.10.255.0/24` 预留给 Node

{% highlight json %}
{
    "Network": "10.10.0.0/16",
    "SubnetLen": 24,
    "SubnetMin": "10.10.10.0",
    "SubnetMax": "10.10.255.0",
    "Backend": {
        "Type": "udp",
        "Port": 7890
    }
}
{% endhighlight %}

    etcdctl --peers=$MASTER_IP:4001 set /coreos.com/network/config "$(<flanneld.json)"
    etcdctl --peers=$MASTER_IP:4001 ls --recursive -p

flanneld 配置如下：

* `--iface=eth0`
* `--etcd-endpoints=http://$MASTER_IP:4001`

`/run/flannel/subnet.env` 里有 `FLANNEL_SUBNET` 和 `FLANNEL_MTU`

## 创建 cbr0

	ip link set dev cbr0 down
	brctl delbr cbr0

    brctl addbr cbr0
    ip addr add $FLANNEL_SUBNET dev cbr0
    ip link set dev cbr0 up

    ip addr show cbr0

## 启动 docker

docker daemon 配置如下：

* `--bridge=cbr0`
* `--iptables=false`
* `--ip-masq=false`
* `--mtu=$FLANNEL_MTU`

## 启动 kubelet

kubelet 配置如下：

* `--config=/etc/kubernetes/manifests`
* `--configure-cbr0=false`
* `--register-node=false`
* `--api-servers=$MASTER_IP:8080`
* `--address=0.0.0.0`
* `--hostname-override=$FLANNEL_SUBNET_IP`

## 启动 apiserver controller-manager scheduler

### apiserver

* `--etcd-servers=http://$MASTER_IP:4001`
* `--service-cluster-ip-range=10.10.1.0/24`
* `--bind-address=$MASTER_IP`
* `--insecure-bind-address=0.0.0.0`

### controller-manager

* `--kubeconfig=/root/.kube/config`

### scheduler

* `--kubeconfig=/root/.kube/config`

## 注册 Node

    kubectl create -f - <<NODE_JSON
    {
      "kind": "Node",
      "apiVersion": "v1",
      "metadata": {
        "name": "$FLANNEL_SUBNET_IP"
      },
      "spec": {
        "podCIDR": "$FLANNEL_SUBNET"
      }
    }
    NODE_JSON

# Boot _minion_ Node

除了不用启动 etcd 和 master components，与 master node 相同。

## 停止 docker
## 启动 flanneld
## 创建 cbr0
## 启动 docker
## 启动 kubelet
## 注册 Node

# 重新回顾 Kubernetes Networking

我的这套设置无疑是满足基本需求的：

* All containers can communicate with all other containers without NAT
* All nodes can communicate with all containers (and vice-versa) without NAT
* The IP that a container sees itself as is the same IP that others see it as

细说就是：

* Nodes IP 都是 `10.10.x.1/24` 即 `cbr0` 的 IP，`x` 由 flanneld 分配
* Containers IP 都从 `10.10.x.2/24` 开始
* Cluster Subnet 是 `10.10.0.0/16`
* Containers 想上外网，必须加上 NAT 规则

For example:

    iptables -w -t nat -A POSTROUTING -o eth0 -j MASQUERADE \! -d ${CLUSTER_SUBNET}

This will rewrite the source address from the Container IP to the Node IP for traffic bound outside the cluster (aka SNAT - to make it seem as if packets came from the Node itself), and kernel connection tracking will ensure that responses destined to the node still reach the pod.


