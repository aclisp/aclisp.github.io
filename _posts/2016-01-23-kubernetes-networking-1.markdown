---
layout: post
title:  "Kubernetes 网络设计 (1)"
date:   2016-01-23
categories: blog
---

使用 `--net=host` 来跑应用的限制很多，特别是弹性伸缩，故障恢复都不好做。一个很关键的问题就是主机端口已经被占用了怎么办？那就没法调度到这台主机了。另一个问题，来自于用户体验不好：用户必须指定应用的监听端口；用户与用户之间需要协商端口分配。还有一个问题是，容器的隔离性不好，容器内部的服务不能监听任意端口。

最初的想法，要有一个全局的负责端口分配和管理的模块。但是，觉得仍然不好用！

由于不能做网络改造（该死的运维！），给每个容器分配一个 IP 不可行。于是想到了端口映射。这跟 [Mesos - Marathon](https://mesosphere.github.io/marathon/docs/application-basics.html) 的做法非常相似了。

每个应用（即服务）暴露多个端口映射：

* `name` 端口映射的名称，可以被 `Service` 的 `targetPort` 引用。
* `containerPort` 容器内的监听端口，应用可以随意指定。
* `hostPort` 一般不写，即为 `0`。让系统分配一个。而且只要容器不迁移，就不变！

同时，使用 [Headless services](http://kubernetes.io/v1.1/docs/user-guide/services.html#headless-services) 并指定 `type` 为 `NodePort` 让系统分配一个。类似于 Marathon 的 `servicePort`。

遵循这种设计，应用部署之后，定时用 API 查 `Endpoints`，把结果更新到 `haproxy` 的配置文件。跟 Marathon 的[服务发现](https://mesosphere.github.io/marathon/docs/service-discovery-load-balancing.html)一样。

Kubernetes 的代码修改：

1. `pkg/controller/endpoint/endpoints_controller.go` 修改 `findPort`，如果是 Headless service 则装配映射后的端口！
1. `hostPort` 的随机分配和持久化！
1. `PortMapping` 反映到 `pod.Status` 里！

