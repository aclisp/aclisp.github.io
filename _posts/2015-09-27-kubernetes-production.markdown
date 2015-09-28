---
layout: post
title:  "生产环境的 Kubernetes"
date:   2015-09-27
categories: blog
---

* TOC
{:toc}

# 每台主机都要安装

## 按照官方指引安装 docker，并修改启动参数
    
    --iptables=false 
    --ip-masq=false 
    --log-level=warn
    --bip=127.0.1.1/24

## 建立 k8s 配置文件和目录
    
    touch /etc/kubeconfig
    mkdir /etc/manifests

## `/etc/kubeconfig` 内容如下

{% highlight yaml %}
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: http://${MASTER_IP}:6443  # kube-apiserver
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: default
    user: myself
  name: default-context
current-context: default-context
kind: Config
preferences:
  colors: true
users:
- name: myself
  user:
    password: secret
    username: admin
{% endhighlight %}

## 安装 k8s 模块到 `/usr/bin`

### 注意这些需要事先确定的变量

* `MASTER_IP` MASTER 主机内网 IP
* `THIS_NODE_IP` 本主机内网 IP

### `kubelet` 启动参数

    --config=/etc/manifests
    --configure-cbr0=false
    --register-node=false
    --address=${THIS_NODE_IP}
    --hostname-override=${THIS_NODE_IP}
    --host-network-sources="file,api"
    --kubeconfig=/etc/kubeconfig
    --pod-infra-container-image="sigmas/pause:0.8.0"
    --api-servers="http://${MASTER_IP}:6443"

### `kube-proxy` 启动参数

    --kubeconfig=/etc/kubeconfig
    --bind-address=${THIS_NODE_IP}

# MASTER 主机安装

## 注意这些需要事先确定的变量

* `NODE_IP` 本主机内网 IP
* `VIP` 虚拟 IP

## etcd

下列内容存为文件 `/etc/manifests/etcd.yaml`

{% highlight yaml %}
apiVersion: v1
kind: Pod
metadata:
  name: etcd
spec:
  hostNetwork: true
  containers:
  - image: sigmas/etcd:2.0.12
    name: etcd
    command:
    - /usr/local/bin/etcd
    - --name=${NODE_IP}
    #- --initial-advertise-peer-urls=http://${NODE_IP}:2380
    #- --listen-peer-urls=http://${NODE_IP}:2380
    - --advertise-client-urls=http://${NODE_IP}:4001
    - --listen-client-urls=http://127.0.0.1:4001,http://${NODE_IP}:4001
    - --data-dir=/var/lib/etcd
    #- --discovery=${DISCOVERY_TOKEN}  # curl https://discovery.etcd.io/new?size=3
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

## kube-apiserver

下列内容存为文件 `/etc/manifests/kube-apiserver.yaml`

{% highlight yaml %}
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
spec:
  hostNetwork: true
  containers:
  - name: kube-apiserver
    image: sigmas/kube-apiserver:1.0.6
    command:
    - /kube-apiserver
    - --etcd-servers=http://127.0.0.1:4001
    - --service-cluster-ip-range=${VIP}/30
    - --bind-address=${NODE_IP}
    - --secure-port=0
    - --insecure-bind-address=${NODE_IP}
    - --insecure-port=6443
    ports:
    - containerPort: 6443
      hostPort: 6443
      name: http
{% endhighlight %}

## kube-controller-manager

下列内容存为文件 `/etc/manifests/kube-controller-manager.yaml`

{% highlight yaml %}
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
spec:
  hostNetwork: true
  containers:
  - name: kube-controller-manager
    image: sigmas/kube-controller-manager:1.0.6
    command:
    - /kube-controller-manager
    - --kubeconfig=/etc/kubeconfig
    volumeMounts:
    - mountPath: /etc/kubeconfig
      name: kubeconfig
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubeconfig
    name: kubeconfig
{% endhighlight %}

## kube-scheduler

下列内容存为文件 `/etc/manifests/kube-scheduler.yaml`

{% highlight yaml %}
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
spec:
  hostNetwork: true
  containers:
  - name: kube-scheduler
    image: sigmas/kube-scheduler:1.0.6
    command:
    - /kube-scheduler
    - --kubeconfig=/etc/kubeconfig
    volumeMounts:
    - mountPath: /etc/kubeconfig
      name: kubeconfig
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubeconfig
    name: kubeconfig
{% endhighlight %}
