# 自动启动容器

`Docker`提供了重启策略 来控制您的容器在退出时还是在`Docker`重新启动时自动启动。重新启动策略确保链接的容器以正确的顺序启动。`Docker`建议您使用重启策略，并避免使用流程管理器来启动容器。

重新启动策略`--live-restore`与`dockerd` 命令标志不同。使用`--live-restore`可以使您的容器在`Docker`升级期间保持运行，尽管网络和用户输入被中断。

## 使用重启策略
要为容器配置重启策略，请在使用`docker run`命令时使用`--restart`标志。`--restart`标志的值可以是以下任意值：

| 标志	| 描述 |
| --- | --- |
| `no`	| 不要自动重启容器。（默认）|
| `on-failure` | 如果容器由于错误而退出，请重新启动容器，该错误表示为非零退出代码。 |
| `always`	| 如果容器停止，请务必重新启动它。如果已手动停止，则仅在`Docker`守护程序重新启动或容器本身手动重新启动时才重新启动。
| `unless-stopped`	| 与`always`相似，除了在容器停止（手动或其他方式）时，即使重新启动`Docker`守护程序也不会重新启动容器。

以下示例启动`Redis`容器并将其配置为始终重新启动，除非该容器已明确停止或重新启动`Docker`。
```shell
$ docker run -dit --restart unless-stopped redis
```
### 重新启动策略详细信息
使用重新启动策略时，请记住以下几点：

- 重新启动策略仅在容器成功启动后才生效。在这种情况下，成功启动意味着该容器已启动至少10秒钟，并且`Docker`已开始对其进行监视。这样可以防止根本无法启动的容器进入重启循环。

- 如果手动停止容器，则其重新启动策略将被忽略，直到Docker守护程序重新启动或手动重新启动容器为止。这是防止重新启动循环的另一种尝试。

- 重新启动策略仅适用于容器。`Swarm`服务的重启策略配置不同。

## 使用流程管理器
如果重新启动的政策不适合您的需求，比如当容器之外的进程取决于`Docker`容器，您可以使用进程管理器等 `upstat`， `systemd`，或`supervisor`代替。

> 警告：请勿尝试将`Docker`重新启动策略与主机级流程管理器结合使用，因为这会产生冲突。

要使用流程管理器，请将其配置为使用通常用于手动启动容器的相同命令`docker start`或`docker service`命令来启动您的容器或服务。

### 在容器内使用过程管理器
流程管理器还可以在容器内运行，以检查某个流程是否正在运行，如果没有，则启动/重新启动它。

> 警告：这些不支持`Docker`，仅监视容器内的操作系统进程。

`Docker`不推荐这种方法，因为它依赖于平台，甚至在给定`Linux`发行版的不同版本中也有所不同。