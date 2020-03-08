# 管理镜像

使镜像可供组织内部或外部的其他人使用的最简单方法是使用`Docker`注册表，例如`Docker Hub`，`Docker Trusted Registry`，或运行自己的私有注册表。

## `Docker`中心
[`Docker Hub`](https://docs.docker.com/docker-hub/)是由`Docker`，`Inc.`管理的公共注册表。它集中了有关组织，用户帐户和镜像的信息。它包括一个`Web UI`，使用组织的身份验证和授权，使用诸如`docker`登录，`docker pull`和`docker push`，注释，星号，搜索等命令的`CLI`和API访问。

## `Docker`注册表
`Docker Registry`是`Docker`生态系统的组成部分。注册表是一个存储和内容交付系统，其中包含命名的`Docker`镜像，这些镜像具有不同的标记版本。例如，带有标签`2.0`和最新版本的镜像分发/注册。用户通过使用`docker push`和`pull`命令（例如`docker pull myregistry.com/stevvooe/batman:voice`）与注册表进行交互。

`Docker Hub`是`Docker Registry`的一个实例。

## `Docker`可信注册表
[`Docker Trusted Registry`](https://docs.docker.com/datacenter/dtr/2.1/guides/)是`Docker Enterprise Edition`的一部分，是一个私有的，安全的`Docker Registry`，其中包括镜像签名和内容信任，基于角色的访问控制以及其他企业级功能等功能。

## 内容信任
在网络系统之间传输数据时，信任是一个主要问题。尤其是，当通过不受信任的介质（例如`Internet`）进行通信时，确保系统运行的所有数据的完整性和发布者至关重要。您使用`Docker`将镜像（数据）推入和拉入注册表。内容信任使您能够验证通过任何渠道从注册表接收的所有数据的完整性和发布者。

有关在`Docker`客户端上配置和使用此功能的信息，请参阅内容信任。