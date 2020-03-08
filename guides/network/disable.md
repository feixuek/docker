# 禁用容器网络
如果要完全禁用容器上的网络堆栈，可以在启动容器时使用`--network none`标志。 在容器内，仅创建回送设备。 以下示例说明了这一点。

1. 创建容器。
```shell
$ docker run --rm -dit \
  --network none \
  --name no-net-alpine \
  alpine:latest \
  ash
```

2. 通过在容器内执行一些常见的联网命令，检查容器的网络堆栈。 请注意，没有创建`eth0`。
```shell
$ docker exec no-net-alpine ip link show

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN qlen 1
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
```
```shell
$ docker exec no-net-alpine ip route
```
第二个命令返回空，因为没有路由表。

3. 停止容器。 它是使用--rm标志创建的，因此会自动删除。
```shell
$ docker container rm no-net-alpine
```