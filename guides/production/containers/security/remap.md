# 使用用户名称空间隔离容器

`Linux`名称空间为运行中的进程提供了隔离，从而限制了它们对系统资源的访问，而运行中的进程没有意识到这些限制。有关`Linux`名称空间的更多信息，请参见 [`Linux`名称空间](https://www.linux.com/news/understanding-and-securing-linux-namespaces)。

防止来自容器内部的特权升级攻击的最佳方法是将容器的应用程序配置为以非特权用户身份运行。对于其进程必须以该`root`容器内的用户身份运行的容器，您可以将该用户重新映射到`Docker`主机上的特权较低的用户。已为映射的用户分配了一系列`UID`，这些`UID`在名称空间中的功能与普通`UID`相同，范围为`0`到`65536`，但对主机本身没有特权。

## 关于重新映射和从属用户和组
重新映射本身由两个文件处理：`/etc/subuid`和`/etc/subgid`。每个文件的工作原理相同，但一个文件与用户`ID`范围有关，另一个文件与组`ID`范围有关。考虑以下内容`/etc/subuid`：
```shell
testuser:231072:65536
```
这意味着将 依次`testuser`分配一个下级用户`ID`范围`231072`和下一个`65536`个整数。`UID 231072`作为`UID 0（root）`在名称空间内（在这种情况下，在容器内）映射。`UID 231073` 映射为`UID 1`，依此类推。如果某个进程尝试将特权升级到命名空间之外，则该进程将作为主机上的非特权高编号`UID`运行，甚至不会映射到真实用户。这意味着该进程在主机系统上完全没有特权。

> 多个范围
> 通过在`/etc/subuidor` `/etc/subgid`文件中为同一用户或组添加多个不重叠的映射，可以为给定的用户或组分配多个下属范围 。在这种情况下，`Docker`只使用前五个映射，按照只有五个条目内核的限制`/proc/self/uid_map`和`/proc/self/gid_map`。

在将`Docker`配置为使用`userns-remap`功能时，可以选择指定现有用户和/或组，也可以指定`default`。如果指定`default`，`dockremap`则会创建一个用户和组并将其用于此目的。

> 警告：某些发行版，例如`RHEL`和`CentOS 7.3`，不会自动将新组添加到`/etc/subuid`和`/etc/subgid`文件中。在这种情况下，您负责编辑这些文件并分配不重叠的范围。先决条件中介绍了此步骤。

范围不重叠是非常重要的，以使进程无法在其他名称空间中获得访问权限。在大多数`Linux`发行版中，添加或删除用户时，系统实用程序都会为您管理范围。

这种重新映射对容器是透明的，但是在容器需要访问`Docker`主机上的资源的情况下（例如，将绑定绑定绑定到系统用户无法写入的文件系统区域中），会带来一些配置复杂性。从安全角度来看，最好避免这些情况。

## 前提条件
1. 从属`UID`和`GID`范围必须与现有用户相关联，即使该关联是实现细节。用户在下拥有命名空间的存储目录`/var/lib/docker/。`如果您不想使用现有用户，`Docker`可以为您创建一个并使用它。如果要使用现有的用户名或用户`ID`，则它必须已经存在。通常，这意味着相关条目需要位于 `/etc/passwd`和中`/etc/group`，但是如果您使用其他身份验证后端，则此要求可能会有所不同。

要验证这一点，请使用以下`id`命令：
```shell
$ id testuser

uid=1001(testuser) gid=1001(testuser) groups=1001(testuser)
```

2. 在主机上处理名称空间重新映射的方式是使用两个文件 `/etc/subuid`和`/etc/subgid`。这些文件通常在添加或删除用户或组时自动进行管理，但是在某些发行版（例如`RHEL`和`CentOS 7.3`）上，可能需要手动管理这些文件。

每个文件包含三个字段：用户的用户名或`ID`，后跟开头的`UID`或`GID`（在名称空间中被视为`UID`或`GID` 0）以及用户可用的最大`UID`或`GID`。例如，给定以下条目：
```shell
testuser:231072:65536
```
这意味着由开头的用户命名空间进程`testuser`由主机`UID 231072`（看起来像`0`名称空间中的`UID` ）通过`296607（231072 + 65536-1）`拥有。这些范围不应重叠，以确保命名空间进程不能访问彼此的命名空间。

添加用户后，检查`/etc/subuid`并`/etc/subgid`查看您的用户是否在每个条目中都有一个条目。如果不是，则需要添加它，请注意避免重叠。

3. 如果要使用`dockremap`由`Docker`自动创建的用户，请在 配置和重新启动`Docker` 之后检查`dockremap`这些文件中的条目。

如果`Docker`主机上有任何特权用户需要写入的位置，请相应地调整这些位置的权限。如果要使用`dockremap`由`Docker`自动创建的用户，也是如此，但是只有在配置并重新启动`Docker`之后才能修改权限。

4. `userns-remap`有效启用将掩盖现有的镜像和容器层以及中的其他`Docker`对象`/var/lib/docker/`。这是因为`Docker`需要调整这些资源的所有权并将其实际存储在的子目录中`/var/lib/docker/`。最好在新的`Docker`安装上而不是现有的`Docker`上启用此功能。

同样，如果禁用`userns-remap`，则无法访问在启用该资源时创建的任何资源。

5. 检查用户名称空间的限制，以确保可以使用。

## 在守护程序上启用`userns-remap` 
您可以从标志开始`dockerd`，也可以`--userns-remap`按照以下过程使用`daemon.json`配置文件配置守护程序。`daemon.json`建议使用该方法。如果使用标志，请使用以下命令作为模型：
```shell
$ dockerd --userns-remap="testuser:testuser"
```
1. 编辑`/etc/docker/daemon.json`。假设该文件先前为空，则以下条目允许`userns-remap`使用名为的用户和组 `testuser`。您可以按`ID`或名称寻址用户和组。如果组名或`ID`与用户名或`ID`不同，则只需要指定即可。如果同时提供用户名和组名或`ID`，请用冒号（`:`）分隔。以下格式的值的所有作品，假定的`UID`和`GID testuser`是`1001`：

- `testuser`
- `testuser:testuser`
- `1001`
- `1001:1001`
- `testuser:1001`
- `1001:testuser`

```json
{
  "userns-remap": "testuser"
}
```

> 注意：要使用`dockremap`用户并让`Docker`为您创建用户，请将值设置为`default`而不是`testuser`。

保存文件并重新启动`Docker`。

2. 如果使用的是`dockremap`用户，请验证`Docker`是否使用`id`命令创建了该用户。
```shell
$ id dockremap

uid=112(dockremap) gid=116(dockremap) groups=116(dockremap)
```
验证条目已添加到`/etc/subuid和/etc/subgid`：
```shell
$ grep dockremap /etc/subuid

dockremap:231072:65536

$ grep dockremap /etc/subgid

dockremap:231072:65536
```
如果不存在这些条目，请以`root`用户身份编辑文件并分配一个起始的`UID`和`GID`，该`ID`是分配的最高的`UID`和`GID`（在本例中为`65536`）。注意不要在范围内重叠。

3. 使用该`docker image ls` 命令验证先前的镜像不可用。输出应为空。

4. 从`hello-world`镜像启动一个容器。
```shell
$ docker run hello-world
```

5. 验证名称空间目录中是否存在`/var/lib/docker/`以该名称空间用户的`UID`和`GID` 命名的目录，该目录由该`UID`和`GID`拥有，并且不是组或世界可读的。某些子目录仍归所有者所有，`root`并具有不同的权限。
```shell
$ sudo ls -ld /var/lib/docker/231072.231072/

drwx------ 11 231072 231072 11 Jun 21 21:19 /var/lib/docker/231072.231072/

$ sudo ls -l /var/lib/docker/231072.231072/

total 14
drwx------ 5 231072 231072 5 Jun 21 21:19 aufs
drwx------ 3 231072 231072 3 Jun 21 21:21 containers
drwx------ 3 root   root   3 Jun 21 21:19 image
drwxr-x--- 3 root   root   3 Jun 21 21:19 network
drwx------ 4 root   root   4 Jun 21 21:19 plugins
drwx------ 2 root   root   2 Jun 21 21:19 swarm
drwx------ 2 231072 231072 2 Jun 21 21:21 tmp
drwx------ 2 root   root   2 Jun 21 21:19 trust
drwx------ 2 231072 231072 3 Jun 21 21:19 volumes
```
您的目录列表可能会有一些差异，特别是如果您使用的容器存储驱动程序不同于`aufs`。

使用由重新映射的用户拥有的目录，而不是直接位于其下方的相同目录，`/var/lib/docker/`并且`/var/lib/docker/tmp/`可以删除未使用的版本（例如此处的示例）。`userns-remap`启用后，`Docker`不会使用它们。

## 禁用命名空间重新映射为一个容器
如果在守护程序上启用用户名称空间，则所有容器都将在默认情况下启用用户名称空间的情况下启动。在某些情况下，例如特权容器，您可能需要禁用特定容器的用户名称空间。有关 这些限制中的某些限制，请参见 用户名称空间已知限制。

为了禁用用户命名空间特定容器中，添加`--userns=host` 标志到`docker container create`，`docker container run`或`docker container exec`命令。

使用此标志时会产生副作用：不会对该容器启用用户重新映射，但是由于容器之间共享只读（镜像）层，因此仍将重新映射容器文件系统的所有权。

这意味着整个容器文件系统将属于`--userns-remap`守护程序配置中指定的用户（`231072`在以上示例中）。这可能导致容器内程序的意外行为。例如`sudo`（检查其二进制文件是否属于`user 0`）或带有`setuid`标志的二进制文件。

## 用户命名空间已知限制
以下标准`Docker`功能与运行启用了用户名称空间的`Docker`守护程序不兼容：

- 与主机（`--pid=host`或`--network=host`）共享`PID`或`NET`名称空间。
- 外部（卷或存储）驱动程序，这些驱动程序不知道或无法使用守护程序用户映射。
- `--privileged`在`docker run`不指定的情况下 使用`mode`标志`--userns=host`。

用户名称空间是一项高级功能，需要与其他功能配合。例如，如果从主机装载了卷，则必须预先安排文件所有权，需要对卷内容的读取或写入访问权限。

尽管用用户命名空间的容器进程中的`root`用户具有容器中超级用户的许多预期特权，但是`Linux`内核基于内部知识（这是一个用用户命名空间的进程）施加了限制。一个值得注意的限制是无法使用该`mknod`命令。由`root`用户运行时，在容器内创建设备的权限被拒绝。