# 简介

`Docker`容器和服务如此强大的原因之一是您可以将它们连接在一起，或将它们连接到非`Docker`工作负载。 `Docker`容器和服务甚至不需要知道它们已部署在`Docker`上，也不必知道它们的对等对象是否也是`Docker`工作负载。 无论您的`Docker`主机运行`Linux`，`Windows`还是两者结合，您都可以使用`Docker`以与平台无关的方式管理它们。

本主题定义了一些基本的`Docker`网络概念，并使您准备设计和部署应用程序以充分利用这些功能。

大多数内容适用于所有`Docker`安装。 但是，一些高级功能仅适用于`Docker EE`客户。

## 本主题范围
本主题不会涉及有关`Docker`网络如何工作的特定于操作系统的详细信息，因此您将找不到有关`Docker`如何在`Linux`上操作`iptables`规则或如何在`Windows`服务器上操作路由规则的信息，也不会找到有关`Docker`如何工作的详细信息。 形成并封装数据包或处理加密。 请参阅[`Docker iptables`](https://docs.docker.com/network/iptables/)和[`Docker`参考架构：设计可扩展的便携式`Docker`容器网络](https://success.docker.com/article/networking)，以获取更深入的技术细节。

此外，本主题未提供有关如何创建，管理和使用`Docker`网络的任何教程。 每个部分均包含指向相关教程和命令参考的链接。

## 网络驱动

`Docker`的网络子系统可使用驱动程序插入。默认情况下，有几个驱动程序，它们提供核心联网功能：

- `bridge`：默认网络驱动程序。如果您未指定驱动程序，则这是您正在创建的网络类型。当您的应用程序在需要通信的独立容器中运行时，通常会使用网桥网络。请参阅[网桥网络](https://docs.docker.com/network/bridge/)。

- `host`：对于独立容器，取消容器与`Docker`主机之间的网络隔离，并直接使用主机的网络。主机仅可用于`Docker 17.06`及更高版本上的集群服务。请参阅[使用主机网络](https://docs.docker.com/network/host/)。

- `overlay`：覆盖网络将多个`Docker`守护进程连接在一起，并使群集服务能够相互通信。您还可以使用覆盖网络来促进`swarm`服务和独立容器之间或不同`Docker`守护程序上的两个独立容器之间的通信。这种策略消除了在这些容器之间进行操作系统级路由的需要。请参阅[叠加网络](https://docs.docker.com/network/overlay/)。

- `macvlan`：`Macvlan`网络允许您为容器分配`MAC`地址，使其在网络上显示为物理设备。 `Docker`守护程序通过其`MAC`地址将流量路由到容器。在处理希望直接连接到物理网络而不是通过`Docker`主机的网络堆栈进行路由的旧应用程序时，使用`macvlan`驱动程序有时是最佳选择。请参阅[`Macvlan`网络](https://docs.docker.com/network/macvlan/)。

- `none`：对于此容器，禁用所有联网。通常与自定义网络驱动程序一起使用。没有一个不适用于`swarm`服务。请参阅[禁用容器联网](https://docs.docker.com/network/none/)。

- [网络插件](https://docs.docker.com/network/none/)：您可以在`Docker`中安装和使用第三方网络插件。这些插件可从`Docker Hub`或第三方供应商处获得。有关安装和使用给定网络插件的信息，请参阅供应商的文档。

### 网络总结

- 当您需要多个容器在同一`Docker`主机上进行通信时，最好使用用户定义的网桥网络。
- 当网络堆栈不应与`Docker`主机隔离时，但您希望容器的其他方面隔离，则主机网络是最佳选择。
- 当您需要在不同`Docker`主机上运行的容器进行通信时，或者当多个应用程序使用群服务一起工作时，覆盖网络是最好的。
- 从`VM`设置迁移或需要容器看起来像网络上的物理主机（每个都有唯一的`MAC`地址）时，`Macvlan`网络是最好的。
- 第三方网络插件使您可以将`Docker`与专用网络堆栈集成。

## 网络教程

既然您已经了解了有关`Docker`网络的基础知识，请使用以下教程加深您的理解：

- [独立网络教程](https://docs.docker.com/network/network-tutorial-standalone/)
- [主机网络教程](https://docs.docker.com/network/network-tutorial-host/)
- [叠加网络教程](https://docs.docker.com/network/network-tutorial-overlay/)
- [Macvlan网络教程](https://docs.docker.com/network/network-tutorial-macvlan/)