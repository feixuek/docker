# CentOS 安装 Docker CE

## 前提条件

### 操作系统要求

要安装`Docker Engine-Community`，您需要一个`CentOS 7`的维护版本。不支持或未测试存档版本。

该`centos-extras`库必须启用。默认情况下，此存储库是启用的，但是如果已禁用它，则需要 重新启用它。

建议使用`overlay2`存储驱动程序。

### 卸载旧版本

旧版本的 Docker 称为 `docker` 或者 `docker-engine`，如果已安装这些程序，请卸载它们以及相关的依赖项。本：

```bash
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```
如果`yum`报告未安装这些软件包，则可以。

`/var/lib/docker/`下的内容，包括镜像，容器，卷和网络被保留。现在称为`Docker Engine-Community`软件包`docker-ce`。

## 安装`Docker Engine-Community`
您可以根据需要以不同的方式安装`Docker Engine-Community`：

- 大多数用户会 设置`Docker`的存储库并从中进行安装，以简化安装和升级任务。这是推荐的方法。

- 一些用户下载并手动安装`RPM`软件包， 并完全手动管理升级。这在诸如在无法访问互联网的空白系统上安装`Docker`的情况下很有用。

- 在测试和开发环境中，一些用户选择使用自动 便利脚本来安装`Docker`。

### 使用存储库安装
在新主机上首次安装`Docker Engine-Community`之前，需要设置`Docker`存储库。之后，您可以从存储库安装和更新`Docker`。

1. 设置存储库

```bash
$ sudo yum install -y yum-utils \
           device-mapper-persistent-data \
           lvm2
```

2. 使用以下命令来设置稳定的存储库。

```bash
$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

#### 安装 Docker CE

1. 安装最新版本 `docker-ce`。

```bash
$ sudo yum install docker-ce docker-ce-cli containerd.io
```

2. 要安装特定版本的`Docker Engine-Community`，请在存储库中列出可用版本，然后选择并安装：

一种。列出并排序您存储库中可用的版本。此示例按版本号（从高到低）对结果进行排序，并被截断：

```bash
$ yum list docker-ce --showduplicates | sort -r

docker-ce.x86_64  3:18.09.1-3.el7                     docker-ce-stable
docker-ce.x86_64  3:18.09.0-3.el7                     docker-ce-stable
docker-ce.x86_64  18.06.1.ce-3.el7                    docker-ce-stable
docker-ce.x86_64  18.06.0.ce-3.el7                    docker-ce-stable
```
```bash
$ sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
```

3. 启动Docker
```bash
$ sudo systemctl start docker
```

4. 通过运行`hello-world` 映像来验证是否正确安装了`Docker Engine-Community` 。
```bash
$ sudo docker run hello-world
```

### 使用快捷脚本自动安装


```bash
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh

<output truncated>
```

## 卸载Docker CE

```bash
$ sudo yum remove docker-ce
$ sudo rm -rf /var/lib/docker
```

## 参考

[官方文档](https://docs.docker.com/install/linux/docker-ce/centos/)
