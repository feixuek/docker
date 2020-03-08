# 用Docker开发

此页面包含要使用`Docker`构建新应用程序的应用程序开发人员的资源列表。

## 先决条件
在[入门](https://docs.docker.com/get-started/)中的学习模块中进行操作，以了解如何构建镜像并将其作为容器化应用程序运行。

## 在`Docker`上开发新应用
如果您刚刚开始在`Docker`上开发全新的应用程序，请查看这些资源以了解一些最常见的模式，以从`Docker`中获得最大收益。

- 使用[多阶段构建](https://docs.docker.com/develop/develop-images/multistage-build/)来保持镜像精简
- 使用[卷](https://docs.docker.com/storage/volumes/)管理应用程序数据并[绑定挂载](https://docs.docker.com/storage/bind-mounts/)
- 使用[kubernetes](https://docs.docker.com/get-started/kube-deploy/)扩展您的应用
- 将您的应用程序扩展为[`swarm`服务](https://docs.docker.com/get-started/swarm-deploy/)
- [通用应用程序开发最佳实践](https://docs.docker.com/develop/dev-best-practices/)

## 了解如何使用`Docker`开发特定于语言的应用程序
- [`Docker for Java`开发人员](https://github.com/docker/labs/tree/master/developer-tools/java/)实验室
- [将`node.js`应用移植到`Docker`](https://github.com/docker/labs/tree/master/developer-tools/nodejs/porting)
- [`Docker Lab`上的`Ruby on Rails`应用](https://github.com/docker/labs/tree/master/developer-tools/ruby)
- [`Dockerize .Net Core`应用程序](https://docs.docker.com/engine/examples/dotnetcore/)
- 使用`Docker Compose`在[`Linux`上使用`SQL Server Dockerize ASP.NET Core`应用程序](https://docs.docker.com/compose/aspnet-mssql-compose/)

## 使用SDK或API进行高级开发
在您可以编写`Dockerfiles`或`Compose`文件并使用`Docker CLI`之后，通过使用适用于`Go/Python`的`Docker Engine SDK`或直接使用`HTTP API`将其提升到一个新的水平。