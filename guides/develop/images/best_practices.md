# 编写`Dockerfile`的最佳实践

本文档介绍了用于构建有效镜像的推荐最佳实践和方法。

`Docker`通过读取`Dockerfile`的指令自动构建镜像，该文件是一个文本文件，其中依次包含构建给定镜像所需的所有命令。 	`Dockerfile`遵循特定的格式和指令集，您可以在[`Dockerfile`参考](https://docs.docker.com/engine/reference/builder/)中找到该文件。

`Docker`镜像由只读层组成，每个只读层代表一个`Dockerfile`指令。 各个层堆叠在一起，每个层都是上一层的变化的增量。 考虑以下`Dockerfile`：

```Dockerfile
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```

每条指令创建一层：

- `FROM`从`ubuntu:18.04` `Docker`镜像创建一个层。
- `COPY`从`Docker`客户端的当前目录添加文件。
- `RUN`使用`make`构建您的应用程序。
- `CMD`指定在容器内运行什么命令。

运行镜像并生成容器时，可以在基础层之上添加一个新的可写层（“容器层”）。 对运行中的容器所做的所有更改（例如写入新文件，修改现有文件和删除文件）都将写入此薄可写容器层。

有关镜像层（以及`Docker`如何构建和存储镜像）的更多信息，请参阅关于[存储驱动程序](https://docs.docker.com/storage/storagedriver/)。

## 一般准则和建议
### 创建临时容器
`Dockerfile`定义的镜像应该生成尽可能短暂的容器。 "短暂"是指可以停止并破坏容器，然后对其进行重建和替换，并使用绝对的最小限度的设置和配置。

请参阅"十二因子应用程序"方法下的["过程"](https://12factor.net/processes)，以了解以这种无状态方式运行容器的动机。

### 了解构建上下文
当您发出`docker build`命令时，当前的工作目录称为构建上下文。默认情况下，假定`Dockerfile`位于此处，但是您可以使用文件标志（`-f`）指定其他位置。无论`Dockerfile`实际位于何处，当前目录中文件和目录的所有递归内容都将作为构建上下文发送到`Docker`守护程序。

> 构建上下文示例

> 创建一个用于构建上下文的目录，并将其`cd`进入目录。将"hello" 写入名为`hello`的文本文件，并创建一个在其上运行`cat`的`Dockerfile`。从构建上下文（`.`）中构建镜像：
```shell
mkdir myproject && cd myproject
echo "hello" > hello
echo -e "FROM busybox\nCOPY /hello /\nRUN cat /hello" > Dockerfile
docker build -t helloapp:v1 .
```
> 将`Dockerfile`和`hello`移到单独的目录中，并构建镜像的第二个版本（不依赖上次构建的缓存）。 使用`-f`指向`Dockerfile`并指定构建上下文的目录：
```shell
mkdir -p dockerfiles context
mv Dockerfile dockerfiles && mv hello context
docker build --no-cache -t helloapp:v2 -f dockerfiles/Dockerfile context
```
疏忽地包含了构建镜像所不需要的文件会导致较大的构建上下文和较大的镜像大小。 这会增加生成镜像的时间，拉动和推动镜像的时间以及容器运行时的大小。 要查看您的构建上下文有多大，请在构建`Dockerfile`时查找如下消息：
```shell
Sending build context to Docker daemon  187.8MB
```

### 通过`stdin`管道`Dockerfile`
`Docker`可以通过使用本地或远程构建上下文的`stdin`传递`Dockerfile`来构建镜像。 通过`stdin`插入`Dockerfile`可以在不将`Dockerfile`写入磁盘的情况下执行一次性构建，或者在生成`Dockerfile`且以后不应持续的情况下使用。

> 为了方便起见，本节中的示例使用此处的文档，但是可以使用在`stdin`上提供`Dockerfile`的任何方法。

> 例如，以下命令是等效的：
```shell
echo -e 'FROM busybox\nRUN echo "hello world"' | docker build -
```
```shell
docker build -<<EOF
FROM busybox
RUN echo "hello world"
EOF
```

> 您可以使用首选方法或最适合您的用例的方法替换示例。

在不发送构建上下文的情况下，使用来自`STDIN`的`DOCKERFILE`建立镜像
使用此语法可使用来自`stdin`的`Dockerfile`构建镜像，而无需发送其他文件作为构建上下文。 连字符（`-`）占据`PATH`的位置，并指示`Docker`从`stdin`而不是目录中读取构建上下文（仅包含`Dockerfile`）：
```shell
docker build [OPTIONS] -
```
以下示例使用通过`stdin`传递的`Dockerfile`构建镜像。 没有文件作为构建上下文发送到守护程序。
```shell
docker build -t myimage:latest -<<EOF
FROM busybox
RUN echo "hello world"
EOF
```
在您的`Dockerfile`不需要将文件复制到镜像中的情况下，省略构建上下文会很有用，并且由于没有文件发送到守护程序，因此可以提高构建速度。

如果要通过从构建上下文中排除某些文件来提高构建速度，请使用`.dockerignore`进行排除。

> 注意：如果使用此语法，尝试构建使用`COPY`或`ADD`的`Dockerfile`将会失败。 以下示例说明了这一点：
```shell
# create a directory to work in
mkdir example
cd example

# create an example file
touch somefile.txt

docker build -t myimage:latest -<<EOF
FROM busybox
COPY somefile.txt .
RUN cat /somefile.txt
EOF

# observe that the build fails
...
Step 2/3 : COPY somefile.txt .
COPY failed: stat /var/lib/docker/tmp/docker-builder249218248/somefile.txt: no such file or directory
```
使用`STDIN`的`DOCKERFILE`从本地构建上下文构建
使用此语法可使用本地文件系统上的文件，但使用来自`stdin`的`Dockerfile`构建镜像。 该语法使用`-f`（或`--file`）选项指定要使用的`Dockerfile`，并使用连字符（`-`）作为文件名来指示`Docker`从`stdin`中读取`Dockerfile`：
```shell
docker build [OPTIONS] -f- PATH
```
下面的示例使用当前目录（`.`）作为构建上下文，并使用`Dockerfile`构建镜像，并使用[`here`文档](http://tldp.org/LDP/abs/html/here-docs.html)通过`stdin`传递该文件。
```shell
# create a directory to work in
mkdir example
cd example

# create an example file
touch somefile.txt

# build an image using the current directory as context, and a Dockerfile passed through stdin
docker build -t myimage:latest -f- . <<EOF
FROM busybox
COPY somefile.txt .
RUN cat /somefile.txt
EOF
```
使用`STDIN`的`DOCKERFILE`从远程构建上下文构建
使用此语法使用来自`stdin`的`Dockerfile`，使用来自远程`git`存储库中的文件来构建镜像。 该语法使用`-f`（或`--file`）选项指定要使用的`Dockerfile`，并使用连字符（`-`）作为文件名来指示`Docker`从`stdin`中读取`Dockerfile`：
```shell
docker build [OPTIONS] -f- PATH
```
如果您要从不包含`Dockerfile`的存储库中构建镜像，或者想要使用自定义`Dockerfile`进行构建而又不维护自己的存储库派生，则此语法很有用。

下面的示例使用`stdin`中的`Dockerfile`构建镜像，并从`GitHub`上的`hello-world` `Git`存储库添加`hello.c`文件。
```shell
docker build -t myimage:latest -f- https://github.com/docker-library/hello-world.git <<EOF
FROM busybox
COPY hello.c .
EOF
```
> 引擎盖下
> 当使用远程`Git`存储库作为构建上下文构建镜像时，`Docker`在本地计算机上执行存储库的`git`克隆，并将这些文件作为构建上下文发送到守护程序。此功能要求将`git`安装在运行`docker build`命令的主机上。

### 用`.dockerignor`e排除
要排除与构建无关的文件（无需重构源存储库），请使用`.dockerignore`文件。该文件支持类似于`.gitignore`文件的排除模式。有关创建一个的信息，请参见`.dockerignore`文件。

### 使用多阶段构建
多阶段构建使您可以大幅度减小最终镜像的大小，而不必努力减少中间层和文件的数量。

由于镜像是在生成过程的最后阶段生成的，因此可以利用生成缓存来最小化镜像层。

例如，如果您的构建包含多个层，则可以将它们从更改频率较低（以确保生成缓存可重用）到更改频率较高的顺序排序：

- 安装构建应用程序所需的工具

- 安装或更新库依赖项

- 生成您的申请

`Go`应用程序的`Dockerfile`可能类似于：
```Dockerfile
FROM golang:1.11-alpine AS build

# Install tools required for project
# Run `docker build --no-cache .` to update dependencies
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies with Gopkg.toml and Gopkg.lock
# These layers are only re-built when Gopkg files are updated
COPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# Install library dependencies
RUN dep ensure -vendor-only

# Copy the entire project and build it
# This layer is rebuilt when a file changes in the project directory
COPY . /go/src/project/
RUN go build -o /bin/project

# This results in a single layer image
FROM scratch
COPY --from=build /bin/project /bin/project
ENTRYPOINT ["/bin/project"]
CMD ["--help"]
```

### 不要安装不必要的软件包
为了降低复杂性，依赖性，文件大小和构建时间，请避免仅仅因为它们“很容易安装”而安装多余或不必要的软件包。例如，您不需要在数据库镜像中包含文本编辑器。

### 解耦应用
每个容器应该只有一个关注点。将应用程序解耦到多个容器中，可以更轻松地水平缩放和重复使用容器。例如，一个`Web`应用程序堆栈可能由三个单独的容器组成，每个容器都有自己的唯一镜像，以分离的方式管理Web应用程序，数据库和内存中的缓存。

将每个容器限制为一个进程是一个很好的经验法则，但这并不是一成不变的规则。例如，不仅可以使用初始化进程来生成容器，而且某些程序还可以自行生成其他进程。例如，`Celery`可以产生多个工作进程，而`Apache`可以为每个请求创建一个进程。

根据您的最佳判断，使容器保持清洁和模块化。如果容器相互依赖，则可以使用`Docker`容器网络来确保这些容器可以通信。

### 减少层数
在较旧的`Docker`版本中，务必最小化镜像中的层数以确保其性能。添加了以下功能来减少此限制：

- 只有指令`RUN`，`COPY`，`ADD`会创建层。其他说明创建临时的中间镜像，并且不会增加构建的大小。
- 尽可能使用多阶段构建，并且仅将所需的工件复制到最终镜像中。这使您可以在中间构建阶段中包含工具和调试信息，而无需增加最终镜像的大小。

### 排序多行参数
尽可能通过字母数字排序多行参数来简化以后的更改。 这有助于避免软件包重复，并使列表更易于更新。 这也使`PR`易于阅读和查看。 在反斜杠（`\`）之前添加空格也有帮助。

这是来自`buildpack-deps`镜像的示例：
```Dockerfile
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion
```

### 利用构建缓存
构建镜像时，`Docker`会逐步执行`Dockerfile`中的指令，并按指定的顺序执行每个指令。在检查每条指令时，`Docker`会在其缓存中寻找一个可以重用的现有镜像，而不是创建一个新的（重复的）镜像。

如果根本不想使用缓存，则可以在`docker build`命令上使用`--no-cache=true`选项。但是，如果您确实让`Docker`使用其缓存，那么了解何时可以找到匹配的镜像非常重要。 `Docker`遵循的基本规则概述如下：

- 从已在缓存中的父镜像开始，将下一条指令与从该基本镜像派生的所有子镜像进行比较，以查看是否其中一个是使用完全相同的指令构建的。如果不是，则高速缓存无效。
- 在大多数情况下，仅将`Dockerfile`中的指令与子镜像之一进行比较就足够了。但是，某些说明需要更多的检查和解释。
- 对于`ADD`和`COPY`指令，将检查镜像中文件的内容，并为每个文件计算一个校验和。在这些校验和中不考虑文件的最后修改时间和最后访问时间。在高速缓存查找期间，将校验和与现有镜像中的校验和进行比较。如果文件中的任何内容（例如内容和元数据）发生了更改，则缓存将无效。
- 除了`ADD`和`COPY`命令以外，缓存检查不会查看容器中的文件来确定缓存是否匹配。例如，在处理`RUN apt-get -y update`命令时，不会检查容器中更新的文件，以确定是否存在缓存命中。在这种情况下，仅使用命令字符串本身来查找匹配项。

缓存无效后，所有后续`Dockerfile`命令都会生成新镜像，并且不使用缓存。

## `Dockerfile`说明
这些建议旨在帮助您创建高效且可维护的`Dockerfile`。

### `FROM`
`Dockerfile`的[`FROM`指令参考](https://docs.docker.com/engine/reference/builder/#from)

尽可能使用当前的官方镜像作为镜像的基础。 我们建议使用`Alpine`镜像，因为它受到严格控制且尺寸较小（当前小于`5MB`），同时仍是完整的`Linux`发行版。

### `LABEL`
[了解`LABEL`标签](https://docs.docker.com/config/labels-custom-metadata/)

您可以在镜像上添加标签，以帮助按项目组织镜像，记录许可信息，帮助自动化或出于其他原因。 对于每个标签，添加一行以`LABEL`开头并带有一个或多个键值对的行。 以下示例显示了不同的可接受格式。 内嵌包含解释性注释。

> 带空格的字符串必须用引号引起来，否则必须转义空格。 内引号字符（`"`）也必须转义。

```Dockerfile
# Set one or more individual labels
LABEL com.example.version="0.0.1-beta"
LABEL vendor1="ACME Incorporated"
LABEL vendor2=ZENITH\ Incorporated
LABEL com.example.release-date="2015-02-12"
LABEL com.example.version.is-production=""
```
一幅镜像可以有多个标签。 在`Docker 1.10`之前，建议将所有标签合并为一个`LABEL`指令，以防止创建额外的层。 不再需要此操作，但仍支持组合标签。
```Dockerfile
# Set multiple labels on one line
LABEL com.example.version="0.0.1-beta" com.example.release-date="2015-02-12"
```
上面也可以这样写：
```Dockerfile
# Set multiple labels at once, using line-continuation characters to break long lines
LABEL vendor=ACME\ Incorporated \
      com.example.is-beta= \
      com.example.is-production="" \
      com.example.version="0.0.1-beta" \
      com.example.release-date="2015-02-12"
```
请参阅了解[对象标签](https://docs.docker.com/config/labels-custom-metadata/)以获取有关可接受的标签键和值的准则。有关查询标​​签的信息，请参阅[管理对象上的标签](https://docs.docker.com/config/labels-custom-metadata/#managing-labels-on-objects)中与过滤相关的项目。另请参阅`Dockerfile`参考中的[`LABEL`](https://docs.docker.com/engine/reference/builder/#label)。

### `RUN`
[`RUN`指令的`Dockerfile`参考](https://docs.docker.com/engine/reference/builder/#run)

将多行长或复杂的`RUN`语句分割成多行，并用反斜杠分隔，以使`Dockerfile`更具可读性，可理解性和可维护性。

#### `APT-GET`

`RUN`的最常见用例可能是`apt-get`的应用程序。因为它安装软件包，所以`RUN apt-get`命令需要注意一些陷阱。

避免使用`RUN apt-get`升级和`dist-upgrade`，因为父镜像中的许多“基本”程序包无法在无特权的容器中升级。如果父镜像中包含的软件包已过期，请联系其维护者。如果您知道有特定的软件包`foo`需要更新，请使用`apt-get install -y foo`自动更新。
```shell
RUN apt-get update && apt-get install -y \
    package-bar \
    package-baz \
    package-foo
```
在`RUN`语句中单独使用`apt-get update`会导致缓存问题，并且后续的`apt-get`安装说明将失败。 例如，假设您有一个`Dockerfile`：
```shell
FROM ubuntu:18.04
RUN apt-get update
RUN apt-get install -y curl
```
构建镜像后，所有层都在`Docker`缓存中。 假设您稍后通过添加额外的软件包来修改`apt-get install`：
```shell
FROM ubuntu:18.04
RUN apt-get update
RUN apt-get install -y curl nginx
```
`Docker`将初始指令和修改后的指令视为相同，并重复使用先前步骤中的缓存。 结果，由于构建使用了缓存版本，因此不执行`apt-get`更新。 由于未运行`apt-get`更新，因此您的构建可能会获得`curl`和`nginx`软件包的过时版本。

使用`RUN apt-get update && apt-get install -y`可确保您的`Dockerfile`安装最新的软件包版本，而无需进一步的编码或手动干预。 这种技术称为“缓存清除”。 您还可以通过指定软件包版本来实现缓存清除。 这称为版本固定，例如：
```shell
RUN apt-get update && apt-get install -y \
    package-bar \
    package-baz \
    package-foo=1.3.*
```
版本固定会强制构建物检索特定版本，而不管缓存中的内容是什么。 该技术还可以减少由于所需包装中的意外更改而导致的故障。

下面是格式正确的`RUN`指令，演示了所有的`apt-get`建议。
```shell
RUN apt-get update && apt-get install -y \
    aufs-tools \
    automake \
    build-essential \
    curl \
    dpkg-sig \
    libcap-dev \
    libsqlite3-dev \
    mercurial \
    reprepro \
    ruby1.9.1 \
    ruby1.9.1-dev \
    s3cmd=1.1.* \
 && rm -rf /var/lib/apt/lists/*
```

`s3cmd`参数指定版本`1.1。*`。 如果镜像先前使用的是旧版本，则指定新版本会导致`apt-get`更新的缓存崩溃，并确保安装新版本。 在每行上列出软件包还可以防止软件包重复中的错误。

另外，当通过删除`/var/lib/apt/lists`清理`apt`缓存时，由于`apt`缓存未存储在层中，因此会减小镜像大小。 由于`RUN`语句以`apt-get`更新开始，因此始终在`apt-get`安装之前刷新程序包缓存。

> 官方的`Debian`和`Ubuntu`镜像会自动运行`apt-get clean`，因此不需要显式调用。

#### 使用管道

某些`RUN`命令取决于使用管道字符（`|`）将一个命令的输出管道传输到另一个命令的能力，如以下示例所示：
```shell
RUN wget -O - https://some.site | wc -l > /number
```

`Docker`使用`/bin/sh -c`解释器执行这些命令，该解释器仅评估管道中最后一个操作的退出代码以确定成功。 在上面的示例中，即使`wget`命令失败，只要`wc -l`命令成功，此构建步骤也会成功并生成一个新镜像。

如果您希望由于管道中的任何阶段的错误而导致命令失败，请在`set -o pipefail &&`之前添加前缀，以确保意外的错误可以防止构建意外进行。 例如：
```shell
RUN set -o pipefail && wget -O - https://some.site | wc -l > /number
```

> 并非所有的`shell`程序都支持`-o pipefail`选项。
> 如果是基于`Debian`的镜像上的`dash`，请考虑使用`RUN`的`exec`形式显式选择一个确实支持`pipefail`选项的`shell`。 例如：
```shell
RUN ["/bin/bash", "-c", "set -o pipefail && wget -O - https://some.site | wc -l > /number"]
```

### `CMD`
[`CMD`指令的`Dockerfile`参考](https://docs.docker.com/engine/reference/builder/#cmd)

应使用`CMD`指令来运行镜像所包含的软件以及所有参数。 `CMD`几乎应始终以`CMD ["executable","param1","param2"…]`的形式使用。因此，如果镜像用于服务（例如`Apache`和`Rails`），则将运行诸如`CMD [“"apache2","-DFOREGROUND"]`之类的内容。实际上，建议将这种形式的指令用于任何基于服务的镜像。

在大多数其他情况下，应为`CMD`提供交互式`shell`，例如`bash`，`python`和`perl`。例如，`CMD ["perl","-de0"]`，`CMD["python"]`或`CMD ["php","-a"]`。使用这种形式意味着，当您执行诸如`docker run -it python`之类的操作时，您将进入可用的`shell`中，随时可以使用。除非您和您的预期用户已经非常熟悉`ENTRYPOINT`的工作原理，否则`CMD`很少以`CMD ["param"，"param"]`的形式与`ENTRYPOINT`结合使用。

### `EXPOSE`
[有关`EXPOSE`指令的`Dockerfile`参考](https://docs.docker.com/engine/reference/builder/#expose)

`EXPOSE`指令指示容器在其上侦听连接的端口。因此，应为应用程序使用通用的传统端口。例如，包含`Apache Web`服务器的镜像将使用`EXPOSE 80`，而包含`MongoDB`的镜像将使用`EXPOSE 27017`，依此类推。

对于外部访问，您的用户可以执行带有标志的`docker run`，该标志指示如何将指定端口映射到他们选择的端口。对于容器链接，`Docker`为从接收者容器到源容器的路径（即，`MYSQL_PORT_3306_TCP`）提供了环境变量。

### `ENV`
[`ENV`指令的`Dockerfile`参考]()

为了使新软件更易于运行，可以使用`ENV`为容器安装的软件更新`PATH`环境变量。 例如，`ENV PATH /usr/local/nginx/bin:$PATH`确保`CMD [
"nginx"]`正常工作。

`ENV`指令还可用于提供特定于您希望容器化的服务的必需环境变量，例如`Postgres`的`PGDATA`。

最后，`ENV`还可以用于设置常用的版本号，以便更容易维护版本凹凸，如以下示例所示：

```shell
ENV PG_MAJOR 9.3
ENV PG_VERSION 9.3.4
RUN curl -SL http://example.com/postgres-$PG_VERSION.tar.xz | tar -xJC /usr/src/postgress && …
ENV PATH /usr/local/postgres-$PG_MAJOR/bin:$PATH
```

类似于在程序中具有常量变量（与硬编码值相反），此方法使您可以更改单个`ENV`指令以自动神奇地修改容器中软件的版本。

每条`ENV`线都会创建一个新的中间层，就像`RUN`命令一样。 这意味着，即使您在以后的层中取消设置环境变量，它也仍然保留在该层中，并且其值也无法转储。 您可以通过创建如下所示的`Dockerfile`，然后对其进行构建来进行测试。
```shell
FROM alpine
ENV ADMIN_USER="mark"
RUN echo $ADMIN_USER > ./mark
RUN unset ADMIN_USER
```
```shell
$ docker run --rm test sh -c 'echo $ADMIN_USER'

mark
```
为避免这种情况，并真正取消设置环境变量，请在外壳程序中使用`RUN`命令和`shell`命令，以在单个层中全部设置，使用和取消设置该变量。 您可以使用分隔命令`;`要么`&&`。 如果您使用第二种方法，并且其中一个命令失败，则`docker build`也将失败。 这通常是个好主意。 将`\`用作`Linux Dockerfiles`的行连续字符可提高可读性。 您还可以将所有命令放入一个`Shell`脚本中，并让`RUN`命令运行该`Shell`脚本。

```shell
FROM alpine
RUN export ADMIN_USER="mark" \
    && echo $ADMIN_USER > ./mark \
    && unset ADMIN_USER
CMD sh
```
```shell
$ docker run --rm test sh -c 'echo $ADMIN_USER'
```

### `ADD`或`COPY`
[用于`ADD`指令的`Dockerfile`参考](https://docs.docker.com/engine/reference/builder/#add)
[`COPY`指令的`Dockerfile`参考](https://docs.docker.com/engine/reference/builder/#copy)

尽管`ADD`和`COPY`在功能上相似，但通常来说`COPY`是首选。 那是因为它比`ADD`更透明。 `COPY`仅支持将本地文件基本复制到容器中，而`ADD`的某些功能（如仅本地`tar`提取和远程`URL`支持）并不是立即显而易见的。 因此，与`ADD rootfs.tar.xz /`中一样，`ADD`的最佳用途是将本地`tar`文件自动提取到镜像中。

如果您有多个使用不同上下文的文件的`Dockerfile`步骤，请单独复制而不是一次全部复制。 这样可以确保仅在特别需要的文件发生更改时，才使每个步骤的构建缓存无效（强制重新运行该步骤）。

例如：
```shell
COPY requirements.txt /tmp/
RUN pip install --requirement /tmp/requirements.txt
COPY . /tmp/
```
与放置`COPY`相比，导致`RUN`步骤的缓存失效更少。 `/tmp/`之前。

由于镜像大小很重要，因此强烈建议不要使用`ADD`从远程`URL`获取软件包。 您应该使用`curl`或`wget`代替。 这样，您可以在提取文件后删除不再需要的文件，而不必在镜像中添加其他图层。 例如，您应该避免做以下事情：
```shell
ADD http://example.com/big.tar.xz /usr/src/things/
RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
RUN make -C /usr/src/things all
```

相反，请执行以下操作：
```shell
RUN mkdir -p /usr/src/things \
    && curl -SL http://example.com/big.tar.xz \
    | tar -xJC /usr/src/things \
    && make -C /usr/src/things all
```

对于不需要`ADD`的`tar`自动提取功能的其他项目（文件，目录），应始终使用`COPY`。

### `ENTRYPOINT`
[`ENTRYPOINT`指令的`Dockerfile`参考](https://docs.docker.com/engine/reference/builder/#entrypoint)

`ENTRYPOINT`的最佳用途是设置镜像的主命令，使该镜像像该命令一样运行（然后使用`CMD`作为默认标志）。

让我们从命令行工具`s3cmd`的镜像示例开始：
```shell
ENTRYPOINT ["s3cmd"]
CMD ["--help"]
```
现在可以像这样运行镜像以显示命令的帮助：
```shell
$ docker run s3cmd
```
或使用正确的参数执行命令：
```shell
$ docker run s3cmd ls s3://mybucket
```
这很有用，因为镜像名称可以用作对二进制文件的引用，如上面的命令所示。

`ENTRYPOIN`T指令也可以与辅助脚本结合使用，即使启动该工具可能需要一个以上的步骤，也可以使其与上述命令类似地工作。

例如，`Postgres Official Image`使用以下脚本作为其`ENTRYPOINT`：
```shell
#!/bin/bash
set -e

if [ "$1" = 'postgres' ]; then
    chown -R postgres "$PGDATA"

    if [ -z "$(ls -A "$PGDATA")" ]; then
        gosu postgres initdb
    fi

    exec gosu postgres "$@"
fi

exec "$@"
```
> 将应用程序配置为`PID 1`
> 该脚本使用`exec Bash`命令，以便最终运行的应用程序成为容器的`PID1`。这使该应用程序可以接收发送到该容器的所有`Unix`信号。 有关更多信息，请参见`ENTRYPOINT`参考。

将帮助程序脚本复制到容器中，并在容器启动时通过`ENTRYPOINT`运行：
```shell
COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["postgres"]
```
该脚本允许用户以多种方式与`Postgres`进行交互。

它可以简单地启动`Postgres`：
```shell
$ docker run postgres
```
或者，它可以用于运行`Postgres`并将参数传递给服务器：
```shell
$ docker run postgres postgres --help
```
最后，它也可以用于启动一个完全不同的工具，例如`Bash`：
```shell
$ docker run --rm -it postgres bash
```

### `VOLUME`
[`VOLUME`指令的`Dockerfile`参考](https://docs.docker.com/engine/reference/builder/#volume)

`VOLUME`指令应用于公开由`Docker`容器创建的任何数据库存储区，配置存储或文件`/`文件夹。强烈建议您将`VOLUME`用于镜像的任何可变和`/`或用户可维修的部分。

### `USER`
[`USER`指令的`Dockerfile`参考](https://docs.docker.com/engine/reference/builder/#user)

如果服务可以在没有特权的情况下运行，请使用`USER`更改为非`root`用户。首先在`Dockerfile`中使用`RUN groupadd -r postgres && useradd --no-log-init -r -g postgres postgres`等创建用户和组。

考虑一个明确的`UID/GID`

> 为镜像中的用户和组分配了不确定的`UID/GID`，因为无论镜像重建如何，都将分配“下一个” `UID/GID`。因此，如果有必要，您应该分配一个明确的`UID/GID`。

> 由于`Go`存档`/tar`软件包处理稀疏文件中的一个未解决的错误，尝试在`Docker`容器内创建具有非常大的`UID`的用户可能会导致磁盘耗尽，因为容器层中的`/var/log/faillog`充满了`NULL(\0)`字符。一种解决方法是将`--no-log-init`标志传递给`useradd`。 `Debian/Ubuntu adduser`包装器不支持此标志。

避免安装或使用`sudo`，因为它具有不可预测的`TTY`和信号转发行为，可能会导致问题。如果您绝对需要类似于`sudo`的功能，例如将守护程序初始化为`root`却以非`root`身份运行，请考虑使用`gosu`。

最后，为减少层次和复杂性，请避免频繁来回切换`USER`。

### `WORKDIR`
[`WORKDIR`指令的`Dockerfile`参考]()

为了清楚和可靠，您应该始终为`WORKDIR`使用绝对路径。另外，您应该使用`WORKDIR`而不是增加诸如`RUN cd…&& do-something`之类的指令，这些指令难以阅读，排除故障和维护。

### `ONBUILD`
[`Dockerfile`的`ONBUILD`指令参考]()

当前`Dockerfile`构建完成后，将执行`ONBUILD`命令。 `ONBUILD`在从当前镜像派生的任何子镜像中执行。将`ONBUILD`命令视为父`Dockerfile`给子`Dockerfile`的指令。

`Docker`构建在子`Dockerfile`中的任何命令之前执行`ONBUILD`命令。

对于要从​​给定镜像构建的镜像，`ONBUILD`非常有用。例如，您可以将`ONBUILD`用于语言堆栈镜像，以在`Dockerfile`中构建用该语言编写的任意用户软件，如`Ruby`的`ONBUILD`变体所示。

使用`ONBUILD`构建的镜像应获得单独的标签，例如：`ruby:1.9-onbuild`或`ruby:2.0-onbuild`。

将`ADD`或`COPY`设置为`ONBUILD`时要小心。如果新构建的上下文缺少要添加的资源，则` onbuild`镜像将灾难性地失败。如上所述，添加一个单独的标签，可以通过允许`Dockerfile`作者做出选择来缓解这种情况。

## 官方镜像的例子
这些官方镜像具有示例性的`Dockerfile`：

- [`Go`](https://hub.docker.com/_/golang/)
- [`Perl`](https://hub.docker.com/_/perl/)
- [`Hy`](https://hub.docker.com/_/hylang/)
- [`Ruby`](https://hub.docker.com/_/ruby/)