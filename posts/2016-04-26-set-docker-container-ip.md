# 固定 docker 容器的 IP"

# 起因

我们利用 flannel 搭建了覆盖网络。这样，每个容器获得一个独特的容器 IP。

容器通过这个 IP 提供对外服务。等等……这个 IP 是由 docker 分配的，主机重启或者 Pod Infra 重启就变了！

Kubernetes 的推荐做法是创建 [“服务”](http://kubernetes.io/docs/user-guide/services/) 对象。但我觉得这只适用于负载均衡的场景，
而不适用于 [ZooKeeper 集群](https://github.com/kubernetes/kubernetes/issues/260)。

我提倡用一种更简单的办法，无必要引入“服务” 对象来增加困扰。

# 方式

Docker 1.10 的 [ChangeLog](https://github.com/docker/docker/blob/master/CHANGELOG.md) 里有一个更新可以利用

> Add `--ip` and `--ip6` on `run` and `network connect` to support custom IP addresses for a container in a network
> [#19001](https://github.com/docker/docker/pull/19001)

# 代码跟踪

## Docker 1.7.1 的容器 IP 分配

[libnetwork/network.go](https://github.com/docker/docker/blob/v1.7.1/vendor/src/github.com/docker/libnetwork/network.go)

```golang
func (n *network) CreateEndpoint(name string, options ...EndpointOption) (Endpoint, error) {
    if name == "" {
        return nil, ErrInvalidName(name)
    }
    ep := &endpoint{name: name, iFaces: []*endpointInterface{}, generic: make(map[string]interface{})}
    ep.id = types.UUID(stringid.GenerateRandomID())
    ep.network = n
    ep.processOptions(options...)

    d := n.driver
    err := d.CreateEndpoint(n.id, ep.id, ep, ep.generic)
    if err != nil {
        return nil, err
    }

    n.Lock()
    n.endpoints[ep.id] = ep
    n.Unlock()
    return ep, nil
}
```

跟踪到 libnetwork/drivers/bridge/bridge.go

```golang
func (d *driver) CreateEndpoint(nid, eid types.UUID, epInfo driverapi.EndpointInfo, epOptions map[string]interface{}) error {
    // ...
    // v4 address for the sandbox side pipe interface
    ip4, err := ipAllocator.RequestIP(n.bridge.bridgeIPv4, nil)
    if err != nil {
        return err
    }
    ipv4Addr := &net.IPNet{IP: ip4, Mask: n.bridge.bridgeIPv4.Mask}
    // ...
}
```

`RequestIP` 的参数说明：在 bridgeIPv4 的子网内分配一个 IP。

## 最新的 master 分支实现

[libnetwork/network.go](https://github.com/docker/docker/blob/master/vendor/src/github.com/docker/libnetwork/network.go)

```golang
func (n *network) CreateEndpoint(name string, options ...EndpointOption) (Endpoint, error) {
    // ...
    if err = ep.assignAddress(ipam.driver, true, n.enableIPv6 && !n.postIPv6); err != nil {
        return nil, err
    }
    if err = n.addEndpoint(ep); err != nil {
        return nil, err
    }
    // ...
}
```

# 实现

Kubernetes 的代码修改：

**注意**：最好升级到 kubernetes 1.2 和 docker1.10。docker 1.11 把 daemon 拆分了，kubernetes 还未宣称支持。

*TODO*

1. `podIP` 的随机分配和持久化！
1. `podIP` 反映到 `pod.Status` 里！

