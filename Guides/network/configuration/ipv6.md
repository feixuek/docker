# 启用IPv6支持

在`Docker`容器或`swarm`服务中使用`IPv6`之前，需要在`Docker`守护进程中启用`IPv6`支持。 之后，您可以选择对任何容器，服务或网络使用`IPv4`或`IPv6`（或两者）。

> 注意：仅在`Linux`主机上运行的`Docker`守护进程支持`IPv6`网络。

1. 编辑`/etc/docker/daemon.json`并将`ipv6`密钥设置为`true`。
```shell
{
  "ipv6": true
}
```
保存文件

2. 重新加载`Docker`配置文件
```shell
$ systemctl reload docker
```
现在，您可以使用`--ipv6`标志创建网络，并使用`--ip6`标志分配容器`IPv6`地址。