# `NameSpace`：命名空间

`Docker`使用一种称为命名空间的技术来提供称为容器的隔离工作区。运行容器时，`Docker`会为该容器创建一组命名空间。

这些命名空间提供了一层隔离。容器的每个方面都在单独的命名空间中运行，并且其访问仅限于该命名空间。

`Docker Engine`在`Linux`上使用以下命名空间：

- `pid`名称空间：进程隔离（`PID`：进程`ID`）。
- `net`名称空间：管理网络接口（`NET`：网络）。
- `ipc`名称空间：管理对`IPC`资源的访问（`IPC`：进程间通信）。
- `mnt`名称空间：管理文件系统安装点（`MNT`：安装）。
- `uts`名称空间：隔离内核和版本标识符。 （`UTS`：`Unix`时间共享系统）。

## `Namespace`机制

`Linux`里面的`Namespace`机制，其实只是`Linux`创建新进程的一个可选参数。我们知道，在`Linux`系统中创建线程的系统调用是`clone()`，比如：

```C
int pid = clone(main_function, stack_size, SIGCHLD, NULL);
```

这个系统调用就会为我们创建一个新的进程，并且返回它的进程号`pid`。

而当我们用`clone()`系统调用创建一个新进程时，就可以在参数中指定`CLONE_NEWPID`参数，比如：

```C
int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL);
```

这时，新创建的这个进程将会"看到"一个全新的进程空间，在这个进程空间里，它的`PID`是1。之所以说"看到"，是因为在宿主机真实的进程空间里，这个进程的`PID`还是真实的数值，比如100。

当然，我们还可以多次执行上面的`clone()`调用，这样就会创建多个`PID Namespace`，而每个`Namespace`里的应用进程，都会认为自己是当前容器里的第`1`号进程，它们既看不到宿主机里真正的进程空间，也看不到其他`PID Namespace`里的具体情况。

而除了`PID Namespace`，`Linux`操作系统还提供了`Mount`、`UTS`、`IPC`、`Network`和`User`这些`Namespace`，这就是`Linux`容器最基本的实现原理。

### `docker exec`实现原理
查看`f678aeb5b138`宿主机上的进程号
```bash
docker inspect --format '{{.State.Pid}}' f678aeb5b138

7658
```
查看`Pid=7658`的命名空间
```bash
ls -l /proc/7658/ns

lrwxrwxrwx 1 root root 0 Mar 12 16:05 ipc -> ipc:[4026532327]
lrwxrwxrwx 1 root root 0 Mar 12 16:05 mnt -> mnt:[4026532325]
lrwxrwxrwx 1 root root 0 Mar 12 15:43 net -> net:[4026532330]
lrwxrwxrwx 1 root root 0 Mar 12 16:05 pid -> pid:[4026532328]
lrwxrwxrwx 1 root root 0 Mar 12 16:05 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Mar 12 16:05 uts -> uts:[4026532326]
```
可以看到，一个进程的每种`Linux Namespace`，都在它对应的`/proc/[进程号]/ns`下有一个对应的虚拟文件，并且链接到一个真实的`Namespace文件上，此时使用`setns`系统调用即可进入该命名空间，这便是`docker exec`的实现原理。