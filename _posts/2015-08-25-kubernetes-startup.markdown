---
layout: post
title:  "启动 Kubernetes 集群"
date:   2015-08-25
categories: blog
---

这次咱们直接开始启动一个 Kubernetes 集群。只记录过程，Know Why 参考上一篇。

# 本机 VirtualBox 环境

| Machine | IP | Route | CPU | Mem | Network |
| --- | --- | --- | --- | --- | --- |
| Desktop | 192.168.56.1 | - | 4 core | 8 G | - |
| dev | 192.168.56.101 | 192.168.56.1 | 4 core | 1 G | host-only |
| node-1 | 192.168.56.102 | 192.168.56.1 | 2 core | 512 M | host-only |
| node-2 | 192.168.56.103 | 192.168.56.1 | 2 core | 512 M | host-only |

# 启动 docker


# 启动 kubelet


# 启动 _master_ components

## etcd

## apiserver 

## controller-manager 

## scheduler

# 注册 Node

