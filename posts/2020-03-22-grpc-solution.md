# GRPC 解决方案

1. GRPC 是微服务的最终解决方案
2. 服务注册，用 https://github.com/etcd-io/etcd/tree/master/clientv3/naming
3. 服务发现，用 https://github.com/grpc/grpc-go/tree/master/examples/features
4. 客户端接入，用能接入服务发现的自研网关
5. 我的参考实现 https://github.com/aclisp/grpc-padentic-helloworld
   - client 支持 stream 的 golang grpc 客户端
   - server 实现了 stream 
   - router 实现路由管理
   - proxy 接入服务发现的 grpc 透明代理
   - json_proxy 一个简易的 json grpc 透明代理
