# 使用覆盖网络

覆盖网络驱动程序会在多个`Docker`守护程序主机之间创建一个分布式网络。该网络位于特定于主机的网络之上（（覆盖）特定主机的网络），从而在启用加密后允许与其连接的容器（包括`swarm`服务容器）进行安全通信。 `Docker`透明地处理每个数据包与正确的`Docker`守护程序主机和正确的目标容器之间的路由。

初始化群集或将`Docker`主机加入现有群集时，将在该`Docker`主机上创建两个新网络：

- 一个称为入口的覆盖网络，用于处理与群服务相关的控制和数据流量。创建群集服务并且不将其连接到用户定义的覆盖网络时，默认情况下它将连接到入口网络。
- 一个名为`docker_gwbridge`的网桥网络，该网络将各个`Docker`守护程序连接到该集群中的其他守护程序。
您可以使用`docker network create`创建用户定义的覆盖网络，就像创建用户定义的桥网络一样。服务或容器可以一次连接到多个网络。服务或容器只能在它们各自连接的网络之间进行通信。

尽管您可以将群集服务和独立容器都连接到覆盖网络，但是默认行为和配置问题有所不同。因此，本主题的其余部分分为适用于所有覆盖网络的操作，适用于群集服务网络的操作以及适用于独立容器使用的覆盖网络的操作。

## 覆盖网络的通用操作

### 创建一个覆盖网络
> 前提
> - 使用覆盖网络的`Docker`守护程序的防火墙规则
> 您需要打开以下端口，以往返于参与覆盖网络的每个`Docker`主机的流量：
> 	- 用于群集管理通信的`TCP`端口2377
> 	- `TCP`和`UDP`端口7946，用于节点之间的通信
> 	- `UDP`端口4789，用于覆盖网络流量
> - 在创建覆盖网络之前，您需要使用`docker swarm init`将`Docker`守护程序初始化为`swarm`管理器，或者使用`docker swarm join`将其加入现有的`swarm`。 这两种方法均会创建默认的入口覆盖网络，默认情况下，群集服务会使用该入口网络。 即使您从未计划使用群体服务，也需要这样做。 之后，您可以创建其他用户定义的覆盖网络。

要创建用于群服务的覆盖网络，请使用以下命令：
```shell
$ docker network create -d overlay my-overlay
```
要创建覆盖网络，群集服务或独立容器可以使用该覆盖网络与在其他`Docker`守护程序上运行的其他独立容器进行通信，请添加`--attachable`标志：
```shell
$ docker network create -d overlay --attachable my-attachable-overlay
```
您可以指定IP地址范围，子网，网关和其他选项。 有关详细信息，请参阅`docker network create --help`。

###  加密覆盖网络上的流量
默认情况下，在`GCM`模式下使用`AES`算法对所有群集服务管理流量进行加密。群中的管理器节点每12小时旋转一次用于加密八卦数据的密钥。

要同时加密应用程序数据，请在创建覆盖网络时添加`--opt`加密。这样可以在`vxlan`级别启用`IPSEC`加密。这种加密对性能的影响是不可忽略的，因此在生产中使用此选项之前，应先对其进行测试。

启用覆盖加密后，`Docker`将在所有计划为覆盖网络上附加的服务任务的节点之间创建`IPSEC`隧道。这些隧道还在`GCM`模式下使用`AES`算法，管理器节点每12小时自动旋转一次密钥。

> 不要将`Windows`节点附加到加密的覆盖网络。
> `Windows`不支持覆盖网络加密。如果`Windows`节点尝试连接到加密的覆盖网络，则不会检测到错误，但是该节点无法通信。

群体模式覆盖网络和独立容器
您可以将覆盖网络功能与`--opt`加密`--attachable`一起使用，并将非托管容器附加到该网络：
```shell
$ docker network create --opt encrypted --driver overlay --attachable my-attachable-multi-host-network
```

### 自定义默认入口网络
大多数用户不需要配置入口网络，但是`Docker 17.05`和更高版本允许您进行配置。如果自动选择的子网与网络上已经存在的子网发生冲突，或者您需要自定义其他低级网络设置（例如`MTU`），这将很有用。

自定义入口网络涉及删除并重新创建它。这通常在您在集群中创建任何服务之前完成。如果您具有发布端口的现有服务，则需要先删除这些服务，然后才能删除入口网络。

在不存在入口网络的时间内，未发布端口的现有服务将继续运行，但负载不平衡。这会影响发布端口的服务，例如发布端口`80`的`WordPress`服务。

1. 使用`docker network inspect ingress`，检查入口，并删除容器与其连接的所有服务。这些是发布端口的服务，例如发布端口`80`的`WordPress`服务。如果未停止所有此类服务，则下一步将失败。

2. 删除现有的入口网络：
```shell
$ docker network rm ingress

WARNING! Before removing the routing-mesh network, make sure all the nodes
in your swarm run the same docker engine version. Otherwise, removal may not
be effective and functionality of newly created ingress networks will be
impaired.
Are you sure you want to continue? [y/N]
```
3. 使用`--ingress`标志以及要设置的自定义选项创建一个新的覆盖网络。 本示例将`MTU`设置为`1200`，将子网设置为`10.11.0.0/16`，并将网关设置为`10.11.0.2`。
```shell
$ docker network create \
  --driver overlay \
  --ingress \
  --subnet=10.11.0.0/16 \
  --gateway=10.11.0.2 \
  --opt com.docker.network.driver.mtu=1200 \
  my-ingress
```
> 注意：您可以使用除入口之外的其他名称来命名入口网络，但只能有一个。 尝试创建第二个失败。

4. 重新启动您在第一步中停止的服务。

### 自定义`docker_gwbridge`接口
`docker_gwbridge`是一个虚拟网桥，用于将覆盖网络（包括入口网络）连接到单个`Docker`守护程序的物理网络。 当您初始化群集或将`Docker`主机加入群集时，`Docker`会自动创建它，但它不是`Docker`设备。 它存在于`Docker`主机的内核中。 如果需要自定义其设置，则必须在将`Docker`主机加入群集之前或从群集中暂时删除主机之后进行。

1. 停止Docker。

2. 删除现有的`docker_gwbridge`接口。
```
$ sudo ip link set docker_gwbridge down

$ sudo ip link del dev docker_gwbridge
```
3. 启动`Docker`。 不要加入或初始化群体。

4. 使用`docker network create`命令使用自定义设置手动创建或重新创建`docker_gwbridge`桥。 本示例使用子网`10.11.0.0/16`。 有关可自定义选项的完整列表，请参阅网桥驱动程序选项。
```shell
$ docker network create \
--subnet 10.11.0.0/16 \
--opt com.docker.network.bridge.name=docker_gwbridge \
--opt com.docker.network.bridge.enable_icc=false \
--opt com.docker.network.bridge.enable_ip_masquerade=true \
docker_gwbridge
```
5. 初始化或加入群。 由于该桥已经存在，因此Docker不会使用自动设置来创建它。

## [swarm服务的操作](https://docs.docker.com/network/overlay/#operations-for-swarm-services)
## [覆盖网络上独立容器的操作](https://docs.docker.com/network/overlay/#operations-for-standalone-containers-on-overlay-networks)