# 第一部分：安装

欢迎！ 我们很高兴您想学习`Docker`。 `Docker`快速入门培训模块教您如何：

1. 设置您的`Docker`环境（在此页面上）

2. 生成并运行镜像

3. 在`Docker Hub`上共享镜像

## Docker概念
`Docker`是供开发人员和系统管理员使用容器构建，运行和共享应用程序的平台。使用容器部署应用程序称为容器化。容器并不是新事物，但用于轻松部署应用程序的容器却是新事物。

容器化越来越受欢迎，因为容器是：

- 灵活：即使最复杂的应用程序也可以容器化。
- 轻量级：容器利用并共享了主机内核，在系统资源方面比虚拟机更加有效。
- 可移植：您可以在本地构建，部署到云并在任何地方运行。
- 松散耦合：容器是高度自给自足并封装的容器，使您可以在不破坏其他容器的情况下更换或升级它们。
- 可扩展：您可以在数据中心内增加并自动分布容器副本。
- 安全：容器将积极的约束和隔离应用于流程，而无需用户方面的任何配置。

### 镜像和容器
从根本上讲，一个容器不过是一个正在运行的进程，并对其应用了一些附加的封装功能，以使其与主机和其他容器隔离。容器隔离的最重要方面之一是每个容器都与自己的专用文件系统进行交互。该文件系统由`Docker`镜像提供。镜像包括运行应用程序所需的一切-代码或二进制文件，运行时，依赖项以及所需的任何其他文件系统对象。

### 容器和虚拟机

容器在`Linux`上本地运行，并与其他容器共享主机的内核。它运行一个离散进程，不占用任何其他可执行文件更多的内存，从而使其轻巧。

相比之下，虚拟机（`VM`）运行具有"虚拟机管理程序"对主机资源的虚拟访问权的成熟"来宾:操作系统。通常，`VM`会产生大量开销，超出了应用程序逻辑所消耗的开销。

![container](images/Container@2x.png)

![vm](VM@2x.png)

## 设置自己的Docker环境
### 下载并安装Docker
`Docker Desktop`是适用于`Mac`或`Windows`环境的易于安装的应用程序，可让您在几分钟内开始编码和容器化。 `Docker Desktop`包含直接从您的机器构建，运行和共享容器化应用程序所需的一切。

按照适用于您的操作系统的说明下载并安装`Docker Desktop`：

[Mac](https://docs.docker.com/docker-for-mac/install/)
[Windows](https://docs.docker.com/docker-for-windows/install/)

### 测试Docker版本
成功安装`Docker Desktop`之后，打开终端并运行`docker --version`来检查计算机上安装的`Docker`的版本。
```shell
$ docker --version
Docker version 19.03.5, build 633a0ea
```

### 测试Docker安装
1. 通过运行`hello-world`镜像来测试`Docker`安装
```shell
    $ docker run hello-world

    Unable to find image 'hello-world:latest' locally
    latest: Pulling from library/hello-world
    ca4f61b1923c: Pull complete
    Digest: sha256:ca0eeb6fb05351dfc8759c20733c91def84cb8007aa89a5bf606bc8b315b9fc7
    Status: Downloaded newer image for hello-world:latest

    Hello from Docker!
    This message shows that your installation appears to be working correctly.
```

2. 运行`docker image ls`列出您下载到计算机上的`hello-world`镜像。

3. 列出显示消息后退出的`hello-world`容器（由镜像生成）。 如果它仍在运行，则不需要--all选项：
```shell
运行docker image ls列出您下载到计算机上的hello-world映像。

列出显示消息后退出的hello-world容器（由图像生成）。 如果它仍在运行，则不需要--all选项：
```

## 结论

至此，您已经在开发机器上安装了`Docker Desktop`，并进行了快速测试，以确保您已设置为构建和运行第一个容器化应用程序。


# 第二部分：构建并运行自己的镜像

