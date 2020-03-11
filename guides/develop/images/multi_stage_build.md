# 使用多阶段构建

多阶段构建是一项新功能，需要在守护进程和客户端上使用`Docker 17.05`或更高版本。多级构建对于在优化`Dockerfile`的同时使其易于阅读和维护的任何人都非常有用。

## 在多阶段构建之前
关于构建镜像，最具挑战性的事情之一是保持镜像尺寸变小。`Dockerfile`中的每条指令都会在镜像上添加一层，您需要记住在移至下一层之前清除不需要的任何工件。为了编写一个真正有效的`Dockerfile`，传统上，您需要使用`shell`技巧和其他逻辑来使各层尽可能的小，并确保每一层都具有上一层所需的部件。

实际上，通常只有一个`Dockerfile`用于开发（包含构建应用程序所需的一切），而精简的`Dockerfile`用于生产时，它仅包含您的应用程序以及运行该应用程序所需的内容。这被称为“构建者模式”。维护两个`Dockerfile`是不理想的。

这是一个遵循上述构建器模式的`Dockerfile.build`和`Dockerfile`的示例：

```
Dockerfile.build
```
```shell
FROM golang:1.7.3
WORKDIR /go/src/github.com/alexellis/href-counter/
COPY app.go .
RUN go get -d -v golang.org/x/net/html \
  && CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
```
请注意，此示例还使用`Bash &&`运算符将两个`RUN`命令人工压缩在一起，以避免在镜像中创建额外的层。 这是容易失败的并且难以维护。 例如，插入另一个命令很容易，而忘记使用`\`字符继续该行。
```shell
Dockerfile
```
```shell
FROM alpine:latest  
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY app .
CMD ["./app"]  
```
```shell
build.sh
```
```shell
#!/bin/sh
echo Building alexellis2/href-counter:build

docker build --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy \  
    -t alexellis2/href-counter:build . -f Dockerfile.build

docker container create --name extract alexellis2/href-counter:build  
docker container cp extract:/go/src/github.com/alexellis/href-counter/app ./app  
docker container rm -f extract

echo Building alexellis2/href-counter:latest

docker build --no-cache -t alexellis2/href-counter:latest .
rm ./app
```
运行`build.sh`脚本时，它需要构建第一个镜像，从中创建一个容器以复制工件，然后构建第二个镜像。 这两个镜像都占用了系统空间，并且本地磁盘上仍然有应用程序构件。

多阶段构建极大地简化了这种情况！

## 使用多阶段构建
通过多阶段构建，您可以在`Dockerfile`中使用多个`FROM`语句。 每个`FROM`指令可以使用不同的基础，并且每个都开始构建的新阶段。 您可以有选择地将部件从一个阶段复制到另一个阶段，从而在最终镜像中留下不需要的所有内容。 为了展示它是如何工作的，让我们调整上一节中的`Dockerfile`以使用多阶段构建。
```shell
Dockerfile
```
```shell
FROM golang:1.7.3
WORKDIR /go/src/github.com/alexellis/href-counter/
RUN go get -d -v golang.org/x/net/html  
COPY app.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine:latest  
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=0 /go/src/github.com/alexellis/href-counter/app .
CMD ["./app"]  
```
您只需要单个`Dockerfile`。 您也不需要单独的构建脚本。 只需运行`docker build`。
```shell
$ docker build -t alexellis2/href-counter:latest .
```
最终结果是与以前相同的微小生产镜像，并大大降低了复杂性。 您无需创建任何中间镜像，也无需将任何工件提取到本地系统。

它是如何工作的？ 第二条`FROM`指令以`alpine:latest`镜像为基础开始新的构建阶段。 `COPY --from = 0`行仅将之前阶段的构建工件复制到此新阶段。 `Go SDK`和任何中间工件都被保留了下来，没有保存在最终镜像中。

## 命名您的构建阶段
默认情况下，未命名阶段，您可以通过它们的整数来引用它们，对于第一个`FROM`指令，它们以`0`开头。 但是，可以通过在`FROM`指令中添加`AS <NAME>`来命名阶段。 本示例通过命名阶段并使用`COPY`指令中的名称来改进前一个示例。 这意味着，即使稍后对`Dockerfile`中的指令进行了重新排序，`COPY`也不会中断。
```shell
FROM golang:1.7.3 AS builder
WORKDIR /go/src/github.com/alexellis/href-counter/
RUN go get -d -v golang.org/x/net/html  
COPY app.go    .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine:latest  
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /go/src/github.com/alexellis/href-counter/app .
CMD ["./app"]  
```
## 在特定的构建阶段停止
构建镜像时，不必构建整个`Dockerfile`，包括每个阶段。 您可以指定目标构建阶段。 以下命令假定您正在使用以前的`Dockerfile`，但在名为`builder`的阶段停止：
```shell
$ docker build --target builder -t alexellis2/href-counter:latest .
```
一些可能非常强大的方案是：

- 调试特定的构建阶段
- 使用启用了所有调`debug`号或工具的调试阶段以及精益`production`阶段
- 使用`testing`阶段，在该阶段中，您的应用将填充测试数据，但使用另一个使用真实数据的阶段进行生产构建

## 将外部镜像用作`stage`
使用多阶段构建时，您不仅限于从之前在`Dockerfile`中创建的阶段进行复制。 您可以使用`COPY --from`指令从单独的镜像进行复制，方法是使用本地镜像名称，本地或`Docker`仓库中可用的标签或标签`ID`。 
`Docker`客户端在必要时提取镜像并从那里复制部件。 语法为：
```shell
COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf
```

## 将上一个阶段用作新阶段
在使用`FROM`指令时，可以通过引用前一个阶段结束的地方来进行选择。 例如：
```shell
FROM alpine:latest as builder
RUN apk --no-cache add build-base

FROM builder as build1
COPY source1.cpp source.cpp
RUN g++ -o /binary source.cpp

FROM builder as build2
COPY source2.cpp source.cpp
RUN g++ -o /binary source.cpp
```