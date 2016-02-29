---
layout: post
title:  "Kubernetes 网络设计 (2)"
date:   2016-02-29
categories: blog
---

# 如何搭建覆盖网络

搭建用于容器云的跨主机覆盖网络，其实与 docker 无关。

这时，选取 flannel 这种一人项目，能完全掌控。

启动参数就用 

    -etcd-endpoints="http://etcd1:4001,http://etcd2:4001,http://etcd3:4001"

本机自动被分配到一个 `/26` 子网，而整个大网由 `Network` 指定：

    {
        "Network": "192.168.0.0/16",
        "SubnetLen": 26,
        "SubnetMin": "192.168.4.0",
        "SubnetMax": "192.168.254.192",
        "Backend": {
            "Type": "udp"
        }
    }

# 覆盖网络对业务容器的作用

免除了端口映射的麻烦！

## 场景一：业务有自己的服务注册、服务发现的机制。

容器化以后，业务在不需要修改代码的情况下，就能把容器 IP:Port 上报，用于服务发现。

## 场景二：业务需要作数据同步。

容器化以后，把容器 IP:Port 在需同步的实例之间互相告知。


