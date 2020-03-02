# 与独立容器联网

本系列教程探讨了独立`Docker`容器的网络连接。有关使用群集服务联网，请参阅使用[`swarm`服务联网](https://docs.docker.com/network/network-tutorial-overlay/)。如果您需要总体上了解有关`Docker`网络的更多信息，请参阅概述。

本主题包括三个不同的教程。您可以在`Linux`，`Windows`或`Mac`上运行它们中的每一个，但对于最后两个，则需要在其他位置运行第二个`Docker`主机。

- 使用默认桥接网络演示了如何使用`Docker`自动为您设置的默认桥接网络。该网络不是生产系统的最佳选择。

- 使用用户定义的桥接网络显示了如何创建和使用自己的自定义桥接网络，以连接在同一`Docker`主机上运行的容器。建议将其用于生产中运行的独立容器。

尽管覆盖网络通常用于集群服务，但`Docker 17.06`及更高版本允许您将覆盖网络用于独立容器。这是使用叠加网络的教程的一部分。

## 使用默认桥接网络

在此示例中，您在同一`Docker`主机上启动了两个不同的`alpine`容器，并进行了一些测试以了解它们如何相互通信。 您需要安装并运行`Docker`。

1. 打开一个终端窗口。 在执行其他任何操作之前，请先列出当前网络。 如果您从未在此`Docker`守护程序上添加网络或初始化群组，则应该看到以下内容。 您可能会看到不同的网络，但至少应该看到以下内容（网络ID会有所不同）：
```shell
$ docker network ls

NETWORK ID          NAME                DRIVER              SCOPE
17e324f45964        bridge              bridge              local
6ed54d316334        host                host                local
7092879f2cc8        none                null                local
```
列出了默认的`bridge`网络，以及`host`和`none`。 后两个不是完全成熟的网络，但用于启动直接连接到`Docker`守护程序主机的网络堆栈的容器，或用于启动不包含网络设备的容器。 本教程将把两个容器连接到`bridge`网络。

2. 启动两个运行`ash`的`alpine`容器，这是`alpine`的默认外壳，而不是`bash`。 `-dit`标志的意思是启动分离的容器（在后台），交互的（具有键入的能力）和`TTY`（以便您可以看到输入和输出）。 由于您是分开启动的，因此您不会立即连接到该容器。 而是将打印容器的ID。 因为您未指定任何`--network`标志，所以容器将连接到默认的`bridge`网络。
```shell
$ docker run -dit --name alpine1 alpine ash

$ docker run -dit --name alpine2 alpine ash
```
检查两个容器是否都启动了
```shell
$ docker container ls

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
602dbf1edc81        alpine              "ash"               4 seconds ago       Up 3 seconds                            alpine2
da33b7aa74b0        alpine              "ash"               17 seconds ago      Up 16 seconds                           alpine1
```

3. 检查`bridge`网络以查看连接了哪些容器。
```shell
$ docker network inspect bridge

[
    {
        "Name": "bridge",
        "Id": "17e324f459648a9baaea32b248d3884da102dde19396c25b30ec800068ce6b10",
        "Created": "2017-06-22T20:27:43.826654485Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "602dbf1edc81813304b6cf0a647e65333dc6fe6ee6ed572dc0f686a3307c6a2c": {
                "Name": "alpine2",
                "EndpointID": "03b6aafb7ca4d7e531e292901b43719c0e34cc7eef565b38a6bf84acf50f38cd",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "da33b7aa74b0bf3bda3ebd502d404320ca112a268aafe05b4851d1e3312ed168": {
                "Name": "alpine1",
                "EndpointID": "46c044a645d6afc42ddd7857d19e9dcfb89ad790afb5c239a35ac0af5e8a5bc5",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```
在顶部附近，列出了有关`birdge`网络的信息，包括`Docker`主机和`bridge`网络之间的网关的`IP`地址（`172.17.0.1`）。 在`Containers`项下，列出了每个连接的容器及其`IP`地址信息（`alpine1`为`172.17.0.2`，`alpine2`为`172.17.0.3`）。

4. 容器在后台运行。 使用`docker attach`命令连接到`alpine1`。
```shell
$ docker attach alpine1

/ #
```
提示符将更改为`＃`以指示您是容器中的`root`用户。 使用`ip addr show`命令从容器中查看`alpine1`的网络接口：
```shell
# ip addr show

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
27: eth0@if28: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:2/64 scope link
       valid_lft forever preferred_lft forever
```

第一个接口是回送设备。 现在忽略它。 请注意，第二个接口的IP地址为`172.17.0.2`，与上一步中为`alpine1`显示的地址相同。

5. 在`alpine1`内部，通过`ping google.com`确保您可以连接到互联网。 `-c 2`标志将命令限制为两次`ping`尝试。
```shell
# ping -c 2 google.com

PING google.com (172.217.3.174): 56 data bytes
64 bytes from 172.217.3.174: seq=0 ttl=41 time=9.841 ms
64 bytes from 172.217.3.174: seq=1 ttl=41 time=9.897 ms

--- google.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 9.841/9.869/9.897 ms
```

6. 现在尝试`ping`第二个容器。 首先，通过其`IP`地址`172.17.0.3`对其执行`ping`操作：
```shell
# ping -c 2 172.17.0.3

PING 172.17.0.3 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.086 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.094 ms

--- 172.17.0.3 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.086/0.090/0.094 ms
```
这样成功了。 接下来，尝试按容器名称`ping alpine2`容器。 这将失败。

```shell
# ping -c 2 alpine2

ping: bad address 'alpine2'
```

7. 通过使用分离序列`CTRL + p` `CTRL + q`（按住`CTRL`并键入`p`后跟`q`）从`alpine1`分离而不停止它。 如果愿意，请附加到`alpine2`并在那里重复步骤4、5和6，用`alpine1`代替`alpine2`。

8. 停止并卸下两个容器。
```shell
$ docker container stop alpine1 alpine2
$ docker container rm alpine1 alpine2
```

请记住，不建议将默认`bridge`网络用于生产。 要了解用户定义的`bridge`网络，请继续阅读下一个教程。

## 使用用户定义的桥接网络

在此示例中，我们再次启动两个`alpine`容器，但是将它们附加到我们已经创建的名为`alpine-net`的用户定义网络上。 这些容器根本没有连接到默认桥网络。 然后，我们启动连接到`bridge`网络但未连接到`alpine-net`的第三个`apline`容器，以及连接到两个网络的第四个`alpine`容器。


1. 创建`alpine-net`。 您不需要`--driver bridge`标志，因为它是默认标志，但是此示例显示了如何指定它。

```shell
$ docker network create --driver bridge alpine-net
```

2. 列出`Docker`的网路
```shell
$ docker network ls

NETWORK ID          NAME                DRIVER              SCOPE
e9261a8c9a19        alpine-net          bridge              local
17e324f45964        bridge              bridge              local
6ed54d316334        host                host                local
7092879f2cc8        none                null                local
```
检查`alpine-net`。 这显示了它的`IP`地址以及没有容器连接到它的事实：
```shell
$ docker network inspect alpine-net

[
    {
        "Name": "alpine-net",
        "Id": "e9261a8c9a19eabf2bf1488bf5f208b99b1608f330cff585c273d39481c9b0ec",
        "Created": "2017-09-25T21:38:12.620046142Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```
请注意，该网络的网关是`172.18.0.1`，而默认`bridge`网络的网关是172.17.0.1。 您系统上的确切`IP`地址可能不同。

3. 创建四个容器。 注意`--network`标志。 您只能在`docker run`命令期间连接到一个网络，因此您以后需要使用`docker network connect`来将`alpine4`连接到`bridge`网络。
```shell
$ docker run -dit --name alpine1 --network alpine-net alpine ash

$ docker run -dit --name alpine2 --network alpine-net alpine ash

$ docker run -dit --name alpine3 alpine ash

$ docker run -dit --name alpine4 --network alpine-net alpine ash

$ docker network connect bridge alpine4
```
确认所有容器都跑起来了
```shell
$ docker container ls

CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
156849ccd902        alpine              "ash"               41 seconds ago       Up 41 seconds                           alpine4
fa1340b8d83e        alpine              "ash"               51 seconds ago       Up 51 seconds                           alpine3
a535d969081e        alpine              "ash"               About a minute ago   Up About a minute                       alpine2
0a02c449a6e9        alpine              "ash"               About a minute ago   Up About a minute                       alpine1
```

4. 再次检查`bridge`网络和`alpine-net`网络
```shell
$ docker network inspect bridge

[
    {
        "Name": "bridge",
        "Id": "17e324f459648a9baaea32b248d3884da102dde19396c25b30ec800068ce6b10",
        "Created": "2017-06-22T20:27:43.826654485Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "156849ccd902b812b7d17f05d2d81532ccebe5bf788c9a79de63e12bb92fc621": {
                "Name": "alpine4",
                "EndpointID": "7277c5183f0da5148b33d05f329371fce7befc5282d2619cfb23690b2adf467d",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "fa1340b8d83eef5497166951184ad3691eb48678a3664608ec448a687b047c53": {
                "Name": "alpine3",
                "EndpointID": "5ae767367dcbebc712c02d49556285e888819d4da6b69d88cd1b0d52a83af95f",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```
`alpine3`和`alpine4`容器连接到`bridge`网络
```shell
$ docker network inspect alpine-net

[
    {
        "Name": "alpine-net",
        "Id": "e9261a8c9a19eabf2bf1488bf5f208b99b1608f330cff585c273d39481c9b0ec",
        "Created": "2017-09-25T21:38:12.620046142Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "0a02c449a6e9a15113c51ab2681d72749548fb9f78fae4493e3b2e4e74199c4a": {
                "Name": "alpine1",
                "EndpointID": "c83621678eff9628f4e2d52baf82c49f974c36c05cba152db4c131e8e7a64673",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            },
            "156849ccd902b812b7d17f05d2d81532ccebe5bf788c9a79de63e12bb92fc621": {
                "Name": "alpine4",
                "EndpointID": "058bc6a5e9272b532ef9a6ea6d7f3db4c37527ae2625d1cd1421580fd0731954",
                "MacAddress": "02:42:ac:12:00:04",
                "IPv4Address": "172.18.0.4/16",
                "IPv6Address": ""
            },
            "a535d969081e003a149be8917631215616d9401edcb4d35d53f00e75ea1db653": {
                "Name": "alpine2",
                "EndpointID": "198f3141ccf2e7dba67bce358d7b71a07c5488e3867d8b7ad55a4c695ebb8740",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
```
`alpine1`,`alpine2`和`alpine4`容器连接到`bridge`网络

5. 在像`alpine-net`这样的用户定义网络上，容器不仅可以按`IP`地址进行通信，而且还可以将容器名称解析为`IP`地址。 此功能称为自动服务发现。 让我们连接到`alpine1`并进行测试。 `alpine1`应该能够将`alpine2`和`alpine4`（以及`alpine1`本身）解析为`IP`地址。
```shell
$ docker container attach alpine1

# ping -c 2 alpine2

PING alpine2 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.085 ms
64 bytes from 172.18.0.3: seq=1 ttl=64 time=0.090 ms

--- alpine2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.085/0.087/0.090 ms

# ping -c 2 alpine4

PING alpine4 (172.18.0.4): 56 data bytes
64 bytes from 172.18.0.4: seq=0 ttl=64 time=0.076 ms
64 bytes from 172.18.0.4: seq=1 ttl=64 time=0.091 ms

--- alpine4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.076/0.083/0.091 ms

# ping -c 2 alpine1

PING alpine1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.026 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.054 ms

--- alpine1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.026/0.040/0.054 ms
```

6. 从`alpine1`，您根本不能连接到`alpine3`，因为它不在`alpine-net`网络上。
```shell
# ping -c 2 alpine3

ping: bad address 'alpine3'
```
不仅如此，您也无法通过`alpine`1通过其`IP`地址连接到`alpine3`。 回顾`docker`网络，检查`bridge`网络的输出，并找到`alpine3`的`IP`地址：`172.17.0.2`尝试`ping`通它。
```shell
# ping -c 2 172.17.0.2

PING 172.17.0.2 (172.17.0.2): 56 data bytes

--- 172.17.0.2 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

使用分离序列`CTRL + p` `CTRL + q`（按住`CTRL`并键入`p`后跟`q`）从`alpine1`分离。

7. 请记住，`alpine4`已连接到默认`bridge`网络和`alpine-net`。 它应该能够到达所有其他容器。 但是，您将需要按其`IP`地址寻址`alpine3`。 附加到它并运行测试。
```shell
 docker container attach alpine4

# ping -c 2 alpine1

PING alpine1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.074 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.082 ms

--- alpine1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.074/0.078/0.082 ms

# ping -c 2 alpine2

PING alpine2 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.075 ms
64 bytes from 172.18.0.3: seq=1 ttl=64 time=0.080 ms

--- alpine2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.075/0.077/0.080 ms

# ping -c 2 alpine3
ping: bad address 'alpine3'

# ping -c 2 172.17.0.2

PING 172.17.0.2 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.089 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.075 ms

--- 172.17.0.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.075/0.082/0.089 ms

# ping -c 2 alpine4

PING alpine4 (172.18.0.4): 56 data bytes
64 bytes from 172.18.0.4: seq=0 ttl=64 time=0.033 ms
64 bytes from 172.18.0.4: seq=1 ttl=64 time=0.064 ms

--- alpine4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.033/0.048/0.064 ms
```
8. 作为最终测试，请通过`ping google.com`确保您的容器都可以连接到互联网。 您已经对`alpine4`感兴趣，因此从那里尝试开始。 接下来，从`alpine4`分离并连接到`alpine3`（仅连接到`bridge`网络），然后重试。 最后，连接到`alpine1`（仅连接到`alpine-net`网络），然后重试。
```shell
# ping -c 2 google.com

PING google.com (172.217.3.174): 56 data bytes
64 bytes from 172.217.3.174: seq=0 ttl=41 time=9.778 ms
64 bytes from 172.217.3.174: seq=1 ttl=41 time=9.634 ms

--- google.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 9.634/9.706/9.778 ms

CTRL+p CTRL+q

$ docker container attach alpine3

# ping -c 2 google.com

PING google.com (172.217.3.174): 56 data bytes
64 bytes from 172.217.3.174: seq=0 ttl=41 time=9.706 ms
64 bytes from 172.217.3.174: seq=1 ttl=41 time=9.851 ms

--- google.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 9.706/9.778/9.851 ms

CTRL+p CTRL+q

$ docker container attach alpine1

# ping -c 2 google.com

PING google.com (172.217.3.174): 56 data bytes
64 bytes from 172.217.3.174: seq=0 ttl=41 time=9.606 ms
64 bytes from 172.217.3.174: seq=1 ttl=41 time=9.603 ms

--- google.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 9.603/9.604/9.606 ms

CTRL+p CTRL+q
```

9. 停止并删除所有容器和`network-net`网络
```shell
$ docker container stop alpine1 alpine2 alpine3 alpine4

$ docker container rm alpine1 alpine2 alpine3 alpine4

$ docker network rm alpine-net
```

##  其他网络教程
既然您已经完成了独立容器的网络教程，那么您可能需要运行以下其他网络教程：

- [主机网络教程](https://docs.docker.com/network/network-tutorial-host/)
- [叠加网络教程](https://docs.docker.com/network/network-tutorial-overlay/)
- [Macvlan网络教程](https://docs.docker.com/network/network-tutorial-macvlan/)