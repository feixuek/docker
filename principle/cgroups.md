# `Cgroups`：控制组

`Linux`上的`Docker`引擎还依赖于另一种称为控制组（`Cgroups`）的技术。 `Cgroup`将应用程序限制为一组特定的资源。控制组允许`Docker Engine`将可用的硬件资源共享给容器，并有选择地实施限制和约束。例如，您可以限制特定容器可用的内存。

## `Cgroup`机制

基于`Linux Namespace`的隔离机制相比于虚拟化技术也有很多不足之处，其中最主要的问题就是：隔离得不彻底。

- 容器的本质是进程，多个个容器之间使用的就还是同一个宿主机的操作系统内核。
- 在`Linux`内核中，有很多资源和对象是不能被`Namespace`化的，最典型的例子就是：时间。因此如果一个容器修改了时间，那么整个宿主机的时间都将发生变化。

因此需要一种机制限制容器使用宿主机的资源，比如`CPU`和内存，该机制就是`cgroup`。

在`Linux`中，`Cgroups`给用户暴露出来的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的`/sys/fs/cgroup`路径下。

### 理解`cgroup`机制示例

1. 查看`Cgroups`暴露出来的文件系统
```bash
mount -t cgroup
```
你将看到下面类似的输出：
```bash
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_prio,net_cls)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct,cpu)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
```
可以看到，在`/sys/fs/cgroup`下面有很多诸如`cpuset`、`cpu`、 `memory`这样的子目录，也叫子系统。这些都是我这台机器当前可以被`Cgroups`进行限制的资源种类。而在子系统对应的资源种类下，你就可以看到该类资源具体可以被限制的方法。

2. 查看`CPU`子系统的配置文件

```bash
ls /sys/fs/cgroup/cpu
```

```bash
cgroup.clone_children  cgroup.sane_behavior  cpuacct.usage_percpu  cpu.rt_period_us   cpu.stat           release_agent  user.slice
cgroup.event_control   cpuacct.stat          cpu.cfs_period_us     cpu.rt_runtime_us  docker             system.slice
cgroup.procs           cpuacct.usage         cpu.cfs_quota_us      cpu.shares         notify_on_release  tasks
```

其中，`cfs_period_us`和`cfs_quota_us`这两个组合使用，可以用来限制进程在长度为`cfs_period`的一段时间内，只能被分配到总量为`cfs_quota`的`CPU`时间。

3. 使用`cgroup`
```bash
cd /sys/fs/cgroup/cpu
mkdir test 
cd test
ls -al
``` 
你将看到系统自动创建的资源限制文件
```bash
drwxr-xr-x 2 root root 0 Mar 12 11:17 .
drwxr-xr-x 6 root root 0 Oct 13 01:16 ..
-rw-r--r-- 1 root root 0 Mar 12 11:17 cgroup.clone_children
--w--w--w- 1 root root 0 Mar 12 11:17 cgroup.event_control
-rw-r--r-- 1 root root 0 Mar 12 11:17 cgroup.procs
-r--r--r-- 1 root root 0 Mar 12 11:17 cpuacct.stat
-rw-r--r-- 1 root root 0 Mar 12 11:17 cpuacct.usage
-r--r--r-- 1 root root 0 Mar 12 11:17 cpuacct.usage_percpu
-rw-r--r-- 1 root root 0 Mar 12 11:17 cpu.cfs_period_us
-rw-r--r-- 1 root root 0 Mar 12 11:17 cpu.cfs_quota_us
-rw-r--r-- 1 root root 0 Mar 12 11:17 cpu.rt_period_us
-rw-r--r-- 1 root root 0 Mar 12 11:17 cpu.rt_runtime_us
-rw-r--r-- 1 root root 0 Mar 12 11:17 cpu.shares
-r--r--r-- 1 root root 0 Mar 12 11:17 cpu.stat
-rw-r--r-- 1 root root 0 Mar 12 11:17 notify_on_release
-rw-r--r-- 1 root root 0 Mar 12 11:17 tasks
```

4. 在后台执行一个死循环脚本
```bash
while : ; do : ; done &
[1] 32395
```
查看cpu占用
```bash
top | grep 32395
```
可以看到cpu基本被打满了
```bash
32395 root      20   0  114664   1712    144 R  93.8  0.0   0:26.16 bash                                                                                   
32395 root      20   0  114664   1712    144 R 100.0  0.0   0:29.17 bash                                                                                   
32395 root      20   0  114664   1712    144 R  99.7  0.0   0:32.17 bash                                                                                   
32395 root      20   0  114664   1712    144 R 100.0  0.0   0:35.17 bash
```
5. 我们看一下`test`文件下的`cfs_period_us`和`cfs_quota_us`两个文件内容
```bash
cat cpu.cfs_period_us 
100000
cat cpu.cfs_quota_us 
-1
```
`-1`即没有任何限制，因此进程`32395`会将`cpu`吃满。

5. 修改进程`32395`的`cpu`限制
```bash
echo 20000 > cpu.cfs_quota_us
echo 32395 > tasks
```
修改之后变为在`100000us`内进程`32395`只能占用`20000us`的`cpu`，即`20%`占比。

6. 再次查看`cpu`占用情况
```bash
top | grep 32395
```
你将看到
```bash
32395 root      20   0  114664   1720    136 R  20.0  0.0   0:08.54 bash                                                                                   
32395 root      20   0  114664   1720    136 R  19.9  0.0   0:09.14 bash                                                                                   
32395 root      20   0  114664   1720    136 R  19.9  0.0   0:09.74 bash 
```
此时进程`32395`的`cpu`占用只有20%了。

7. 最后`kill`进程`32395`
```bash
kill -9 32395
```

### 总结
从上面的小例子我们看到`CPU`子系统对进程的限制能力，除`CPU`子系统外，`Cgroups`的每一项子系统都有其独有的资源限制能力，比如：

- `blkio`，为块设备设定`I/O`限制，一般用于磁盘等设备；
- `cpuset`，为进程分配单独的`CPU`核和对应的内存节点；
- `memory`，为进程设定内存使用的限制

以上介绍了`Cgroups`的原理，容器的本质是进程，因此`docker`对每个容器的资源限制可以通过相同的方式来实现。