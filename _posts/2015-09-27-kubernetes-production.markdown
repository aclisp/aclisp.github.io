---
layout: post
title:  "生产环境的 Kubernetes"
date:   2015-09-27
categories: blog
---

* TOC
{:toc}

# 每台主机都要安装

*   文件和目录
    - /etc/kubeconfig 是配置文件
    - /etc/manifests 是容器目录

*   配置文件
    - API server 地址：http://$MASTER_IP:6443
    - 缺省 namespace：default
    
*   kubelet
    - --config=/etc/manifests
    - --configure-cbr0=false
    - --register-node=false
    - --address=THIS_NODE_IP
    - --hostname-override=THIS_NODE_IP
    - --host-network-sources="file,api"
    - --kubeconfig=/etc/kubeconfig
    - --pod-infra-container-image="sigmas/pause"

*   kube-proxy
    - --kubeconfig=/etc/kubeconfig
    - --hostname-override=THIS_NODE_IP
    - --bind-address=THIS_NODE_IP

# 3 个 MASTER NODE 安装

*   etcd

{% highlight yaml %}
apiVersion: v1
kind: Pod
metadata:
  name: etcd
spec:
  hostNetwork: true
  containers:
  - image: sigmas/etcd
    name: etcd
    command:
    - /usr/local/bin/etcd
    - --name=${NODE_IP}
    - --initial-advertise-peer-urls=http://${NODE_IP}:2380
    - --listen-peer-urls=http://${NODE_IP}:2380
    - --advertise-client-urls=http://${NODE_IP}:4001
    - --listen-client-urls=http://127.0.0.1:4001,http://${NODE_IP}:4001
    - --data-dir=/var/lib/etcd
    - --discovery=${DISCOVERY_TOKEN}  # curl https://discovery.etcd.io/new?size=3
    ports:
    - containerPort: 2380
      hostPort: 2380
      name: serverport
    - containerPort: 4001
      hostPort: 4001
      name: clientport
    volumeMounts:
    - mountPath: /var/lib/etcd
      name: varetcd
  volumes:
  - hostPath:
      path: /var/lib/etcd
    name: varetcd
{% endhighlight %}

*   kube-apiserver

{% highlight yaml %}
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
spec:
  hostNetwork: true
  containers:
  - name: kube-apiserver
    image: sigmas/kube-apiserver
    command:
    - /usr/local/bin/kube-apiserver
    - --etcd-servers=http://127.0.0.1:4001
    - --service-cluster-ip-range=${VIP}/32
    - --bind-address=${NODE_IP}
    - --secure-port=0
    - --insecure-bind-address==${NODE_IP}
    - --insecure-port=6443
    ports:
    - containerPort: 6443
      hostPort: 6443
      name: http
{% endhighlight %}

*   apiserver-loadbalancer

需要 haproxy + keepalived 

*   kube-controller-manager

{% highlight yaml %}
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
spec:
  hostNetwork: true
  containers:
  - name: kube-controller-manager
    image: sigmas/kube-controller-manager
    command:
    - /usr/local/bin/kube-controller-manager
    - --kubeconfig=/etc/kubeconfig
{% endhighlight %}

*   kube-scheduler

{% highlight yaml %}
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
spec:
  hostNetwork: true
  containers:
  - name: kube-scheduler
    image: sigmas/kube-scheduler
    command:
    - /usr/local/bin/kube-scheduler
    - --kubeconfig=/etc/kubeconfig
{% endhighlight %}

*   podmaster

等 apiserver-loadbalancer 
