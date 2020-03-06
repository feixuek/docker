# 使用卷

卷是用于持久化由`Docker`容器生成和使用的数据的首选机制。 尽管绑定挂载取决于主机的目录结构，但卷完全由`Docker`管理。 与绑定挂载相比，卷具有几个优点：

- 与绑定挂载相比，卷更易于备份或迁移。
- 您可以使用`Docker CLI`命令或`Docker API`管理卷。
- 卷在`Linux`和`Windows`容器上均可工作。
- 可以在多个容器之间更安全地共享卷。
- 卷驱动程序使您可以将卷存储在远程主机或云提供程序上，以加密卷内容或添加其他功能。
- 可以通过容器预先填充新卷的内容。

此外，与将数据持久保存在容器的可写层中相比，卷通常是更好的选择，因为卷不会增加使用卷的容器的大小，并且卷的内容存在于给定容器的生命周期之外。

![卷](images/types-of-mounts-volume.png)

如果您的容器生成非持久状态数据，请考虑使用`tmpfs`挂载以避免将数据永久存储在任何地方，并通过避免写入容器的可写层来提高容器的性能。

卷使用`rprivate`绑定传播，并且无法为卷配置绑定传播。

## 选择-v或--mount标志
最初，`-v`或`--volume`标志用于独立容器，而`--mount`标志用于群服务。 但是，从`Docker 17.06`开始，您还可以将`--mount`与独立容器一起使用。 通常，`--mount`更为明确和详细。 最大的区别是`-v`语法在一个字段中将所有选项组合在一起，而`--mount`语法将它们分开。 这是每个标志的语法比较。

> 新用户应尝试使用`--mount`语法，该语法比`--volume`语法更简单。

如果需要指定卷驱动程序选项，则必须使用`--mount`。

- `-v`或`--volume`：由三个字段组成，以冒号`(:)`分隔。这些字段必须以正确的顺序排列，并且每个字段的含义不是立即显而易见的。
	- 对于命名卷，第一个字段是卷的名称，在给定的主机上是唯一的。对于匿名卷，将省略第一个字段。
	- 第二个字段是文件或目录在容器中的安装路径。
	- 第三个字段是可选的，并且是选项的逗号分隔列表，例如`ro`。这些选项将在下面讨论。
- `--mount`：由多个键值对组成，这些对值之间用逗号分隔，每个键对均由一个`<key>=<value>`元组组成。`--mount`语法比`-v`或`--volume`更为冗长，但是键的顺序并不重要，并且标志的值更易于理解。
	- 安装的`type`，可以是`bind`，`volume`或`tmpfs`。本主题讨论卷，因此类型始终是`volume`。
	- 挂载的`source`。对于命名卷，这是卷的名称。对于匿名卷，将省略此字段。可以指定为`source`或`src`。
	- `destination`将文件或目录在容器中的安装路径作为其值。可以指定为`destination`，`dst`或`target`。
	- `readonly`选项（如果存在）使绑定安装以只读方式安装到容器中。
	- 可以多次指定的`volume-opt`选项采用由选项名称及其值组成的键值对。

> 从外部CSV解析器转义值

> 如果您的卷驱动程序接受逗号分隔的列表作为选项，则必须从外部CSV解析器中转义该值。要转义一个`volume-opt`	，请用双引号（`"`）包围它，并用单引号（`'`）包围整个安装参数。

> 例如，`local`驱动程序在`o`参数中接受安装选项作为逗号分隔的列表。 本示例显示了转义列表的正确方法。
> ```shell
$ docker service create \
     --mount 'type=volume,src=<VOLUME-NAME>,dst=<CONTAINER-PATH>,volume-driver=local,volume-opt=type=nfs,volume-opt=device=<nfs-server>:<nfs-path>,"volume-opt=o=addr=<nfs-address>,vers=4,soft,timeo=180,bg,tcp,rw"'
    --name myservice \
    <IMAGE>
  ```

### -v和--mount行为之间的区别
与绑定挂载相反，所有卷选项都可用于`--mount`和`-v`标志。

将卷与服务一起使用时，仅支持`--mount`。

## 创建和管理卷
与绑定挂载不同，您可以在任何容器范围之外创建和管理卷。
```shell
$ docker volume create my-vol
```
列出卷
```shell
$ docker volume ls

local               my-vol
```
检查一个卷
```shell
$ docker volume inspect my-vol
[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```
删除一个卷
```shell
$ docker volume rm my-vol
```

## 启动具有卷的容器
如果您使用不存在的卷启动容器，则`Docker`将为您创建该卷。 以下示例将卷`myvol2`安装到容器中的`/app/`中。

下面的`-v`和`--mount`示例产生相同的结果。 除非在运行第一个容器后删除`devtest`容器和`myvol2`卷，否则都无法运行它们。

```shell
$ docker run -d \
  --name devtest \
  --mount source=myvol2,target=/app \
  nginx:latest
```
```shell
$ docker run -d \
  --name devtest \
  -v myvol2:/app \
  nginx:latest
```
使用`docker inspect devtest`验证卷是否已正确创建和安装。 查找`Mounts`部分：
```shell
"Mounts": [
    {
        "Type": "volume",
        "Name": "myvol2",
        "Source": "/var/lib/docker/volumes/myvol2/_data",
        "Destination": "/app",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    }
],
```
这表明安装是一个卷，它显示了正确的源和目标，并且该安装是可读写的。

停止容器并删除容器。 卷清除是一个单独的步骤。
```shell
$ docker container stop devtest

$ docker container rm devtest

$ docker volume rm myvol2
```

### 启动带卷的服务
启动服务并定义卷时，每个服务容器都使用其自己的本地卷。 如果使用`local`卷驱动程序，则所有容器都不能共享此数据，但是某些卷驱动程序确实支持共享存储。 适用于`AWS`的`Docker`和适用于`Azure`的`Docker`均使用`Cloudstor`插件支持持久存储。

以下示例使用四个副本启动`nginx`服务，每个副本使用一个名为`myvol2`的本地卷。
```shell
$ docker service create -d \
  --replicas=4 \
  --name devtest-service \
  --mount source=myvol2,target=/app \
  nginx:latest
```
使用`docker service ps devtest-serv`确认服务是否运行
```shell
$ docker service ps devtest-service

ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
4d7oz1j85wwn        devtest-service.1   nginx:latest        moby                Running             Running 14 seconds ago
```
删除服务并停止所有任务
```
$ docker service rm devtest-service
```
删除服务不会删除该服务创建的任何卷。 卷删除是一个单独的步骤。

语法服务差异
`docker service create`命令不支持`-v`或`--volume`标志。 将卷安装到服务的容器中时，必须使用`--mount`标志。

### 使用容器填充卷
如果您如上所述启动一个容器来创建新的卷，并且该容器在要挂载的目录中具有文件或目录（例如`/app/`），则目录的内容将被复制到该卷中。 然后，该容器将安装并使用该卷，并且使用该卷的其他容器也可以访问预填充的内容。

为了说明这一点，此示例启动了一个`nginx`容器，并使用容器的`/usr/share/nginx/ html`目录的内容填充新的卷`nginx-vol`，`Nginx`在该目录中存储其默认`HTML`内容。

`--mount`和`-v`示例具有相同的最终结果。
```shell
$ docker run -d \
  --name=nginxtest \
  --mount source=nginx-vol,destination=/usr/share/nginx/html \
  nginx:latest
```
```shell
$ docker run -d \
  --name=nginxtest \
  -v nginx-vol:/usr/share/nginx/html \
  nginx:latest
```
运行这些示例中的任何一个之后，请运行以下命令以清理容器和卷。 卷清除是一个单独的步骤。
```
$ docker container stop nginxtest

$ docker container rm nginxtest

$ docker volume rm nginx-vol
```

## 使用只读卷

对于某些开发应用程序，容器需要写入绑定挂载，以便将更改传播回Docker主机。 在其他时候，容器仅需要对数据的读取访问权限。 请记住，多个容器可以挂载相同的卷，并且可以同时对其中一些容器进行读写挂载，而对其他容器则同时进行只读挂载。

此示例修改了上面的示例，但通过在容器中的安装点之后将`ro`添加到（默认为空）选项列表中，将目录作为只读卷进行挂载。 如果存在多个选项，请用逗号分隔。
```shell
$ docker run -d \
  --name=nginxtest \
  --mount source=nginx-vol,destination=/usr/share/nginx/html,readonly \
  nginx:latest
```
```
$ docker run -d \
  --name=nginxtest \
  -v nginx-vol:/usr/share/nginx/html:ro \
  nginx:latest
```
使用`docker inspect nginxtest`来验证是否正确创建了只读挂载。 查找"Mounts"部分：
```shell
"Mounts": [
    {
        "Type": "volume",
        "Name": "nginx-vol",
        "Source": "/var/lib/docker/volumes/nginx-vol/_data",
        "Destination": "/usr/share/nginx/html",
        "Driver": "local",
        "Mode": "",
        "RW": false,
        "Propagation": ""
    }
],
```
停止并删除容器，删除卷，卷清除是一个单独的步骤。
```shell
$ docker container stop nginxtest

$ docker container rm nginxtest

$ docker volume rm nginx-vol
```

## 机器间共享数据
在构建容错应用程序时，您可能需要配置同一服务的多个副本才能访问相同的文件。

![volumes-shared-storage.svg](images/volumes-shared-storage.svg)

开发应用程序时，有几种方法可以实现此目的。 一种是向您的应用程序添加逻辑，以将文件存储在像`Amazon S3`这样的云对象存储系统上。 另一个方法是使用支持将文件写入外部存储系统（例如`NFS`或`Amazon S3`）的驱动程序来创建卷。

卷驱动程序使您可以从应用程序逻辑中抽象底层存储系统。 例如，如果您的服务使用带有`NFS`驱动程序的卷，则可以更新服务以使用其他驱动程序（例如，将数据存储在云中），而无需更改应用程序逻辑。

## 使用卷驱动程序
使用`docker volume create`创建卷时，或者启动使用尚未创建的卷的容器时，可以指定卷驱动程序。 以下示例使用`vieux/ sshfs`卷驱动程序，首先在创建独立卷时使用，然后在启动创建新卷的容器时使用。

### 初始设置
本示例假定您有两个节点，其中第一个是`Docker`主机，并且可以使用`SSH`连接到第二个。

在`Docker`主机上，安装`vieux/sshfs`插件：
```shell
$ docker plugin install --grant-all-permissions vieux/sshfs
```

### 使用驱动创建卷
此示例指定了`SSH`密码，但是如果两个主机都配置了共享密钥，则可以省略该密码。 每个卷驱动程序可能具有零个或多个可配置选项，每个选项均使用`-o`标志指定。
```shell
$ docker volume create --driver vieux/sshfs \
  -o sshcmd=test@node2:/home/test \
  -o password=testpassword \
  sshvolume
```

### 启动一个使用卷驱动程序创建卷的容器
此示例指定了`SSH`密码，但是如果两个主机都配置了共享密钥，则可以省略该密码。 每个卷驱动程序可能具有零个或多个可配置选项。 如果卷驱动程序要求您传递选项，则必须使用`--mount`标志挂载卷，而不是`-v`。
```shell
$ docker run -d \
  --name sshfs-container \
  --volume-driver vieux/sshfs \
  --mount src=sshvolume,target=/app,volume-opt=sshcmd=test@node2:/home/test,volume-opt=password=testpassword \
  nginx:latest
```

### 创建一个创建`NFS`卷的服务
本示例说明了创建服务时如何创建`NFS`卷。 本示例使用`10.0.0.10`作为`NFS`服务器，并使用`/var/docker-nfs`作为`NFS`服务器上的导出目录。 请注意，指定的卷驱动程序是本地的。

- NFSV3

```
$ docker service create -d \
  --name nfs-service \
  --mount 'type=volume,source=nfsvolume,target=/app,volume-driver=local,volume-opt=type=nfs,volume-opt=device=:/var/docker-nfs,volume-opt=o=addr=10.0.0.10' \
  nginx:latest
```

- NFSV4

```
docker service create -d \
    --name nfs-service \
    --mount 'type=volume,source=nfsvolume,target=/app,volume-driver=local,volume-opt=type=nfs,volume-opt=device=:/,"volume-opt=o=10.0.0.10,rw,nfsvers=4,async"' \
    nginx:latest
```

## 备份，还原或迁移数据卷
卷对于备份，还原和迁移很有用。 使用`--volumes-from`标志来创建一个安装该卷的新容器。

### 备份容器
例如，创建一个名为`dbstore`的新容器：
```shell
$ docker run -v /dbdata --name dbstore ubuntu /bin/bash
```

然后在下一个命令中，我们

- 启动一个新容器，并从`dbstore`容器装入卷
- 将本地主机目录挂载为`/backup`
- 将命令将`dbdata`卷的内容拖到`/backup`目录中的`backup.tar`文件中。

```shell
$ docker run --rm --volumes-from dbstore -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /dbdata
```
当命令完成并且容器停止时，我们剩下了`dbdata`卷的备份。

### 从备份还原容器
使用刚刚创建的备份，您可以将其还原到同一容器或在其他位置创建的另一个容器。

例如，创建一个名为`dbstore2`的新容器：
```shell
$ docker run -v /dbdata --name dbstore2 ubuntu /bin/bash
```
然后将备份文件解压缩到新容器的数据卷中：
```shell
$ docker run --rm --volumes-from dbstore2 -v $(pwd):/backup ubuntu bash -c "cd /dbdata && tar xvf /backup/backup.tar --strip 1"
```
您可以使用首选工具使用上述技术来自动执行备份，迁移和还原测试。

## 删除卷
删除容器后，`Docker`数据卷仍然存在。 有两种类型的卷需要考虑：

- 命名卷具有来自容器外部的特定来源，例如`awesome:/bar`。
- 匿名卷没有特定来源，因此在删除容器时，请指示`Docker Engine`守护程序将其删除。

### 删除匿名卷
要自动删除匿名卷，请使用`--rm`选项。 例如，此命令创建一个匿名`/foo`卷。 删除容器后，`Docker`引擎将删除`/foo`卷，但不会删除`awesome`的卷。
```shell
$ docker run --rm -v /foo -v awesome:/bar busybox top
```

### 删除所有卷
要删除所有未使用的卷并释放空间：
```shell
# docker volume prune
```