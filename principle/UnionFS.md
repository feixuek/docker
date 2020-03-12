# `UnionFS`：联合文件系统

联合文件系统或`UnionFS`是通过创建镜像进行操作的文件系统，使其非常轻便且快速。 `Docker Engine`使用`UnionFS`为容器提供构建模块。 `Docker Engine`可以使用多个`UnionFS`变体，包括`AUFS`，`btrfs`，`vfs`，`overlay2`和`DeviceMapper`。

## `rootfs`：根文件系统
在`Namespace`中提到过，`Linux`操作系统提供了很多命名空间，其中包括`Mount Namespace`，执行了`Mount Namespace`之后，进程实际使用的其实还是宿主机的文件系统，要想起到隔离的作用，还需要进行挂载操作，这一步非常重要。

> 挂载操作使用`chroot`系统调用来完成。

而挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，就是所谓的"容器镜像"。即：`rootfs`（根文件系统）。例如`centos`镜像，包含`centos`操作系统所需的文件系统，但是内核依然是宿主机的内核。

> 如果应用程序要配置内核参数，加载额外的内核模块，以及和内核的直接交互，需要注意，对于该宿主机来说，其上的所有容器这些操作属于全局变量。

以`18.06.1-ce`为例（`centos`操作系统，`overlay2`文件系统），可以查看`centos`镜像的`rootfs`。
```bash
docker image inspect centos:latest
```
你将看到
```json
...
"GraphDriver": {
    "Data": {
        "MergedDir": "/var/lib/docker/overlay2/19b6693ab21861cf8e945f896f9f4ea75247ead0c440b42384e1a62e301af368/merged",
        "UpperDir": "/var/lib/docker/overlay2/19b6693ab21861cf8e945f896f9f4ea75247ead0c440b42384e1a62e301af368/diff",
         "WorkDir": "/var/lib/docker/overlay2/19b6693ab21861cf8e945f896f9f4ea75247ead0c440b42384e1a62e301af368/work"
         },
     "Name": "overlay2"
},
"RootFS": {
     "Type": "layers",
     "Layers": [
        "sha256:0683de2821778aa9546bf3d3e6944df779daba1582631b7ea3517bb36f9e4007"
     ]
},
...
```
```bash
ls -al /var/lib/docker/overlay2/19b6693ab21861cf8e945f896f9f4ea75247ead0c440b42384e1a62e301af368/diff
```
可以看到`centos`的`rootfs`
```bash
total 60
lrwxrwxrwx  1 root root    7 May 11  2019 bin -> usr/bin
drwxr-xr-x  2 root root 4096 Jan 14 05:48 dev
drwxr-xr-x 51 root root 4096 Jan 14 05:49 etc
drwxr-xr-x  2 root root 4096 May 11  2019 home
lrwxrwxrwx  1 root root    7 May 11  2019 lib -> usr/lib
lrwxrwxrwx  1 root root    9 May 11  2019 lib64 -> usr/lib64
drwx------  2 root root 4096 Jan 14 05:48 lost+found
drwxr-xr-x  2 root root 4096 May 11  2019 media
drwxr-xr-x  2 root root 4096 May 11  2019 mnt
drwxr-xr-x  2 root root 4096 May 11  2019 opt
drwxr-xr-x  2 root root 4096 Jan 14 05:48 proc
dr-xr-x---  2 root root 4096 Jan 14 05:49 root
drwxr-xr-x 11 root root 4096 Jan 14 05:49 run
lrwxrwxrwx  1 root root    8 May 11  2019 sbin -> usr/sbin
drwxr-xr-x  2 root root 4096 May 11  2019 srv
drwxr-xr-x  2 root root 4096 Jan 14 05:48 sys
drwxrwxrwt  7 root root 4096 Jan 14 05:49 tmp
drwxr-xr-x 12 root root 4096 Jan 14 05:49 usr
drwxr-xr-x 20 root root 4096 Jan 14 05:49 var
```

## `UnionFS`：联合文件系统

`Union File System`也叫`UnionFS`，最主要的功能是将多个不同位置的目录联合挂载（`union mount`）到同一个目录下。

`Docker`如何使用`UnionFS`呢？以`AUFS`为例，是`UnionFS`的改进，采用分层的概念，将`rootfs`和依赖`rootfs`的其他事项整合起来，在完成目录联合挂载的同时，又不会对`rootfs`造成修改，保证了环境的一致性。

容器和镜像的联合文件系统的区别在于，镜像的联合文件系统都是只读的，而容器的联合文件系统则会在镜像的联合文件系统上新增一个可读写层，容器操作产生的数据都在该可读写层中。

## `Volume`实现原理

在`rootfs`根文件系统提到过，执行`Mount Namespace`之后，容器实际使用的其实还是宿主机的文件系统，要想起到隔离的作用，还需要进行挂载操作，而挂载的即`rootfs`，那么在挂载绑定之前，将宿主机上的目录使用`Linux`的绑定挂载机制挂载到指定的目录上，此时容器使用的既是宿主机上的文件目录，这便是`Volume`的实现原理，如果在创建容器时没有显示指定目录，那么`docker`会在`/var/lib/docker/volumes`下新建一个临时`volume`供容器使用。