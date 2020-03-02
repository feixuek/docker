# Docker和iptables

在`Linux`上，`Docker`处理`iptables`规则以提供网络隔离。尽管这是一个实现细节，并且您不应修改`Docker`插入`iptables`策略中的规则，但是如果您想要拥有自己的策略（除了由`Docker`管理的策略），它确实会对您需要做的事情产生一些影响。

如果您在暴露于`Internet`的主机上运行`Docker`，则可能需要采用`iptables`策略，以防止未经授权访问您主机上运行的容器或其他服务。此页面描述了如何实现此目标以及需要注意的注意事项。

## 在Docker的规则之前添加iptables策略
`Docker`安装了两个名为`DOCKER-USER`和`DOCKER`的自定义`iptables`链，并确保始终由这两个链首先检查传入的数据包。

`Docker`的所有`iptables`规则都已添加到`DOCKER`链中。请勿手动操作此链条。如果您需要添加在`Docker`规则之前加载的规则，请将其添加到`DOCKER-USER`链中。在`Docker`自动创建任何规则之前，将应用这些规则。

在这些链之后，将评估添加到`FORWARD`链中的规则-手动或通过另一个基于`iptables`的防火墙。这意味着，如果您通过`Docker`公开端口，则无论防火墙配置了什么规则，该端口都会公开。如果您希望即使在通过`Docker`公开端口时也要应用这些规则，则必须将这些规则添加到`DOCKER-USER`链中。

### 限制与Docker主机的连接
默认情况下，允许所有外部源`IP`连接到`Docker`主机。要仅允许特定的`IP`或网络访问容器，请在`DOCKER-USER`过滤器链的顶部插入一个否定的规则。例如，以下规则限制了除`192.168.1.1`之外的所有`IP`地址的外部访问：
```shell
$ iptables -I DOCKER-USER -i ext_if ! -s 192.168.1.1 -j DROP
```
请注意，您需要更改`ext_if`以与主机的实际外部接口相对应。 您可以改为允许来自源子网的连接。 以下规则仅允许从子网`192.168.1.0/24`访问：
```shell
$ iptables -I DOCKER-USER -i ext_if ! -s 192.168.1.0/24 -j DROP
```
最后，您可以使用`--src-range`指定要接受的`IP`地址范围（使用`--src-range`或`--dst-range`时也要添加`-m iprange`）：
```shell
$ iptables -I DOCKER-USER -m iprange -i ext_if ! --src-range 192.168.1.1-192.168.1.3 -j DROP
```
您可以将`-s`或`--src-range`与`-d`或`--dst-range`结合使用以控制源和目标。 例如，如果`Docker`守护程序同时监听`192.168.1.99`和`10.1.2.3`，则可以制定特定于`10.1.2.3`的规则，并使`192.168.1.99`保持打开状态。

`iptables`很复杂，更复杂的规则超出了本主题的范围。 有关更多信息，请参见[Netfilter.org HOWTO](https://www.netfilter.org/documentation/HOWTO/NAT-HOWTO.html)。

## 路由器上的Docker
`Docker`还将`FORWARD`链的策略设置为`DROP`。 如果您的`Docker`主机也充当路由器，这将导致该路由器不再转发任何流量。 如果希望系统继续充当路由器，则可以向`DOCKER-USER`链添加显式`ACCEPT`规则以允许它：
```shell
$ iptables -I DOCKER-USER -i src_if -o dst_if -j ACCEPT
```

## 防止Docker操作iptables
可以在`/etc/docker/daemon.json`的`Docker`引擎的配置文件中将`iptables`密钥设置为`false`，但是此选项不适用于大多数用户。 不可能完全阻止`Docker`创建`iptables`规则，并且事后创建它们非常困难，并且超出了这些说明的范围。 将`iptables`设置为`false`很有可能会破坏`Docker`引擎的容器网络。

对于希望将`Docker`运行时构建到其他应用程序中的系统集成商，请探索[`moby`项目](https://mobyproject.org/)。

## 设置容器的默认绑定地址
默认情况下，`Docker`守护进程将公开`0.0.0.0`地址上的端口，即主机上的任何地址。 如果要更改该行为以仅公开内部`IP`地址上的端口，则可以使用`--ip`选项指定其他`IP`地址。 但是，设置`--ip`仅更改默认值，而不将服务限制为该`IP`。