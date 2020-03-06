# 创建基础镜像

大多数`Dockerfil`e从父镜像开始。如果需要完全控制镜像的内容，则可能需要创建基础镜像。区别在于：

- 父镜像是镜像所基于的镜像。它引用`Dockerfile`中`FROM`指令的内容。 `Dockerfile`中的每个后续声明都会修改此父镜像。大多数`Dockerfile`从父镜像而不是基础镜像开始。但是，这些术语有时可以互换使用。

- 基本镜像在其`Dockerfile`中具有从头开始。

本主题向您展示了创建基本镜像的几种方法。具体过程将在很大程度上取决于您要打包的Linux发行版。我们在下面提供了一些示例，建议您提交请求请求以贡献新请求。

## 使用`tar`创建完整镜像
通常，从一台运行您要打包为父镜像打包的发行版的工作计算机开始，尽管某些工具（例如`Debian`的`Debootstrap`）并不需要，您也可以使用这些工具来构建`Ubuntu`镜像。

创建`Ubuntu`父镜像可以很简单：

```shell
$ sudo debootstrap xenial xenial > /dev/null
$ sudo tar -C xenial -c . | docker import - xenial

a29c15f1bf7a

$ docker run xenial cat /etc/lsb-release

DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04 LTS"
```

在`Docker GitHub Repo`中还有更多用于创建父镜像的示例脚本：

- `BusyBox`]
- `Debian/Ubuntu`或`CentOS/RHEL/SLC/etc`上的`CentOS/Science Linux CERN（SLC）`。
- `Debian/Ubuntu`

## 从头开始创建简单的父镜像
您可以使用`Docker`保留的最小镜像刮痕作为构建容器的起点。 使用临时的“镜像”向构建过程发出信号，表示您希望`Dockerfile`中的下一个命令成为镜像中的第一个文件系统层。

当`scratch`出现在集线器上`Docker`的存储库中时，您将无法对其进行拉取，运行或使用`scratch`名称标记任何镜像。 相反，您可以在`Dockerfile`中引用它。 例如，使用`scratch`创建一个最小的容器：
```Dockerfile
FROM scratch
ADD hello /
CMD ["/hello"]
```

假设您按照`https://github.com/docker-library/hello-world/`上的说明构建了"hello"可执行示例，并使用`-static`标志对其进行了编译，则可以使用此`docker`来构建此`Docker`镜像。 构建命令：
```shell
docker build --tag hello .
```
别忘了`.` 最后一个字符，它将构建上下文设置为当前目录。

> 注意：由于适用于`Mac`的`Docker`桌面和适用于`Windows`的`Docker`桌面使用`Linux VM`，因此您需要`Linux二`进制文件，而不是`Mac`或`Windows`二进制文件。 您可以使用`Docker`容器来构建它：
```shell
$ docker run --rm -it -v $PWD:/build ubuntu:16.04

container# apt-get update && apt-get install build-essential
container# cd /build
container# gcc -o hello -static -nostartfiles hello.
```
为了运行新镜像，使用`docker run`命令
```shell
docker run --rm hello
```
本示例创建了教程中使用的`hello-world`镜像。 如果要对其进行测试，则可以克隆镜像存储库。

## 更多资源
有许多可用资源可帮助您编写`Dockerfile`。

- 在参考部分中，有一份[完整的指南](https://docs.docker.com/engine/reference/builder/)，可指导您在`Dockerfile`中使用的所有说明。
- 为了帮助您编写清晰，可读，可维护的`Dockerfile`，我们还编写了[`Dockerfile`最佳做法指南](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)。
- 如果您的目标是创建新的官方镜像，请务必阅读[`Docker`的官方镜像](https://docs.docker.com/docker-hub/official_images/)。