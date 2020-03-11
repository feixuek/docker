# `Docker`的构建增强

`Docker Build`是`Docker`引擎最常用的功能之一-开发人员，构建团队和发行团队的用户都使用`Docker Build`。

`18.09`版本的`Docker Build`增强功能引入了对构建架构的急需的全面检查。通过集成`BuildKit`，用户应该会看到性能，存储管理，功能功能和安全性方面的改进。

- 可以将使用`buildkit`创建的`Docker`镜像推送到`Docker Hub`和`DTR`中，就像使用旧版生成的`Docker`镜像一样
- 适用于旧版构建的`Dockerfile`格式也将与`buildkit`构建一起使用
- 新的`--secret`命令行选项允许用户传递秘密信息，以使用指定的`Dockerfile`构建新镜像

有关构建选项的更多信息，请参见命令行[构建选项上的参考指南](https://docs.docker.com/engine/reference/commandline/build/)。

## 要求
- 系统要求是`docker-ce x86_64`，`ppc64le`，`s390x`，`aarch64`，`armhf`；或仅`docker-ee x86_64`
- 下载自定义前端的镜像所需的网络连接

## 局限性
- 仅支持构建`Linux`容器
- `BuildKit`模式与`UCP 3.2`或更高版本兼容
- `BuildKit`模式与`Swarm Classic`不兼容

## 启用`buildkit`构建
全新安装`docker`的最简单方法是在调用`docker build`命令时设置`DOCKER_BUILDKIT = 1`环境变量，例如：
```shell
$ DOCKER_BUILDKIT=1 docker build .
```
要默认启用`docker buildkit`，请将`/etc/docker/daemon.json`功能中的守护程序配置设置为`true`并重新启动守护程序：
```shell
{ "features": { "buildkit": true } }
```

## 新的`Docker Build`命令行构建输出
新的`Docker`构建`BuildKit TTY`输出（默认）：
```shell
$ docker build . 
[+] Building 70.9s (34/59)                                                      
 => [runc 1/4] COPY hack/dockerfile/install/install.sh ./install.sh       14.0s
 => [frozen-images 3/4] RUN /download-frozen-image-v2.sh /build  buildpa  24.9s
 => [containerd 4/5] RUN PREFIX=/build/ ./install.sh containerd           37.1s
 => [tini 2/5] COPY hack/dockerfile/install/install.sh ./install.sh        4.9s
 => [vndr 2/4] COPY hack/dockerfile/install/vndr.installer ./              1.6s
 => [dockercli 2/4] COPY hack/dockerfile/install/dockercli.installer ./    5.9s
 => [proxy 2/4] COPY hack/dockerfile/install/proxy.installer ./           15.7s
 => [tomlv 2/4] COPY hack/dockerfile/install/tomlv.installer ./           12.4s
 => [gometalinter 2/4] COPY hack/dockerfile/install/gometalinter.install  25.5s
 => [vndr 3/4] RUN PREFIX=/build/ ./install.sh vndr                       33.2s
 => [tini 3/5] COPY hack/dockerfile/install/tini.installer ./              6.1s
 => [dockercli 3/4] RUN PREFIX=/build/ ./install.sh dockercli             18.0s
 => [runc 2/4] COPY hack/dockerfile/install/runc.installer ./              2.4s
 => [tini 4/5] RUN PREFIX=/build/ ./install.sh tini                       11.6s
 => [runc 3/4] RUN PREFIX=/build/ ./install.sh runc                       23.4s
 => [tomlv 3/4] RUN PREFIX=/build/ ./install.sh tomlv                      9.7s
 => [proxy 3/4] RUN PREFIX=/build/ ./install.sh proxy                     14.6s
 => [dev 2/23] RUN useradd --create-home --gid docker unprivilegeduser     5.1s
 => [gometalinter 3/4] RUN PREFIX=/build/ ./install.sh gometalinter        9.4s
 => [dev 3/23] RUN ln -sfv /go/src/github.com/docker/docker/.bashrc ~/.ba  4.3s
 => [dev 4/23] RUN echo source /usr/share/bash-completion/bash_completion  2.5s
 => [dev 5/23] RUN ln -s /usr/local/completion/bash/docker /etc/bash_comp  2.1s
```
新的`Docker`构建`BuildKit`普通输出：
```shell
$ docker build --progress=plain . 

#1 [internal] load .dockerignore
#1       digest: sha256:d0b5f1b2d994bfdacee98198b07119b61cf2442e548a41cf4cd6d0471a627414
#1         name: "[internal] load .dockerignore"
#1      started: 2018-08-31 19:07:09.246319297 +0000 UTC
#1    completed: 2018-08-31 19:07:09.246386115 +0000 UTC
#1     duration: 66.818µs
#1      started: 2018-08-31 19:07:09.246547272 +0000 UTC
#1    completed: 2018-08-31 19:07:09.260979324 +0000 UTC
#1     duration: 14.432052ms
#1 transferring context: 142B done


#2 [internal] load Dockerfile
#2       digest: sha256:2f10ef7338b6eebaf1b072752d0d936c3d38c4383476a3985824ff70398569fa
#2         name: "[internal] load Dockerfile"
#2      started: 2018-08-31 19:07:09.246331352 +0000 UTC
#2    completed: 2018-08-31 19:07:09.246386021 +0000 UTC
#2     duration: 54.669µs
#2      started: 2018-08-31 19:07:09.246720773 +0000 UTC
#2    completed: 2018-08-31 19:07:09.270231987 +0000 UTC
#2     duration: 23.511214ms
#2 transferring dockerfile: 9.26kB done
```

## 覆盖默认前端
如果您覆盖默认前端，则可以使用`Dockerfile`中的新语法功能。 要覆盖默认前端，请将`Dockerfile`的第一行设置为带有特定前端镜像的注释：
```shell
# syntax = <frontend image>, e.g. # syntax = docker/dockerfile:1.0-experimental
```

## 新的`Docker Build`秘密信息
用于`docker build`的新`--secret`标志允许用户以安全的方式传递要在`Dockerfile`中使用的用于构建`docker`镜像的秘密信息，该信息不会最终存储在最终镜像中。

`id`是要传递到`docker build --secret`的标识符。 该标识符与要在`Dockerfile`中使用的`RUN` `--mount`标识符相关联。 `Docker`不使用将秘密信息保存在`Dockerfile`之外的文件名，因为这可能是敏感信息。

`dst`将秘密文件重命名为`Dockerfile` `RUN`命令中要使用的特定文件。

例如，将秘密信息存储在文本文件中：
```shell
$ echo 'WARMACHINEROX' > mysecret.txt
```
并且使用指定使用`buildkit`前端`docker/dockerfile:1.0-experimental`的`Dockerfile`，可以访问该秘密。

例如：
```shell
# syntax = docker/dockerfile:1.0-experimental
FROM alpine
RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret # shows secret from default secret location
RUN --mount=type=secret,id=mysecret,dst=/foobar cat /foobar # shows secret from custom secret location
```
该`Dockerfile`只是为了证明可以访问该机密。 如您所见，机密信息显示在构建输出中。 构建的最终镜像将没有秘密文件：
```shell
$ docker build --no-cache --progress=plain --secret id=mysecret,src=mysecret.txt .
...
#8 [2/3] RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret
#8       digest: sha256:5d8cbaeb66183993700828632bfbde246cae8feded11aad40e524f54ce7438d6
#8         name: "[2/3] RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret"
#8      started: 2018-08-31 21:03:30.703550864 +0000 UTC
#8 1.081 WARMACHINEROX
#8    completed: 2018-08-31 21:03:32.051053831 +0000 UTC
#8     duration: 1.347502967s


#9 [3/3] RUN --mount=type=secret,id=mysecret,dst=/foobar cat /foobar
#9       digest: sha256:6c7ebda4599ec6acb40358017e51ccb4c5471dc434573b9b7188143757459efa
#9         name: "[3/3] RUN --mount=type=secret,id=mysecret,dst=/foobar cat /foobar"
#9      started: 2018-08-31 21:03:32.052880985 +0000 UTC
#9 1.216 WARMACHINEROX
#9    completed: 2018-08-31 21:03:33.523282118 +0000 UTC
#9     duration: 1.470401133s
...
```

## 使用`SSH`访问构建中的私有数据
> 致谢：有关更多信息和示例，请参阅[`Docker 18.09`中的`Build secrets`和`SSH`转发](https://medium.com/@tonistiigi/build-secrets-and-ssh-forwarding-in-docker-18-09-ae8161d066)。

`docker`构建具有`--ssh`选项，以允许`Docker Engine`转发`SSH`代理连接。 有关`SSH`代理的更多信息，请参见[`OpenSSH`手册页](https://man.openbsd.org/ssh-agent)。

`Dockerfile`中只有通过定义`type=ssh mount`明确请求`SSH`访问的命令才能访问`SSH`代理连接。 其他命令不知道任何可用的`SSH`代理。

要为`Dockerfile`中的`RUN`命令请求`SSH`访问，请使用`ssh`类型定义安装。 这将设置`SSH_AUTH_SOCK`环境变量，以使依赖`SSH`的程序自动使用该套接字。

这是在容器中使用SSH的Dockerfile示例：
```shell
# syntax=docker/dockerfile:experimental
FROM alpine

# Install ssh client and git
RUN apk add --no-cache openssh-client git

# Download public key for github.com
RUN mkdir -p -m 0600 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts

# Clone private repository
RUN --mount=type=ssh git clone git@github.com:myorg/myproject.git myproject
```
创建`Dockerfile`之后，使用`--ssh`选项与`SSH`代理进行连接
```shell
$ docker build --ssh default .
```

## 故障排除：私人仓库的问题
`X509`：未知签署的证书

如果您要从不安全的仓库中获取镜像（使用自签名证书）和/或使用此类仓库作为镜像，那么您将面临`Docker 18.09`中的一个已知问题：
```shell
[+] Building 0.4s (3/3) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 169B
 => [internal] load .dockerignore
 => => transferring context: 2B
 => ERROR resolve image config for docker.io/docker/dockerfile:experimental
------
 > resolve image config for docker.io/docker/dockerfile:experimental:
------
failed to do request: Head https://repo.mycompany.com/v2/docker/dockerfile/manifests/experimental: x509: certificate signed by unknown authority
```
解决方案：正确保护仓库。 您可以免费从`Let's Encrypt`获取`SSL`证书。 参见`https://docs.docker.com/registry/deploying/`

在`SONATYPE NEXUS`版本`<3.15上`运行时，未找到镜像
如果您使用的`Sonatype Nexus`版本`<3.15`运行私有注册表，并收到类似于以下内容的错误：
```shell
------
 > [internal] load metadata for docker.io/library/maven:3.5.3-alpine:
------
------
 > [1/4] FROM docker.io/library/maven:3.5.3-alpine:
------
rpc error: code = Unknown desc = docker.io/library/maven:3.5.3-alpine not found	
```
您可能会遇到以下错误：https://issues.sonatype.org/browse/NEXUS-12684

解决方案是将`Nexus`升级到`3.15`或更高版本。