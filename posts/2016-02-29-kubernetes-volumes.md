# Kubernetes 存储设计

# 存储的用法

基于 Kubernetes 的 Volume `emptyDir` 扩展出一套适用于业务容器化的存储方案。

*   medium = ""
    默认模式。Volume 的生命期等同于 Pod 的生命期。适用于任何要用 docker volume 的场景。

*   medium = "Memory"
    内存盘模式。Volume 的生命期仍然等同于 Pod 的生命期，但是 Host Reboot 会丢失所有数据。适用于要高速访问的持久化临时数据，缓存等。

*   medium = "Disk"
    自动为每个新出现的 Pod 分配空闲的目录 `/data[1-9]` 。Volume 的生命期大于 Pod 的生命期，在 DELETE Pod 之后仍然保留，直到被用户 rm 或者 mv。适用于存放重要业务数据的场景，如 mysql 的数据目录。用户自行决定 /dataN 是普通硬盘，SSD 等。

# 存储的实现

Kubernetes 的代码修改：

1. `pkg/volume/empty_dir/empty_dir.go`


