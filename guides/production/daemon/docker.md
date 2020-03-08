# 配置`Docker`守护程序并对其进行故障排除

成功安装并启动`Docker`之后，`dockerd`守护程序将以其默认配置运行。本主题显示如何自定义配置，手动启动守护程序以及在遇到问题时对守护程序进行故障排除和调试。

## 使用操作系统实用程序启动守护程序
在典型安装中，`Docker`守护进程由系统实用程序启动，而不是由用户手动启动。这使得在机器重启时自动启动`Docker`更容易。

启动`Docker`的命令取决于您的操作系统。在`Install Docker`下检查正确的页面。要将`Docker`配置为在系统启动时自动启动，请参阅将 `Docker`配置为在启动时启动。

## 手动启动守护程序
如果您不想使用系统实用程序来管理`Docker`守护程序，或者只想进行测试，则可以使用以下`dockerd` 命令手动运行它。您可能需要使用`sudo`，具体取决于您的操作系统配置。

当您以这种方式启动`Docker`时，它在前台运行，并将其日志直接发送到您的终端。
```shell
$ dockerd

INFO[0000] +job init_networkdriver()
INFO[0000] +job serveapi(unix:///var/run/docker.sock)
INFO[0000] Listening for HTTP on unix (/var/run/docker.sock)
```
要在手动启动`Docker`时停止它，请在终端中发出`Ctrl+C` 。

## 配置守护进程
有两种方法可以配置`Docker`守护程序：

- 使用`JSON`配置文件。这是首选选项，因为它将所有配置都放在一个位置。
- 启动时使用标志`dockerd`。

只要您未同时在标志和`JSON`文件中指定相同的选项，就可以将这两个选项一起使用。如果发生这种情况，`Docker`守护程序将无法启动并显示错误消息。

要使用`JSON`文件配置`Docker`守护程序，请`/etc/docker/daemon.json`在Linux系统或`C:\ProgramData\docker\config\daemon.json` `Windows` 上创建一个文件 。在`MacOS`上，转到任务栏中的`whale>首选项>守护程序>高级`。

配置文件如下所示：
```json
{
  "debug": true,
  "tls": true,
  "tlscert": "/var/docker/server.pem",
  "tlskey": "/var/docker/serverkey.pem",
  "hosts": ["tcp://192.168.59.3:2376"]
}
```
通过此配置，`Docker`守护程序以调试模式运行，使用`TLS`，并侦听路由到`192.168.59.3`的流量到端口`2376`。您可以在[`dockerd`参考文档](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)中了解可用的配置选项

您还可以手动启动`Docker`守护程序并使用标志对其进行配置。这对于解决问题很有用。

这是一个如何使用与上面相同的配置手动启动`Docker`守护程序的示例：
```shell
dockerd --debug \
  --tls=true \
  --tlscert=/var/docker/server.pem \
  --tlskey=/var/docker/serverkey.pem \
  --host tcp://192.168.59.3:2376
```
您可以在`dockerd`参考文档中或通过运行以下命令了解可用的配置选项 ：
```shell
dockerd --help
```
整个`Docker`文档中讨论了许多特定的配置选项。接下来要去的地方包括：

- [自动启动容器](https://docs.docker.com/config/containers/start-containers-automatically/)
- [限制容器的资源](https://docs.docker.com/config/containers/resource_constraints/)
- [配置存储驱动程序](https://docs.docker.com/storage/storagedriver/select-storage-driver/)
- [容器安全](https://docs.docker.com/engine/security/)

## `Docker`守护程序目录
`Docker`守护程序将所有数据保存在一个目录中。这将跟踪与`Docker`相关的所有内容，包括容器，镜像，卷，服务定义和机密。

默认情况下，该目录为：

- `/var/lib/docker` 在`Linux`上。
- `C:\ProgramData\docker` 在`Windows`上。

您可以使用`data-root`配置选项将`Docker`守护程序配置为使用其他目录 。

由于`Docker`守护程序的状态保留在此目录中，因此请确保为每个守护程序使用专用目录。如果两个守护程序共享同一目录（例如，`NFS`共享），则将遇到难以解决的错误。

## 对守护程序进行故障排除
您可以在守护程序上启用调试功能，以了解该守护程序的运行时活动并帮助进行故障排除。如果守护程序是完全无响应的，您还 可以通过将信号发送到`Docker`守护程序来强制将所有线程的完整堆栈跟踪添加到守护程序日志中`SIGUSR`。

### `daemon.json`和启动脚本之间矛盾排查
如果您使用`daemon.json`文件并`dockerd `手动或使用启动脚本将选项传递给命令，并且这些选项发生冲突，则`Docker`无法启动，并显示以下错误：
```shell
unable to configure the Docker daemon with file /etc/docker/daemon.json:
the following directives are specified both as a flag and in the configuration
file: hosts: (from flag: [unix:///var/run/docker.sock], from file: [tcp://127.0.0.1:2376])
```
如果看到类似于此错误的错误，并且正在使用标志手动启动守护程序，则可能需要调整标志或`daemon.json`来消除冲突。

> 注意：如果看到此特定错误，请继续进行下一部分以解决。

如果要使用操作系统的`init`脚本启动`Docker`，则可能需要以特定于操作系统的方式覆盖这些脚本中的默认值。

#### 将`DAEMON.JSON`中的`HOSTS键`与`SYSTEMD`一起使用
难以解决的配置冲突的一个显着示例是，您想要指定与默认值不同的守护程序地址。`Docker`默认情况下侦听套接字。在使用`Debian`和`Ubuntu`的系统上`systemd`，这意味着`-H`启动时始终使用主机标志`dockerd`。如果在中指定 `hosts`条目，则将`daemon.json`导致配置冲突（如以上消息中所示），并且`Docker`无法启动。

要变通解决此问题，创建`/etc/systemd/system/docker.service.d/docker.conf`具有以下内容的新文件，以删除`-H`默认情况下启动守护程序时使用的参数。
```shell
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
```
有时您可能需要`systemd`使用`Docker` 进行配置，例如 配置`HTTP`或`HTTPS proxy`。

> 注意：如果您覆盖此选项，然后`hosts`在手动启动`Docker`时未在`daemon.json` 或`-H`标志中指定条目，则`Docker`无法启动。

`sudo systemctl daemon-reload`在尝试启动`Docker`之前运行。如果`Docker`成功启动，则它现在正在侦听由`hostskey daemon.json`而不是套接字指定的`IP`地址。

> 重要提示：`Windows`版`Docker`桌面或`Mac`版`Docker`桌面不支持`hostsin`中的设置`daemon.json`。

### 内存不足异常（OOME）
如果您的容器尝试使用的内存超过系统可用的内存，则可能会遇到内存不足异常（`OOME`），并且内核`OOM`杀手可能会杀死容器或`Docker`守护程序。为防止这种情况的发生，请确保您的应用程序在具有足够内存的主机上运行，​​请参阅了解内存不足 的风险。

### 阅读日志
守护程序日志可以帮助您诊断问题。日志可能保存在几个位置之一，具体取决于操作系统配置和使用的日志记录子系统：

| 操作系统 |	位置 |
| --- | --- |
| `RHEL，Oracle Linux` |	 `/var/log/messages` |
| `Debian` |	`/var/log/daemon.log` |
| `Ubuntu 16.04 +，CentOS` |	使用命令 `journalctl -u docker.service`|
| `Ubuntu 14.10-` |	`/var/log/upstart/docker.log`
| `macOS（Docker 18.01+）` | `	~/Library/Containers/com.docker.docker/Data/vms/0/console-ring` |
| `macOS（Docker <18.01）` |	`~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/console-ring`|
| `Windows` |	`AppData\Local` |

### 启用调试
有两种启用调试的方法。推荐的方法是将d`aemon.json`文件中的`debug`密钥设置`true`。该方法适用于每个`Docker`平台。

1. 编辑`daemon.json`文件，该文件通常位于中`/etc/docker/`。如果该文件尚不存在，则可能需要创建它。在`macOS`或`Windows`上，请勿直接编辑文件。而是转到`Preferences/Daemon/Advanced`。

2. 如果文件为空，请添加以下内容：
```json
{
  "debug": true
}
```
如果文件已经包含`JSON`，则只需添加`key` `"debug": true`即可，如果不是结束括号之前的最后一行，请小心在行末添加逗号。还要验证是否`log-level`已设置密钥，将其设置为`info`还是`debug`。`info`是默认的，和可能的值是`debug`，`info`，`warn`，`error`，`fatal`。

3. 向守护程序发送`HUP`信号以使其重新加载其配置。在`Linux`主机上，使用以下命令。
```shell
$ sudo kill -SIGHUP $(pidof dockerd)
```
在`Windows`主机上，重新启动`Docker`。

除了遵循此过程之外，您还可以停止`Docker`守护程序，并使用`debug`标志手动重新启动它`-D`。但是，这可能会导致`Docker`在与主机启动脚本创建的环境不同的环境下重新启动，这可能会使调试更加困难。

### 强制堆栈跟踪记录
如果守护程序无响应，则可以通过向守护程序发送`SIGUSR1`信号来强制记录完整的堆栈跟踪。

- `Linux`：
```shell
$ sudo kill -SIGUSR1 $(pidof dockerd)
```
- `Windows Server`：

下载`docker-signal`。

获取`dockerd`的进程`ID` `Get-Process dockerd`。

运行带有标志的可执行文件`--pid=<PID of daemon>`。

这将强制记录堆栈跟踪，但不会停止守护程序。守护程序日志显示堆栈跟踪或包含堆栈跟踪的文件的路径（如果已将其记录到文件中）。

在处理`SIGUSR1`信号并将堆栈跟踪信息转储到日志之后，守护程序将继续运行。堆栈跟踪可用于确定守护程序中所有`goroutine`和线程的状态。

### 查看堆栈跟踪
可以使用以下方法之一查看`Docker`守护程序日志：

- 通过`journalctl -u docker.service`在`Linux`系统上使用`systemctl`
- `/var/log/messages`，`/var/log/daemon.log`或`/var/log/docker.log`旧版`Linux`系统上

> 注意：无法在`Mac`的`Docker`桌面或`Windows`的`Docker`桌面上手动生成堆栈跟踪。但是，您可以单击`Docker`任务栏图标，然后选择“ 诊断和反馈”，以在遇到问题时将信息发送给`Docker`。

在`Docker`日志中查找如下消息：
```shell
...goroutine stacks written to /var/run/docker/goroutine-stacks-2017-06-02T193336z.log
...daemon datastructure dump written to /var/run/docker/daemon-data-2017-06-02T193336z.log
```
`Docker`保存这些堆栈跟踪和转储的位置取决于您的操作系统和配置。有时您可以直接从堆栈跟踪和转储中获得有用的诊断信息。否则，您可以将此信息提供给`Docker`，以帮助诊断问题。

## 检查`Docker`是否正在运行
独立于操作系统的检查`Docker`是否正在运行的方法是使用`docker info`命令询问`Docker` 。

您也可以使用操作系统实用程序，例如 `sudo systemctl is-active docker`或`sudo status docker`或 `sudo service docker status`，或使用`Windows`实用程序检查服务状态。

最后，您可以`dockerd`使用`ps`或之类的命令在进程的进程列表中检入`top`。