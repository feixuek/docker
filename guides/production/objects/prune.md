# 修剪未使用的Docker对象

`Docker`采取保守的方法来清理未使用的对象（通常称为"垃圾收集"），例如镜像，容器，卷和网络：除非您明确要求`Docker`这样做，否则通常不会删除这些对象。这可能会导致`Docker`使用额外的磁盘空间。对于每种类型的对象，`Docker`提供一个`prune`命令。此外，您可以`docker system prune`用来一次清除多种类型的对象。本主题显示如何使用这些`prune`命令。

## 修剪镜像
该`docker image prune`命令允许您清除未使用的镜像。默认情况下，`docker image prune`仅清除悬空的镜像。悬空镜像是未标记且未被任何容器引用的镜像。要删除悬空的镜像：
```shell
$ docker image prune

WARNING! This will remove all dangling images.
Are you sure you want to continue? [y/N] y
```
要删除现有容器未使用的所有镜像，请使用`-a`标志：
```shell
$ docker image prune -a

WARNING! This will remove all images without at least one container associated to them.
Are you sure you want to continue? [y/N] y
```
默认情况下，系统会提示您继续。要绕过提示，请使用`-f`或`--force`标志。

您可以使用带有`--filter`标志的过滤表达式来限制修剪哪些镜像 。例如，仅考虑24小时前创建的镜像：
```shell
$ docker image prune -a --filter "until=24h"
```
其他过滤表达式也可用。有关 更多示例，请参见[`docker image prune`参考](https://docs.docker.com/engine/reference/commandline/image_prune/)。

## 修剪容器
停止容器时，除非您使用`--rm`标志将其启动，否则不会自动将其删除。要查看`Docker`主机上的所有容器，包括已停止的容器，请使用`docker ps -a`。您可能会惊讶地发现有多少个容器，尤其是在开发系统上！停止的容器的可写层仍会占用磁盘空间。要清理此问题，可以使用`docker container prune`命令。
```shell
$ docker container prune

WARNING! This will remove all stopped containers.
Are you sure you want to continue? [y/N] y
```
默认情况下，系统会提示您继续。要绕过提示，请使用`-f`或`--force`标志。

默认情况下，所有停止的容器都将被删除。您可以使用该`--filter`标志限制范围。例如，以下命令仅删除24小时以上的已停止容器：
```shell
$ docker container prune --filter "until=24h"
```
其他过滤表达式也可用。有关 更多示例，请参见[`docker container prune`参考](https://docs.docker.com/engine/reference/commandline/container_prune/)。

## 修剪卷
卷可由一个或多个容器使用，并占用`Docker`主机上的空间。卷不会自动删除，因为这样做可能会破坏数据。
```shell
$ docker volume prune

WARNING! This will remove all volumes not used by at least one container.
Are you sure you want to continue? [y/N] y
```
默认情况下，系统会提示您继续。要绕过提示，请使用`-f`或`--force`标志。

默认情况下，将删除所有未使用的卷。您可以使用该`--filter`标志限制范围。例如，以下命令仅删除没有标签的卷`keep`：
```shell
$ docker volume prune --filter "label!=keep"
```
其他过滤表达式也可用。有关 更多示例，请参见[`docker volume prune`参考](https://docs.docker.com/engine/reference/commandline/volume_prune/)。

## 修剪网络
`Docker`网络不会占用太多磁盘空间，但它们确实会创建`iptables`规则，桥接网络设备和路由表条目。要清理这些东西，您可以`docker network prune`用来清理任何容器都没有使用的网络。
```shell
$ docker network prune

WARNING! This will remove all networks not used by at least one container.
Are you sure you want to continue? [y/N] y
```
默认情况下，系统会提示您继续。要绕过提示，请使用`-f`或 `--force`标志。

默认情况下，将删除所有未使用的网络。您可以使用该`--filter`标志限制范围。例如，以下命令仅删除早于24小时的网络：
```shell
$ docker network prune --filter "until=24h"
```
其他过滤表达式也可用。有关 更多示例，请参见[`docker network prune`参考](https://docs.docker.com/engine/reference/commandline/network_prune/)。

## 修剪一切
该`docker system prune`命令是修剪镜像，容器和网络的快捷方式。在`Docker 17.06.0`及更早版本中，还修剪了卷。在`Docker 17.06.1`及更高版本中，您必须指定`--volumes`用于`docker system prune`修剪卷的标志 。
```shell
$ docker system prune

WARNING! This will remove:
        - all stopped containers
        - all networks not used by at least one container
        - all dangling images
        - all build cache
Are you sure you want to continue? [y/N] y
```
如果您使用的是`Docker 17.06.1`或更高版本，并且还希望修剪卷，请添加-`-volumes`标志：
```shell
$ docker system prune --volumes

WARNING! This will remove:
        - all stopped containers
        - all networks not used by at least one container
        - all volumes not used by at least one container
        - all dangling images
        - all build cache
Are you sure you want to continue? [y/N] y
```
默认情况下，系统会提示您继续。要绕过提示，请使用`-f`或`--force`标志。