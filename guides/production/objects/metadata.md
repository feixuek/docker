# Docker对象标签

标签是一种将元数据应用于Docker对象的机制，包括：

- 镜像
- 容器
- 本地守护程序
- 卷
- 网路
- `Swarm`节点
- `Swarm`服务

您可以使用标签来组织镜像，记录许可信息，注释容器，卷和网络之间的关系，或者以对您的业务或应用程序有意义的任何方式。

## 标签键和值
标签是键值对，存储为字符串。您可以为一个对象指定多个标签，但是每个键值对在一个对象内必须唯一。如果给同一个键赋予多个值，则最近写入的值将覆盖所有先前的值。

### 键格式建议
标签键是键值对的左侧。键是字母数字字符串，可能包含句点(`.`)和连字符(`-`)。大多数Docker用户都使用其他组织创建的镜像，并且以下准则有助于防止跨对象的标签意外复制，尤其是当您计划将标签用作自动化机制时。

- 第三方工具的作者应在每个标签密钥之前添加其拥有的域的反向`DNS`表示法，例如`com.example.some-label`。
- 未经域所有者的许可，请勿在标签密钥中使用域。
- `com.docker.*`，`io.docker.*`和`org.dockerproject.*`名称空间由`Docker`保留供内部使用。
- 标签键应以小写字母开头和结尾，并且只能包含小写字母数字字符，句点字符(`.`)和连字符(`-`)。不允许连续的句号或连字符。
- 句点字符(`.`)分隔名称空间"字段"。不带名称空间的标签键保留供`CLI`使用，从而使`CLI`的用户可以使用较短的键入友好字符串来交互式地标记`Docker`对象。

这些准则目前尚未执行，其他准则可能适用于特定用例。

### 值准则
标签值可以包含任何可以表示为字符串的数据类型，包括（但不限于）`JSON`，`XML`，`CSV`或`YAML`。唯一的要求是使用特定于结构类型的机制，首先将值序列化为字符串。例如，要将`JSON`序列化为字符串，可以使用`JSON.stringify()``JavaScript`方法。

由于`Docker`不会反序列化该值，因此除非您将此功能内置到第三方工具中，否则在按标签值查询或过滤时，您无法将`JSON`或`XML`文档视为嵌套结构。

## 管理对象上的标签
每种支持标签的对象都有机制来添加和管理它们，并在它们与该对象类型相关时使用它们。这些链接提供了一个开始学习如何在`Docker`部署中使用标签的好地方。

镜像，容器，本地守护程序，卷和网络上的标签在对象的生存期内是静态的。要更改这些标签，必须重新创建对象。`Swarm`节点和服务上的标签可以动态更新。

- 镜像和容器
	- [为镜像添加标签](https://docs.docker.com/engine/reference/builder/#label)
	- [在运行时覆盖容器的标签](https://docs.docker.com/engine/reference/commandline/run/#set-metadata-on-container--l---label---label-file)
	- [检查镜像或容器上的标签](https://docs.docker.com/engine/reference/commandline/inspect/)
	- [按标签过滤镜像](https://docs.docker.com/engine/reference/commandline/images/#filtering)
	- [按标签过滤容器](https://docs.docker.com/engine/reference/commandline/ps/#filtering)
- 本地`Docker`守护程序
	- [在运行时将标签添加到`Docker`守护程序](https://docs.docker.com/engine/reference/commandline/dockerd/)
	- [检查`Docker`守护程序的标签](https://docs.docker.com/engine/reference/commandline/info/)
- 卷
	- [向卷添加标签](https://docs.docker.com/engine/reference/commandline/volume_create/)
	- [检查卷的标签](https://docs.docker.com/engine/reference/commandline/volume_inspect/)
	- [按标签过滤卷](https://docs.docker.com/engine/reference/commandline/volume_ls/#filtering)
- 网路
	- [向网络添加标签](https://docs.docker.com/engine/reference/commandline/network_create/)
	- [检查网络的标签](https://docs.docker.com/engine/reference/commandline/network_inspect/)
	- [按标签过滤网络](https://docs.docker.com/engine/reference/commandline/network_ls/#filtering)
- `Swarm`节点
	- [添加或更新`Swarm`节点的标签](https://docs.docker.com/engine/reference/commandline/node_update/#add-label-metadata-to-a-node)
	- [检查`Swarm`节点的标签](https://docs.docker.com/engine/reference/commandline/node_inspect/)
	- [按标签过滤`Swarm`节点](https://docs.docker.com/engine/reference/commandline/node_ls/#filtering)
- `Swarm`服务
	- [创建`Swarm`服务时添加标签](https://docs.docker.com/engine/reference/commandline/service_create/#set-metadata-on-a-service-l-label)
	- [更新`Swarm`服务的标签](https://docs.docker.com/engine/reference/commandline/service_update/)
	- [检查`Swarm`服务的标签](https://docs.docker.com/engine/reference/commandline/service_inspect/)
	- [按标签过滤`Swarm`服务](https://docs.docker.com/engine/reference/commandline/service_ls/#filtering)