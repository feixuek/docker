# 解决卷错误

本主题讨论使用`Docker`卷或绑定挂载时可能发生的错误。

## 错误：无法删除文件系统
某些基于容器的实用程序（例如`Google cAdvisor`）将`Docker`系统目录（例如`/var/lib/docker/`）安装到容器中。 例如，`cadvisor`文档会指示您按以下方式运行`cadvisor`容器：
```shell
$ sudo docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  google/cadvisor:latest
```
当您绑定挂载`/var/lib/docker/`时，这将有效地将所有其他正在运行的容器的所有资源挂载为文件系统，该文件系统将挂载`/var/lib/docker/`。 当您尝试删除这些容器中的任何一个时，删除尝试可能会失败，并显示如下错误：
```shell
Error: Unable to remove filesystem for
74bef250361c7817bee19349c93139621b272bc8f654ae112dd4eb9652af9515:
remove /var/lib/docker/containers/74bef250361c7817bee19349c93139621b272bc8f654ae112dd4eb9652af9515/shm:
Device or resource busy
```
如果绑定挂载`/var/lib/docker/`的容器在`/var/lib/docker/`内的文件系统句柄上使用`statfs`或`fstatfs`，并且没有关闭它们，则会发生此问题。

通常，我们建议不要以这种方式绑定挂载`/var/lib/docker`。 但是，`cAdvisor`需要此绑定挂载才能实现核心功能。

如果不确定哪个进程导致错误中提到的路径繁忙并阻止其删除，则可以使用`lsof`命令查找其进程。 例如，对于上面的错误：

```shell
$ sudo lsof /var/lib/docker/containers/74bef250361c7817bee19349c93139621b272bc8f654ae112dd4eb9652af9515/shm
```
  
要变通解决此问题，请停止将`/var/lib/docker`绑定挂载的容器，然后再次尝试删除另一个容器。