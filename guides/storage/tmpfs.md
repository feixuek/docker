# 使用`tmpfs`挂载

卷和绑定挂载使您可以在主机和容器之间共享文件，以便即使容器停止后也可以保留数据。

如果您在`Linux`上运行`Docker`，则还有第三种选择：`tmpfs`挂载。 当您创建带有`tmpfs`挂载的容器时，该容器可以在该容器的可写层之外创建文件。

与卷和绑定挂载相反，`tmpfs`挂载是临时的，并且仅保留在主机内存中。 当容器停止时，`tmpfs`挂载将被删除，并且在其中写入的文件将不会保留。

![临时挂载](imagse/types-of-mounts-tmpfs.png)

这对于在主机或容器可写层中临时存储您不想持久保存的敏感文件很有用。

## `tmpfs`安装的局限性
- 与卷和绑定挂载不同，您不能在容器之间共享`tmpfs`挂载。
- 仅当您在`Linux`上运行`Docker`时，此功能才可用。

## 选择`--tmpfs`或`--mount`标志
最初，`--tmpfs`标志用于独立容器，而`--mount`标志用于群集服务。但是，从`Docker 17.06`开始，您还可以将`--mount`与独立容器一起使用。通常，`--mount`更为明确和详细。最大的区别是`--tmpfs`标志不支持任何可配置的选项。

- `--tmpfs`：挂载`tmpfs`挂载而不允许您指定任何可配置选项，并且只能与独立容器一起使用。
- `--mount`：由多个键值对组成，这些对值之间用逗号分隔，每个键对均由一个`<key>=<value>`元组组成。 `--mount`语法比`--tmpfs`更冗长：
	- 挂载的`type`，可以是`bind`，`volume`或`tmpfs`。本主题讨论`tmpfs`，因此类型始终为`tmpfs`。
	- `destination`将	`tmpfs`安装在容器中安装的路径作为其值。可以指定为`destination`，`dst`或`target`。
	- `tmpfs-size`和`tmpfs-mode`选项。请参阅`tmpfs`选项。

下面的示例在可能的情况下同时显示`--mount`和`--tmpfs`语法，并且首先显示`--mount`。

### `--tmpfs`和`--mount`行为之间的区别
- `--tmpfs`标志不允许您指定任何可配置的选项。
- `--tmpfs`标志不能与`swarm`服务一起使用。您必须使用`--mount`。

## 在容器中使用`tmpfs`挂载
要在容器中使用`tmpfs`挂载，请使用`--tmpfs`标志，或将`--mount`标志与`type=tmpfs`和`destination`选项一起使用。没有`tmpfs`挂载的源。以下示例在`Nginx`容器中的`/app`处创建`tmpfs`挂载。第一个示例使用`--mount`标志，第二个示例使用`--tmpfs`标志。

```shell
$ docker run -d \
  -it \
  --name tmptest \
  --mount type=tmpfs,destination=/app \
  nginx:latest
```
```shell
$ docker run -d \
  -it \
  --name tmptest \
  --tmpfs /app \
  nginx:latest
```
通过运行`docker`容器检查`tmptest`并查找`Mounts`部分，验证该挂载是否为`tmpfs`挂载：
```shell
"Tmpfs": {
    "/app": ""
},
```
删除容器
```shell
$ docker container stop tmptest

$ docker container rm tmptest
```
### 指定`tmpfs`选项
`tmpfs`挂载允许两个配置选项，都不是必需的。 如果需要指定这些选项，则必须使用`--mount`标志，因为`--tmpfs`标志不支持它们。

| 选项 | 描述 |
| --- | --- |
| `tmpfs-siz` |`tmpfs`挂载的大小（以字节为单位）。 默认情况下不受限制。 |
| `tmpfs-mode` | `tmpfs`的文件模式（八进制）。 例如`700`或`0770`。默认值为`1777`或世界可写 |

下面的示例将`tmpfs-mode`设置为`1770`，以使它在容器内不可读。
```shell
docker run -d \
  -it \
  --name tmptest \
  --mount type=tmpfs,destination=/app,tmpfs-mode=1770 \
  nginx:latest
```