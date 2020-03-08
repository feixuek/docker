# 使用Macvlan网络

某些应用程序，尤其是旧版应用程序或监视网络流量的应用程序，期望直接连接到物理网络。在这种情况下，您可以使用`macvlan`网络驱动程序为每个容器的虚拟网络接口分配`MAC`地址，使其看起来像是直接连接到物理网络的物理网络接口。在这种情况下，您需要在`Docker`主机上指定一个物理接口以用于`macvlan`，以及`macvlan`的子网和网关。您甚至可以使用不同的物理网络接口隔离您的`macvlan`网络。请记住以下几点：

- 由于`IP`地址耗尽或“VLAN传播”，很容易无意间损坏您的网络，在这种情况下，您的网络中有大量不正确的唯一`MAC`地址。

- 您的网络设备需要能够处理“混杂模式”，在该模式下，可以为一个物理接口分配多个`MAC`地址。

- 如果您的应用程序可以使用网桥（在单个`Docker`主机上）或覆盖（跨多个`Docker`主机进行通信）工作，那么从长远来看，这些解决方案可能会更好。

## 创建一个macvlan网络
创建`macvlan`网络时，它可以处于桥接模式或`802.1q`中继桥接模式。

- 在桥接模式下，`macvlan`流量通过主机上的物理设备。

- 在`802.1q`中继桥接模式下，流量通过`Docker`动态创建的`802.1q`子接口。这使您可以更精细地控制路由和过滤。

### 桥接模式
要创建与给定物理网络接口桥接的`macvlan`网络，请在`docker network create`命令中使用`--driver macvlan`。您还需要指定父级，这是流量将在`Docker`主机上实际通过的接口。

```shell
$ docker network create -d macvlan \
  --subnet=172.16.86.0/24 \
  --gateway=172.16.86.1 \
  -o parent=eth0 pub_net
```

如果您需要排除在`macvlan`网络中使用`IP`地址，例如已经使用了给定`IP`地址，请使用`--aux-addresses`：
```shell
$ docker network create -d macvlan \
  --subnet=192.168.32.0/24 \
  --ip-range=192.168.32.128/25 \
  --gateway=192.168.32.254 \
  --aux-address="my-router=192.168.32.129" \
  -o parent=eth0 macnet32
```

### 802.1q中继桥接模式
如果您指定一个带有点的父接口名称，例如`eth0.50`，则`Docker`会将其解释为`eth0`的子接口，并自动创建该子接口。
```shell
$ docker network create -d macvlan \
    --subnet=192.168.50.0/24 \
    --gateway=192.168.50.1 \
    -o parent=eth0.50 macvlan50
```

### 使用ipvlan而不是macvlan
在上面的示例中，您仍在使用`L3`桥。 您可以改用`ipvlan`，并获得`L2`桥。 指定`-o ipvlan_mode = l2`。
```shell
$ docker network create -d ipvlan \
    --subnet=192.168.210.0/24 \
    --subnet=192.168.212.0/24 \
    --gateway=192.168.210.254 \
    --gateway=192.168.212.254 \
     -o ipvlan_mode=l2 ipvlan210
```
## 使用IPv6
如果已将`Docker`守护程序配置为允许`IPv6`，则可以使用双栈`IPv4/IPv6 macvlan`网络。
```shell
$ docker network create -d macvlan \
    --subnet=192.168.216.0/24 --subnet=192.168.218.0/24 \
    --gateway=192.168.216.1 --gateway=192.168.218.1 \
    --subnet=2001:db8:abc8::/64 --gateway=2001:db8:abc8::10 \
     -o parent=eth0.218 \
     -o macvlan_mode=bridge macvlan216
```