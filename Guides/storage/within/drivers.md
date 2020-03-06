# 关于存储驱动程序

为了有效地使用存储驱动程序，了解`Docker`如何构建和存储镜像以及容器如何使用这些镜像非常重要。您可以使用此信息来做出明智的选择，以最佳方式保存应用程序中的数据并避免性能问题。

存储驱动程序使您可以在容器的可写层中创建数据。删除容器后文件将不会保留，并且读写速度都低于本机文件系统性能。

> 注意：已知有问题的操作包括写密集型数据库存储，尤其是在只写层中存在预先存在的数据时。本文档中提供了更多详细信息。

[了解如何使用卷](https://docs.docker.com/storage/volumes/)来保留数据并提高性能。

## 镜像和层
`Docker`镜像由一系列层组成。每层代表镜像的`Dockerfile`中的一条指令。除最后一层外的每一层都是只读的。考虑以下`Dockerfile`：
```Dockerfile
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```
该`Dockerfile`包含四个命令，每个命令创建一个层。 `FROM`语句从`ubuntu：18.04`镜像创建一个层开始。 `COPY`命令从`Docker`客户端的当前目录添加一些文件。 `RUN`命令使用`make`命令构建您的应用程序。 最后，最后一层指定在容器中运行什么命令。

每层仅是与其之前的层的一组差异。 这些层彼此堆叠。 创建新容器时，可以在基础层之上添加新的可写层。 该层通常称为"容器层"。 对运行中的容器所做的所有更改（例如写入新文件，修改现有文件和删除文件）都将写入此可写容器层。 下图显示了基于`Ubuntu 18.04`镜像的容器。

![容器层](../images/container-layers.jpg)

存储驱动程序处理有关这些层彼此交互的方式的详细信息。 提供了不同的存储驱动程序，它们在不同情况下各有利弊。

## 容器和层
容器和镜像之间的主要区别是可写顶层。 在容器中添加新数据或修改现有数据的所有写操作都存储在此可写层中。 删除容器后，可写层也会被删除。 基础镜像保持不变。

因为每个容器都有其自己的可写容器层，并且所有更改都存储在该容器层中，所以多个容器可以共享对同一基础镜像的访问，但具有自己的数据状态。 下图显示了共享相同`Ubuntu 18.04`镜像的多个容器。

![共享层](../images/sharing-layers.jpg)

> 注意：如果您需要多个镜像才能共享对完全相同的数据的访问权限，请将该数据存储在`Docker`卷中并将其装入到您的容器中。

`Docker`使用存储驱动程序来管理镜像层和可写容器层的内容。每个存储驱动程序对实现的处理方式不同，但是所有驱动程序都使用可堆叠的镜像层和写时复制（`CoW`）策略。

## 磁盘上的容器大小
要查看正在运行的容器的大致大小，可以使用`docker ps -s`命令。有两个不同的列与大小有关。

- `size`：每个容器的可写层使用的数据量（磁盘上）。

- `virtual size`：容器使用的用于只读镜像数据的数据量加上容器的可写层`size`。多个容器可以共享部分或全部只读镜像数据。从同一镜像开始的两个容器共享100％的只读数据，而具有不同镜像且具有相同公共层的两个容器共享这些公共层。因此，您不能只对虚拟大小进行总计。这高估了总磁盘使用量，可能是一笔不小的数目。

磁盘上所有正在运行的容器使用的磁盘总空间是每个容器的`size`和`virtual szie`值的某种组合。如果多个容器从相同的精确镜像开始，则这些容器在磁盘上的总大小将为`SUM`（容器`size`）加上一个镜像大小（`virtual size - size`）。

这也不包括容器可以占用磁盘空间的以下其他方式：

- 如果使用`json-file`日志记录驱动程序，则用于日志文件的磁盘空间。如果您的容器生成大量的日志数据并且未配置日志轮转，那么这可能并非易事。
- 容器使用的卷和绑定挂载。
- 容器的配置文件所用的磁盘空间通常很小。
- 写入磁盘的内存（如果启用了交换）。
- 检查点（如果您使用实验性检查点/恢复功能）。

## 写入时复制（CoW）策略
写入时复制是一种共享和复制文件的策略，可最大程度地提高效率。 如果文件或目录位于镜像的较低层中，而另一层（包括可写层）需要对其进行读取访问，则它仅使用现有文件。 另一层第一次需要修改文件时（在构建镜像或运行容器时），将文件复制到该层并进行修改。 这样可以将`I/O`和每个后续层的大小最小化。 这些优点将在下面更深入地说明。

### 分享可以提升较小的镜像
当您使用`docker pull`从存储库中下拉镜像时，或者当您从本地尚不存在的镜像中创建容器时，每一层都会被分别下拉，并存储在Docker的本地存储区域中，在`Linux`主机上通常是`/var/lib/docker/`。 在此示例中，您可以看到这些层被拉出：
```shell
$ docker pull ubuntu:18.04
18.04: Pulling from library/ubuntu
f476d66f5408: Pull complete
8882c27f669e: Pull complete
d9af21273955: Pull complete
f5029279ec12: Pull complete
Digest: sha256:ab6cb8de3ad7bb33e2534677f865008535427390b117d7939193f8d1a6613e34
Status: Downloaded newer image for ubuntu:18.04
```
这些层中的每一层都存储在`Docker`主机本地存储区域内的自己的目录中。 要检查文件系统上的各层，请列出`/var/lib/docker/<storage-driver>`的内容。 本示例使用`overlay2`存储驱动程序：
```shell
$ ls /var/lib/docker/overlay2
16802227a96c24dcbeab5b37821e2b67a9f921749cd9a2e386d5a6d5bc6fc6d3
377d73dbb466e0bc7c9ee23166771b35ebdbe02ef17753d79fd3571d4ce659d7
3f02d96212b03e3383160d31d7c6aeca750d2d8a1879965b89fe8146594c453d
ec1ec45792908e90484f7e629330666e7eee599f08729c93890a7205a6ba35f5
l
```
目录名称与层`ID`不对应（自`Docker 1.10`开始就是如此）。

现在，假设您有两个不同的`Dockerfile`。 您使用第一个镜像创建一个名为`acme/my-base-image:1.0`的镜像。
```Dockerfile
FROM ubuntu:18.04
COPY . /app
```
第二个基于`acme/my-base-image:1.0`，但具有一些附加层：
```Dockerfile
FROM acme/my-base-image:1.0
CMD /app/hello.sh
```
第二个镜像包含第一个镜像中的所有层，再加上带有CMD指令的新层，以及一个可读写容器层。 `Docker`已经具有第一个镜像中的所有层，因此不需要再次将其拉出。 这两个镜像共享它们共有的任何图层。

如果您从两`个Dockerfiles`构建镜像，则可以使用`docker image ls`和`docker history`命令来验证共享层的加密`ID`是否相同。

1. 新建一个目录`cow-test/`并切换到该目录。

2. 在`cow-test/`中，使用以下内容创建一个名为`hello.sh`的新文件：
```shell
#!/bin/sh
echo "Hello world"
```
保存并赋予可执行权限
```shell
chmod +x hello.sh
```

3. 将上面第一个`Dockerfile`的内容复制到一个名为`Dockerfile.base`的新文件中。

4. 将上面第二个`Dockerfile`的内容复制到一个名为`Dockerfile`的新文件中。

5. 在`cow-test/`目录中，构建第一个镜像。 别忘了加入`final`。 在命令中。 这将设置`PATH`，该路径告诉`Docker`在哪里寻找需要添加到镜像的任何文件。
```shell
$ docker build -t acme/my-base-image:1.0 -f Dockerfile.base .
Sending build context to Docker daemon  812.4MB
Step 1/2 : FROM ubuntu:18.04
 ---> d131e0fa2585
Step 2/2 : COPY . /app
 ---> Using cache
 ---> bd09118bcef6
Successfully built bd09118bcef6
Successfully tagged acme/my-base-image:1.0
```

6. 构建第二个镜像
```shell
$ docker build -t acme/my-final-image:1.0 -f Dockerfile .

Sending build context to Docker daemon  4.096kB
Step 1/2 : FROM acme/my-base-image:1.0
 ---> bd09118bcef6
Step 2/2 : CMD /app/hello.sh
 ---> Running in a07b694759ba
 ---> dbf995fc07ff
Removing intermediate container a07b694759ba
Successfully built dbf995fc07ff
Successfully tagged acme/my-final-image:1.0
```

7. 检查镜像大小
```shell
$ docker image ls

REPOSITORY                         TAG                     IMAGE ID            CREATED             SIZE
acme/my-final-image                1.0                     dbf995fc07ff        58 seconds ago      103MB
acme/my-base-image                 1.0                     bd09118bcef6        3 minutes ago       103MB
```

8. 检查组成每个镜像的层：
```shell
$ docker history bd09118bcef6
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
bd09118bcef6        4 minutes ago       /bin/sh -c #(nop) COPY dir:35a7eb158c1504e...   100B                
d131e0fa2585        3 months ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B                  
<missing>           3 months ago        /bin/sh -c mkdir -p /run/systemd && echo '...   7B                  
<missing>           3 months ago        /bin/sh -c sed -i 's/^#\s*\(deb.*universe\...   2.78kB              
<missing>           3 months ago        /bin/sh -c rm -rf /var/lib/apt/lists/*          0B                  
<missing>           3 months ago        /bin/sh -c set -xe   && echo '#!/bin/sh' >...   745B                
<missing>           3 months ago        /bin/sh -c #(nop) ADD file:eef57983bd66e3a...   103MB      
```
```shell
$ docker history dbf995fc07ff

IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
dbf995fc07ff        3 minutes ago       /bin/sh -c #(nop)  CMD ["/bin/sh" "-c" "/a...   0B                  
bd09118bcef6        5 minutes ago       /bin/sh -c #(nop) COPY dir:35a7eb158c1504e...   100B                
d131e0fa2585        3 months ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B                  
<missing>           3 months ago        /bin/sh -c mkdir -p /run/systemd && echo '...   7B                  
<missing>           3 months ago        /bin/sh -c sed -i 's/^#\s*\(deb.*universe\...   2.78kB              
<missing>           3 months ago        /bin/sh -c rm -rf /var/lib/apt/lists/*          0B                  
<missing>           3 months ago        /bin/sh -c set -xe   && echo '#!/bin/sh' >...   745B                
<missing>           3 months ago        /bin/sh -c #(nop) ADD file:eef57983bd66e3a...   103MB  
```

请注意，除了第二个镜像的顶层之外，所有层都是相同的。所有其他层在两个镜像之间共享，并且仅在`/var/lib/docker/`中存储一次。实际上，新层根本没有占用任何空间，因为它不会更改任何文件，而只是运行命令。

> 注意：`docker history`输出中的`<missing>`行表明这些层是在另一个系统上构建的，并且在本地不可用。这可以忽略。

### 复制使容器高效
启动容器时，将在其他层之上添加一个薄的可写容器层。容器对文件系统所做的任何更改都存储在此处。容器未更改的任何文件都不会复制到此可写层。这意味着可写层尽可能小。

修改容器中的现有文件后，存储驱动程序将执行写时复制操作。涉及的具体步骤取决于特定的存储驱动程序。对于`aufs`，`overlay`和`overlay2`驱动程序，写时复制操作遵循以下大致步骤：

- 在镜像层中搜索要更新的文件。该过程从最新层开始，一次向下一层到基础层。找到结果后，会将它们添加到缓存中以加快将来的操作。

- 对找到的文件的第一个副本执行`copy_up`操作，以将文件复制到容器的可写层。

- 对该文件的副本进行了任何修改，并且容器看不到存在于较低层中的文件的只读副本。

`Btrfs`，`ZFS`和其他驱动程序以不同方式处理写时复制。您可以在稍后的详细说明中阅读有关这些驱动程序方法的更多信息。

写入大量数据的容器比不写入数据的容器消耗更多的空间。这是因为大多数写操作会占用容器的可写薄顶层中的新空间。

> 注意：对于繁重的应用程序，不应将数据存储在容器中。取而代之的是使用`Docker`卷，它们独立于正在运行的容器，并且旨在提高`I/O`效率。此外，卷可以在容器之间共享，而不会增加容器可写层的大小。

`copy_u`p操作可能会导致明显的性能开销。该开销因所使用的存储驱动程序而异。大文件，许多层和深层目录树可以使影响更加明显。事实是，每个`copy_up`操作仅在第一次修改给定文件时才发生，这可以缓解这种情况。

为了验证写时复制的工作方式，以下过程根据我们先前构建的`acme/my-final-image:1.0`镜像启动了5个容器，并检查它们占用了多少空间。

> 注意：此步骤不适用于`Mac`的`Docker`桌面或`Windows`的`Docker`桌面。

1. 在`Docker`主机上的终端上，运行以下`docker run`命令。最后的字符串是每个容器的`ID`。
```shell
$ docker run -dit --name my_container_1 acme/my-final-image:1.0 bash \
  && docker run -dit --name my_container_2 acme/my-final-image:1.0 bash \
  && docker run -dit --name my_container_3 acme/my-final-image:1.0 bash \
  && docker run -dit --name my_container_4 acme/my-final-image:1.0 bash \
  && docker run -dit --name my_container_5 acme/my-final-image:1.0 bash

  c36785c423ec7e0422b2af7364a7ba4da6146cbba7981a0951fcc3fa0430c409
  dcad7101795e4206e637d9358a818e5c32e13b349e62b00bf05cd5a4343ea513
  1e7264576d78a3134fbaf7829bc24b1d96017cf2bc046b7cd8b08b5775c33d0c
  38fa94212a419a082e6a6b87a8e2ec4a44dd327d7069b85892a707e3fc818544
  1a174fc216cccf18ec7d4fe14e008e30130b11ede0f0f94a87982e310cf2e765
```

2. 使用`docker ps`命令确认五个容器都在运行
```shell
CONTAINER ID      IMAGE                     COMMAND     CREATED              STATUS              PORTS      NAMES
1a174fc216cc      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_5
38fa94212a41      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_4
1e7264576d78      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_3
dcad7101795e      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_2
c36785c423ec      acme/my-final-image:1.0   "bash"      About a minute ago   Up About a minute              my_container_1
```

3. 列出本地存储区的内容。
```shell
$ sudo ls /var/lib/docker/containers

1a174fc216cccf18ec7d4fe14e008e30130b11ede0f0f94a87982e310cf2e765
1e7264576d78a3134fbaf7829bc24b1d96017cf2bc046b7cd8b08b5775c33d0c
38fa94212a419a082e6a6b87a8e2ec4a44dd327d7069b85892a707e3fc818544
c36785c423ec7e0422b2af7364a7ba4da6146cbba7981a0951fcc3fa0430c409
dcad7101795e4206e637d9358a818e5c32e13b349e62b00bf05cd5a4343ea513
```

4. 查看它们的大小
```shell
$ sudo du -sh /var/lib/docker/containers/*

32K  /var/lib/docker/containers/1a174fc216cccf18ec7d4fe14e008e30130b11ede0f0f94a87982e310cf2e765
32K  /var/lib/docker/containers/1e7264576d78a3134fbaf7829bc24b1d96017cf2bc046b7cd8b08b5775c33d0c
32K  /var/lib/docker/containers/38fa94212a419a082e6a6b87a8e2ec4a44dd327d7069b85892a707e3fc818544
32K  /var/lib/docker/containers/c36785c423ec7e0422b2af7364a7ba4da6146cbba7981a0951fcc3fa0430c409
32K  /var/lib/docker/containers/dcad7101795e4206e637d9358a818e5c32e13b349e62b00bf05cd5a4343ea513
```

这些容器中的每个仅占用文件系统上`32k`的空间。

写入时复制不仅可以节省空间，还可以缩短启动时间。 当启动一个容器（或同一镜像中的多个容器）时，`Docker`只需要创建可写的容器薄层。

如果`Docker`每次启动新容器都必须制作基础镜像堆栈的完整副本，则容器启动时间和使用的磁盘空间将大大增加。 这将类似于虚拟机的工作方式，每个虚拟机具有一个或多个虚拟磁盘。