# 使用绑定挂载

自`Docker`诞生以来，绑定挂载就已经存在。 与卷相比，绑定挂载的功能有限。 使用绑定挂载时，会将主机上的文件或目录挂载到容器中。 文件或目录由主机上的完整或相对路径引用。 相比之下，当您使用卷时，将在主机上`Docker`的存储目录中创建一个新目录，并且`Docker`管理该目录的内容。

该文件或目录不需要在`Docker`主机上已经存在。 如果尚不存在，则按需创建。 绑定挂载性能非常好，但是它们依赖于具有特定目录结构的主机文件系统。 如果要开发新的`Docker`应用程序，请考虑使用命名卷。 您不能使用`Docker CLI`命令直接管理绑定挂载。

![绑定挂载](images/types-of-mounts-bind.png)

## 选择-v或--mount标志
最初，`-v`或`--volume`标志用于独立容器，而`--mount`标志用于群服务。但是，从`Docker 17.06`开始，您还可以将`--mount`与独立容器一起使用。通常，`--mount`更为明确和详细。最大的区别是`-v`语法在一个字段中将所有选项组合在一起，而`--mount`语法将它们分开。这是每个标志的语法比较。

> 提示：新用户应使用`--mount`语法。有经验的用户可能更熟悉`-v`或`--volume`语法，但建议使用`--mount`，因为研究表明它易于使用。

- `-v`或`--volume`：由三个字段组成，以冒号(:)分隔。这些字段必须以正确的顺序排列，并且每个字段的含义不是立即显而易见的。
	- 对于绑定挂载，第一个字段是主机上文件或目录的路径。
	- 第二个字段是文件或目录在容器中的挂载路径。
	- 第三个字段是可选的，并且是用逗号分隔的选项列表，例如`ro`，`consistent`，`delegated`，`cached`，`z`和`Z`。下面讨论这些选项。

- `--mount`：由多个键值对组成，这些对值之间用逗号分隔，每个键对均由一个`<key>=<value>`元组组成。 `--mount`语法比`-v`或`--volume`更为冗长，但是键的顺序并不重要，并且标志的值更易于理解。
	- 挂载的`type`，可以是`bind`，`volume`或`tmpfs`。本主题讨论挂载绑定，因此类型始终为`bind`。
	- 挂载的`source`。对于挂载绑定，这是`Docker`守护程序主机上文件或目录的路径。可以指定为`source`或`src`。
	- `destination`将文件或目录在容器中的挂载路径作为其值。可以指定为`destination`，`dst`或`target`。
	- `readonly`选项（如果存在）使挂载绑定以只读方式挂载到容器中。
	- `bind-propagation`选项（如果存在）将更改绑定传播。可以是`rprivate`，`private`，`rshared`，`shared`，`rslave`，`slave`之一。
	- 如果存在`consistency`选项，则可以是`consistent`，`delegated`或`cached`之一。此设置仅适用于`Mac`的`Docker`桌面，在所有其他平台上将被忽略。
	- `--mount`标志不支持用于修改`selinux`标签的`z`或`Z`选项。

下面的示例在可能的情况下同时显示`--mount`和`-v`语法，并且首先显示`--mount`。

### `-v`和`--mount`行为之间的区别
由于`-v`和`--volume`标志很长一段时间已成为`Docker`的一部分，因此它们的行为无法更改。这意味着`-v`和`--mount`之间存在一种不同的行为。

如果使用`-v`或`--volume`绑定挂载`Docker`主机上尚不存在的文件或目录，则`-v`为您创建端点。始终将其创建为目录。


如果使用`--mount`绑定挂载`Docker`主机上尚不存在的文件或目录，则`Docker`不会自动为您创建文件或目录，但会生成错误。

## 使用挂载绑定启动容器
考虑以下情况：您拥有目录源，并且在构建源代码时，工件会保存到另一个目录`source/target/`中。 您希望工件可以在`/app/`下对容器可用，并且您希望每次在开发主机上构建源代码时，容器都可以访问新的构建。 使用以下命令将目标目录`/`目录绑定挂载到您的容器中，位于`/app/`。 从源目录中运行命令。 `$(wd)`子命令将扩展到`Linux`或`macOS`主机上的当前工作目录。

下面的`--mount`和`-v`示例产生相同的结果。 除非在运行第一个容器后删除`devtest`容器，否则无法同时运行它们。
如果使用`--mount`绑定挂载`Docker`主机上尚不存在的文件或目录，则`Docker`不会自动为您创建文件或目录，但会生成错误。

```shell
$ docker run -d \
  -it \
  --name devtest \
  --mount type=bind,source="$(pwd)"/target,target=/app \
  nginx:latest
```
```shell
$ docker run -d \
  -it \
  --name devtest \
  -v "$(pwd)"/target:/app \
  nginx:latest
```
使用`docker inspect devtest`来验证绑定绑定是否正确创建。 查找"Mounts"部分：
```shell
"Mounts": [
    {
        "Type": "bind",
        "Source": "/tmp/source/target",
        "Destination": "/app",
        "Mode": "",
        "RW": true,
        "Propagation": "rprivate"
    }
],
```
这表明挂载是绑定挂载，显示正确的源和目标，表明挂载是可读写的，并且传播设置为`rprivate`。

停止容器：
```shell
$ docker container stop devtest

$ docker container rm devtest
```
### 挂载到容器上的非空目录
如果您将绑定挂载到容器上的非空目录中，则该目录的现有内容会被绑定挂载遮盖。 这可能是有益的，例如，当您要测试应用程序的新版本而不构建新镜像时。 但是，这也可能令人惊讶，并且此行为不同于`Docker`卷的行为。

本示例被认为是极端的，但是用主机上的`/tmp/`目录替换了容器的`/usr/`目录的内容。 在大多数情况下，这将导致容器无法正常工作。

`--mount`和`-v`示例具有相同的最终结果。
```shell
$ docker run -d \
  -it \
  --name broken-container \
  --mount type=bind,source=/tmp,target=/usr \
  nginx:latest

docker: Error response from daemon: oci runtime error: container_linux.go:262:
starting container process caused "exec: \"nginx\": executable file not found in $PATH".
```
```shell
$ docker run -d \
  -it \
  --name broken-container \
  -v /tmp:/usr \
  nginx:latest

docker: Error response from daemon: oci runtime error: container_linux.go:262:
starting container process caused "exec: \"nginx\": executable file not found in $PATH".
```
容器创建了但没有启动，删除它
```shell
$ docker container rm broken-container
```

## 使用只读绑定挂载
对于某些开发应用程序，容器需要写入绑定挂载，因此更改将传播回`Docker`主机。 在其他时间，容器仅需要读取访问权限。

本示例修改了上面的示例，但通过在容器中的挂载点之后将`ro`添加到（默认为空）选项列表中，将目录作为只读绑定进行挂载。 如果存在多个选项，请用逗号分隔。

`--mount`和`-v`示例具有相同的结果。
```shell
$ docker run -d \
  -it \
  --name devtest \
  --mount type=bind,source="$(pwd)"/target,target=/app,readonly \
  nginx:latest
```
```shell
$ docker run -d \
  -it \
  --name devtest \
  -v "$(pwd)"/target:/app:ro \
  nginx:latest
```
使用`docker inspect devtest`来验证绑定挂载是否正确创建。 查找"Mounts"部分：
```
"Mounts": [
    {
        "Type": "bind",
        "Source": "/tmp/source/target",
        "Destination": "/app",
        "Mode": "ro",
        "RW": false,
        "Propagation": "rprivate"
    }
],
```
停止容器
```
$ docker container stop devtest

$ docker container rm devtest
```

## 配置绑定传播
对于绑定挂载和卷，绑定传播默认为rprivate。 它仅可用于绑定挂载，并且只能在`Linux`主机上配置。 绑定传播是一个高级主题，许多用户无需配置它。

绑定传播是指是否可以将在给定绑定挂载或命名卷中创建的挂载传播到该挂载的副本。 考虑一个挂载点`/mnt`，它也挂载在`/tmp`上。 传播设置控制`/ tmp/a`上的挂载是否也可以在`/mnt/a`上使用。 每个传播设置都有一个递归对点。 在递归的情况下，请考虑将`/tmp/a`也挂载为`/fo`。 传播设置控制`/ mnt/a`和`/`或`/tmp/a`是否存在。

| 传播设置 | 描述 |
| --- | --- |
| `shared` | 原始挂载的子挂载暴露于副本挂载，副本挂载的子挂载也传播到原始挂载。 |
| `slave` | 与共享挂载类似，但仅在一个方向上。 如果原始挂载公开了子挂载，则副本挂载可以看到它。 但是，如果副本挂载公开了一个子挂载，则原始挂载看不到它。 |
| `private` | 挂载是私人的。 其中的子挂载不暴露于副本挂载，副本挂载的子挂载也不暴露于原始挂载。 |
| `rshared` | 与共享相同，但传播也扩展到嵌套在任何原始或副本挂载点中的挂载点以及从这些挂载点扩展。 |
| `rslave` | 与从属服务器相同，但是传播也扩展到嵌套在任何原始或副本挂载点中的挂载点，以及从这些挂载点扩展。 |
| `rprivate` | 默认值。 与专用相同，这意味着原始或副本挂载点中的任何位置都不会在任何方向上传播挂载点。 |

在可以在挂载点上设置绑定传播之前，主机文件系统需要已经支持绑定传播。

有关绑定传播的更多信息，请参见[`Linux内核`文档](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt)中的共享子树。

以下示例将`target/`目录两次装入容器，第二次装入同时设置`ro`选项和`rslave`绑定传播选项。

`--mount`和`-v`示例具有相同的结果。

```shell
$ docker run -d \
  -it \
  --name devtest \
  --mount type=bind,source="$(pwd)"/target,target=/app \
  --mount type=bind,source="$(pwd)"/target,target=/app2,readonly,bind-propagation=rslave \
  nginx:latest
```
```shell
$ docker run -d \
  -it \
  --name devtest \
  -v "$(pwd)"/target:/app \
  -v "$(pwd)"/target:/app2:ro,rslave \
  nginx:latest
```

现在，如果您创建`/app/foo/`，则`/app2/foo/`也存在。

## 配置`selinux`标签
如果使用`selinux`，则可以添加`z`或`Z`选项来修改要装入容器的主机文件或目录的`selinux`标签。这会影响主机本身上的文件或目录，并可能导致超出`Docker`范围的后果。

- `z`选项指示绑定挂载内容在多个容器之间共享。
- `Z`选项表示绑定挂载内容是私有的且未共享。

这些选项请格外小心。使用`Z`选项绑定安装系统目录（例如`/home`或`/usr`）会使主机无法操作，并且您可能需要手动重新标记主机文件。

> 重要说明：将绑定挂载与服务一起使用时，将忽略`selinux`标签（`:Z`和`:z`）以及`:ro`。有关详细信息，请参见[`moby/moby＃32579`](https://github.com/moby/moby/issues/32579)。

本示例设置`z`选项以指定多个容器可以共享绑定挂载的内容：

不能使用`--mount`标志来修改`selinux`标签。

```shell
$ docker run -d \
  -it \
  --name devtest \
  -v "$(pwd)"/target:/app:z \
  nginx:latest
```

## 配置`macOS`的安装一致性
用于`Mac`的`Docker`桌面使用`osxfs`将从`macOS`共享的目录和文件传播到`Linux VM`。这种传播使这些目录和文件可用于在`Mac`上运行`Docker Desktop`的`Docker`容器。

默认情况下，这些共享是完全一致的，这意味着每次在`macOS`主机上或通过容器中的挂载进行写操作时，所做的更改都会刷新到磁盘上，以便共享中的所有参与者都具有完全一致的视图。在某些情况下，完全一致性可能会严重影响性能。 `Docker 17.05`及更高版本引入了一些选项，可以针对每个容器，每个容器来调整一致性设置。提供以下选项：

- `consistent`或`default`：如上所述，具有完全一致性的默认设置。
- `delegated`：容器运行时的装载视图是权威的。在主机上看到容器中所做的更新之前可能会有所延迟。
- `cached`：`macOS`主机的挂载视图具有权威性。在容器中可以看到主机上进行的更新，这可能会有所延迟。

这些选项在除`macOS`之外的所有主机操作系统上均被完全忽略。

`--mount`和`-v`示例具有相同的结果。

```shell
$ docker run -d \
  -it \
  --name devtest \
  --mount type=bind,source="$(pwd)"/target,destination=/app,consistency=cached \
  nginx:latest
```
```
$ docker run -d \
  -it \
  --name devtest \
  --mount type=bind,source="$(pwd)"/target,destination=/app,consistency=cached \
  nginx:latest
```