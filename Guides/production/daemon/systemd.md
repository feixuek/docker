# 用`systemd`控制`Docker`

许多`Linux`发行版使用`systemd`启动`Docker`守护程序。本文档显示了一些有关如何自定义`Docker`设置的示例。

## 启动`Docker`守护进程
### 手动开始
安装`Docker`之后，您需要启动`Docker`守护程序。大多数`Linux`发行版都`systemctl`用于启动服务。如果没有`systemctl`，请使用`service`命令。

- `systemctl`：
```shell
$ sudo systemctl start docker
```
s- `ervice`：
```shell
$ sudo service docker start
```

### 在系统启动时自动启动
如果您希望`Docker`在启动时启动，请[参阅配置`Docker`在启动时启动](https://docs.docker.com/install/linux/linux-postinstall//#configure-docker-to-start-on-boot)。

## 自定义`Docker`守护程序选项

有多种方法可以为`Docker`守护程序配置守护程序标志和环境变量。推荐的方法是使用与平台无关的 `daemon.json`文件，默认情况下该文件位于`Linux``/etc/docker/`上。请参阅[守护程序配置文件](https://docs.docker.com/engine/reference/commandline/dockerd//#daemon-configuration-file)。

您可以使用来配置几乎所有守护程序配置选项`daemon.json`。以下示例配置两个选项。无法使用`daemon.json`机制配置的一件事是`HTTP`代理。

### 运行时目录和存储驱动程序
您可能需要通过将其移动到单独的分区来控制用于`Docker`镜像，容器和卷的磁盘空间。

为此，请在`daemon.json`文件中设置以下标志：
```json
{
    "data-root": "/mnt/docker-data",
    "storage-driver": "overlay2"
}
```

### `HTTP/HTTP`代理
泊坞窗守护程序使用`HTTP_PROXY`，`HTTPS_PROXY`以及`NO_PROXY`在其启动环境环境变量来配置`HTTP`或`HTTPS`代理的行为。您不能使用该`daemon.json`文件配置这些环境变量。

本示例覆盖默认`docker.service`文件。

如果在公司设置中位于`HTTP`或`HTTPS`代理服务器之后，则需要在`Docker systemd`服务文件中添加此配置。

1. 为`docker`服务创建一个`systemd`插入目录：

```shell
$ sudo mkdir -p /etc/systemd/system/docker.service.d
```

2. 创建一个名为`/etc/systemd/system/docker.service.d/http-proxy.conf` 添加`HTTP_PROXY`环境变量的文件：

```shell
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:80/"
```

或者，如果您位于`HTTPS`代理服务器后面，请创建一个名为的文件，该文件 `/etc/systemd/system/docker.service.d/https-proxy.conf` 会添加`HTTPS_PROXY`环境变量：

```shell
[Service]
Environment="HTTPS_PROXY=https://proxy.example.com:443/"
```

3. 如果您需要在不使用代理的情况下联系内部`Docker`注册表，则可以通过`NO_PROXY`环境变量指定它们。

该`NO_PROXY`变量指定一个字符串，其中包含应以代理方式排除的主机的逗号分隔值。您可以指定以下选项来排除主机：

- `IP`地址前缀（`1.2.3.4`）
- 域名或特殊的`DNS`标签(`*`)
- 域名与该名称和所有子域匹配。以"`.`"开头的域名。仅匹配子域。例如，给定域 `foo.example.com`和`example.com`：
	- `example.com`匹配`example.com`和`foo.example.com`，以及
	- `.example.com` 仅匹配 `foo.example.com`
- 单个星号(`*`)表示不应该进行代理
- `IP`地址前缀（`1.2.3.4:80`）和域名（`foo.example.com:80`）接受文字端口号

配置示例：
```shell
[Service]    
Environment="HTTP_PROXY=http://proxy.example.com:80/" "NO_PROXY=localhost,127.0.0.1,docker-registry.example.com,.corp"
```
或者，如果您位于`HTTPS`代理服务器后面：
```shell
[Service]    
Environment="HTTPS_PROXY=https://proxy.example.com:443/" "NO_PROXY=localhost,127.0.0.1,docker-registry.example.com,.corp"
```

4. 刷新更改：

```shell
$ sudo systemctl daemon-reload
```

5. 重新启动`Docker`：
```shell
$ sudo systemctl restart docker
```

6. 验证配置是否已加载：
```shell
$ systemctl show --property=Environment docker
Environment=HTTP_PROXY=http://proxy.example.com:80/
```
或者，如果您位于`HTTPS`代理服务器后面：
```shell
$ systemctl show --property=Environment docker
Environment=HTTPS_PROXY=https://proxy.example.com:443/
```

## 配置`Docker`守护程序侦听连接的位置
请参阅 [配置`Docker`守护程序在何处侦听连接](https://docs.docker.com/install/linux/linux-postinstall/#control-where-the-docker-daemon-listens-for-connections)。

## 手动创建`systemd``unit`文件
当安装不带软件包的二进制文件时，您可能希望将`Docker`与`systemd`集成。为此，请从`githu`b存储库中将两个单元文件（`service`和`socket`）安装 到`/etc/systemd/system`。