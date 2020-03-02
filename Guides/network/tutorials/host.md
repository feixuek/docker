# 使用主机网络联网

本系列教程介绍了网络独立容器，这些容器直接绑定到`Docker`主机的网络，没有网络隔离。有关其他联网主题，请参见概述。

## 目标
本教程的目标是启动一个直接绑定到`Docker`主机上端口`80`的`nginx`容器。从网络的角度来看，这与`nginx`进程是直接在`Docker`主机而不是在容器中运行的隔离级别相同。但是，以其他所有方式，例如存储，进程名称空间和用户名称空间，`nginx`进程与主机隔离。

## 先决条件
- 此过程要求端口`80`在`Docker`主机上可用。要使`Nginx`在其他端口上侦听，请参阅有关[`Nginx`镜像](https://hub.docker.com/_/nginx/)的文档

- 主机网络驱动程序仅在`Linux`主机上工作，而`Mac`的`Docker`桌面，`Windows`的`Docker`桌面或`Windows Server`的`Docker EE`不支持该主机网络驱动程序。

## 程序
1. 创建并启动容器作为一个独立的过程。 `--rm`选项意味着一旦容器退出/停止，就将其删除。 `-d`标志意味着启动分离的容器（在后台）。
```shell
docker run --rm -d --network host --name my_nginx nginx
```
2. 通过浏览到`http://localhost:80/`访问`Nginx`。

3. 使用以下命令检查您的网络堆栈：
	- 检查所有网络接口，并确认未创建新的网络接口。
	```shell
	ip addr show
	```
	- 使用`netstat`命令验证哪个进程绑定到端口`80`。 您需要使用`sudo`，因为该进程归`Docker`守护程序用户所有，否则您将无法看到其名称或`PID`。
	```shell
	sudo netstat -tulpn | grep :80
	```

4. 停止容器。 使用`--rm`选项启动时，它将自动将其删除。
```shell
docker container stop my_nginx
```

## 其他网络教程
既然您已经完成了独立容器的网络教程，那么您可能需要运行以下其他网络教程：

- [桥接网络教程](https://docs.docker.com/network/network-tutorial-standalone/)
- [叠加网络教程](https://docs.docker.com/network/network-tutorial-overlay/)
- [Macvlan网络教程](https://docs.docker.com/network/network-tutorial-macvlan/)