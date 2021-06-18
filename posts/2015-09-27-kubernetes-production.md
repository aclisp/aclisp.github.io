# 生产环境的 Kubernetes

按照下面的步骤在一个机房的所有主机上依次安装。

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

```yaml
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
```

## 安装 k8s 模块到 `/usr/bin`

1. 把编译出的可执行文件 `kubelet` 和 `kube-proxy` 放入 `/usr/bin` 目录下；
1. 按照操作系统指引建立开机启动的 daemon 进程；
1. 设置好启动参数；
1. 设置进程监控 [monit](http://mmonit.com/monit/)

### 注意这些需要事先确定的变量

* `MASTER_IP` MASTER 主机内网 IP
* `THIS_NODE_IP` 本主机内网 IP

### `kubelet` 启动参数

    --config=/etc/manifests
    --configure-cbr0=false
    --register-node=false
    --address=${THIS_NODE_IP}
    --hostname-override=${THIS_NODE_IP}
    --host-network-sources=file,api
    --kubeconfig=/etc/kubeconfig
    --pod-infra-container-image=sigmas/pause:0.8.0
    --api-servers=http://${MASTER_IP}:6443

### `kube-proxy` 启动参数

    --kubeconfig=/etc/kubeconfig
    --bind-address=${THIS_NODE_IP}

所有主机完成以上步骤之后，开始下面的

# MASTER 主机安装

## 注意这些需要事先确定的变量

* `NODE_IP` 本主机内网 IP
* `VIP` 事先分配好的虚 IP

## etcd

下列内容存为文件 `/etc/manifests/etcd.yaml`

```yaml
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
```

**注意：**

集群的状态数据保存在 etcd 中。为了防止硬盘挂掉，需要配置多个 etcd 实例（每个 MASTER 主机一个）。例如，配置 3 个实例则取消注释掉的三行。

以下的 kube-* 模块目前只部署在一台 MASTER 主机上。

## kube-apiserver

下列内容存为文件 `/etc/manifests/kube-apiserver.yaml`

```yaml
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
```

## kube-controller-manager

下列内容存为文件 `/etc/manifests/kube-controller-manager.yaml`

```yaml
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
```

## kube-scheduler

下列内容存为文件 `/etc/manifests/kube-scheduler.yaml`

```yaml
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
```

## 配置服务

### 对内的 etcd 服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd
spec:
  ports:
  - port: 4001
    protocol: TCP
```

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: etcd
subsets:
- addresses:
  - ip: ${MASTER_IP}
  ports:
  - port: 4001
    protocol: TCP
```

# 存活监控

*   kube-proxy

        curl http://127.0.0.1:10249/healthz

*   kubelet

        curl http://127.0.0.1:10248/healthz

*   其它

        kubectl get cs

