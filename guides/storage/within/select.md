# Docker存储驱动程序

理想情况下，很少有数据写入容器的可写层，并且您使用`Docker`卷来写入数据。但是，某些工作负载要求您能够写入容器的可写层。这是存储驱动程序进入的地方。

`Docker`使用可插拔架构支持几种不同的存储驱动程序。存储驱动程序控制如何在`Docker`主机上存储和管理镜像和容器。

阅读存储驱动程序概述后，下一步就是为您的工作负载选择最佳的存储驱动程序。在做出此决定时，需要考虑三个高层次的因素：

如果内核中支持多个存储驱动程序，则在未明确配置任何存储驱动程序的情况下，`Docker`会优先列出要使用的存储驱动程序，前提是该存储驱动程序满足先决条件。

在大多数情况下，请使用具有最佳整体性能和稳定性的存储驱动程序。

`Docker`支持以下存储驱动程序：

- 对于所有当前支持的`Linux`发行版，`overlay2`是首选的存储驱动程序，不需要任何额外的配置。
- 当在不支持`overlay2`的内核`3.13`的`Ubuntu 14.04`上运行时，`aufs`是`Docker 18.06`及更早版本的首选存储驱动程序。
- `devicemapper`受支持，但在生产环境中需要`direct-lvm`，因为`loopback-lvm`虽然为零配置，但性能却很差。 `devicemapper`是`CentOS`和`RHEL`的推荐存储驱动程序，因为它们的内核版本不支持`overlay2`。但是，当前版本的`CentOS`和`RHEL`现在支持`overlay2`，现在推荐使用该驱动程序。
- 如果`btrfs`和`zfs`存储驱动程序是后备文件系统（安装了`Docker`的主机的文件系统），则使用它们。这些文件系统允许使用高级选项，例如创建“快照”，但需要更多的维护和设置。这些中的每一个都依赖于正确配置的后备文件系统。
- `vfs`存储驱动程序旨在用于测试目的，以及无法使用写时复制文件系统的情况。此存储驱动程序的性能很差，通常不建议在生产中使用。

`Docker`的源代码定义了选择顺序。您可以在[`Docker Engine-Community 19.03`](https://github.com/docker/docker-ce/blob/19.03/components/engine/daemon/graphdriver/driver_linux.go#L50)的源代码上看到订单

如果运行其他版本的`Docker`，则可以使用文件查看器顶部的分支选择器来选择其他分支。

某些存储驱动程序要求您对备份文件系统使用特定格式。如果您有使用特定备份文件系统的外部要求，则可能会限制您的选择。请参阅支持的后备文件系统。

在确定了可以选择的存储驱动程序之后，您的选择取决于工作负载的特征和所需的稳定性。请参阅其他注意事项，以帮助您做出最终决定。

> 注意：您的选择可能会受到`Docker`版本，操作系统和发行版的限制。例如，`aufs`仅在`Ubuntu`和`Debian`上受支持，并且可能需要安装额外的软件包，而`btrfs`仅在`SLES`上受支持，`SLES`仅受`Docker Enterprise`支持。有关更多信息，请参见按`Linux`发行版支持存储驱动程序。

## 每个Linux发行版支持的存储驱动程序
在较高级别上，可以使用的存储驱动程序部分取决于所使用的`Docker`版本。

另外，`Docker`不建议您进行任何禁用操作系统安全功能的配置，例如，如果您在`CentOS`上使用`overlay`或`overlay2`驱动程序，则需要禁用`selinux`。

### `Docker Engine-企业版`和`Docker企业版`
对于`Docker Engine-Enterprise`和`Docker Enterprise`，支持存储驱动程序的权威资源是[产品兼容性列表](https://success.docker.com/article/compatibility-matrix)。 要从`Docker`获得商业支持，您必须使用受支持的配置。

### `Docker Engine-社区`
对于`Docker Engine-社区`，仅测试了一些配置，并且您操作系统的内核可能不支持每个存储驱动程序。 通常，以下配置适用于最新版本的`Linux`发行版：

| 发行版本 | 推荐驱动 | 备选驱动 |
| --- | --- | --- |
| `Docker Engine - Community on Ubuntu` | `overlay2` or `aufs` (Ubuntu 14.04 running on kernel 3.13) | `overlay¹`, `devicemapper²`, `zfs`, `vfs` |
| `Docker Engine - Community on Debian` | `overlay2` (Debian Stretch), `aufs` or `devicemapper` (老版本) | `overlay¹`, `vfs` |
| `Docker Engine - Community on CentOs` | `overlay2` | `overlay¹`, `devicemapper²`, `zfs`, `vfs` |
| `Docker Engine - Community on Fedora` | `overlay2` | `overlay¹`, `devicemapper²`, `zfs`, `vfs` | 

¹）叠加层存储驱动程序已在`Docker Engine-Enterprise 18.09中`弃用，并将在以后的版本中删除。建议覆盖存储驱动程序的用户迁移到`overlay2`。

²）`devicemapper`存储驱动程序已在`Docker Engine 18.09`中弃用，并将在以后的版本中删除。建议将`devicemapper`存储驱动程序的用户迁移到`overlay2`。

可能的话，`overlay2`是推荐的存储驱动程序。首次安装`Docker`时，默认使用`overlay2`。以前，默认情况下会使用`aufs`（如果有），但现在情况不再如此。如果要在以后的新安装中使用`aufs`，则需要显式配置它，并且可能需要安装其他软件包，例如`linux-image-extra`。参见`aufs`。

在使用`aufs`的现有安装中，仍会使用它。

如有疑问，最好的全方位配置是使用具有支持`overlay2`存储驱动程序的内核的现代`Linux`发行版，并使用`Docker`卷处理繁重的工作负载，而不是依赖于将数据写入容器的可写层。

`vfs`存储驱动程序通常不是最佳选择。在使用`vfs`存储驱动程序之前，请务必阅读其性能，存储特性和限制。

> 对不推荐的存储驱动程序的期望：`Docker Engine-Community`不提供商业支持，您可以从技术上使用适用于您平台的任何存储驱动程序。例如，即使不建议在`Docker Engine-Community`的任何平台上使用`btrfs`，也可以在`Docker Engine-Community`中使用`btrfs`，但后果自负。

> 上表中的建议基于自动回归测试和已知适用于大量用户的配置。如果使用推荐的配置并发现可重现的问题，则很可能会很快解决该问题。如果根据该表不推荐您使用的驱动程序，则需要您自担风险。您可以并且仍然应该报告遇到的任何问题。但是，此类问题的优先级比使用推荐配置时遇到的问题低。

### `Mac`的`Docker`桌面和`Windows`的`Docker`桌面
`Mac`的`Docker`桌面和`Windows`的`Docker`桌面仅用于开发，而不用于生产。无法在这些平台上修改存储驱动程序。

## 支持的后备文件系统
对于`Docker`，支持文件系统是`/var/lib/docker/`所在的文件系统。 一些存储驱动程序仅适用于特定的后备文件系统。

| 驱动程序 | 文件系统 | 
| --- | --- |
| `overlay2, overlay` | `xfs` with ftype=1, `ext4` | 
| `aufs` | `xfs`,`ext4` |
| `devicemapper` | `direct-lvm` |
| `btrfs` | `btrfs` |
| `zfs` | `zfs` |
| `vfs` | 任意文件系统 |

## 其他注意事项
### 适合您的工作量
除其他事项外，每个存储驱动程序都有其自己的性能特征，这使其或多或少地适合于不同的工作负载。考虑以下概括：

- `overlay2`，`aufs`和`overlay`全部在文件级别而不是块级别运行。这样可以更有效地使用内存，但是在写繁重的工作负载中，容器的可写层可能会变得很大。
- 块级存储驱动程序（例如`devicemapper`，`btrfs`和`zfs`）在需要大量写入的工作负载（尽管不如`Docker`卷）上表现更好。
- 对于许多具有多层或深文件系统的小型写操作或容器，`overlay`的性能可能优于`overlay2`，但是会消耗更多的`inode`，这可能导致inode耗尽。
- `btrfs`和`zfs`需要大量内存。
- 对于诸如`PaaS`之类的高密度工作负载，`zfs`是一个不错的选择。

每个存储驱动程序的文档中提供了有关性能，适用性和最佳实践的更多信息。

### 共享存储系统和存储驱动程序
如果您的企业使用`SAN`，`NAS`，硬件`RAID`或其他共享存储系统，则它们可以提供高可用性，增强的性能，自动精简配置，重复数据删除和压缩。在许多情况下，`Docker`可以在这些存储系统上运行，但是`Docker`并未与其紧密集成。

每个`Docker`存储驱动程序均基于`Linux`文件系统或卷管理器。确保在共享存储系统之上遵循用于操作存储驱动程序（文件系统或卷管理器）的现有最佳实践。例如，如果在共享存储系统上使用`ZFS`存储驱动程序，请确保遵循在该特定共享存储系统上操作`ZFS`文件系统的最佳实践。

### 稳定性
对于某些用户而言，稳定性比性能更重要。尽管`Docker`认为这里提到的所有存储驱动程序都是稳定的，但其中一些是较新的并且仍在积极开发中。通常，`overlay2`，`aufs`，`overlay`和`devicemapper`是具有最高稳定性的选择。

### 用您自己的工作负载进行测试
在不同的存储驱动程序上运行自己的工作负载时，您可以测试`Docker`的性能。确保使用等效的硬件和工作负载来匹配生产条件，以便您可以看到哪个存储驱动程序提供最佳的整体性能。

## 检查您当前的存储驱动程序
每个单独的存储驱动程序的详细文档详细说明了使用给定存储驱动程序的所有设置步骤。

要查看`Docker`当前正在使用什么存储驱动程序，请使用`docker info`并查找`Storage Driver`行：

```shell
$ docker info

Containers: 0
Images: 0
Storage Driver: overlay2
 Backing Filesystem: xfs
<output truncated>
```

要更改存储驱动程序，请参阅新存储驱动程序的特定说明。一些驱动程序需要其他配置，包括对`Docker`主机上的物理或逻辑磁盘的配置。

> 重要信息：更改存储驱动程序时，所有现有的镜像和容器都将无法访问。这是因为新存储驱动程序无法使用其层。如果还原更改，则可以再次访问旧的镜像和容器，但是使用新驱动程序提取或创建的任何镜像和容器将无法访问。