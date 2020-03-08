# 在容器中运行多个服务

容器的主要运行过程是`Dockerfile`末尾的`ENTRYPOINT`和或`CMD`。通常建议您通过每个容器使用一项服务来分离关注区域。该服务可以派生到多个进程中（例如，`Apache Web`服务器启动多个工作进程）。可以有多个进程，但是要从`Docke`r中获得最大收益，请避免一个容器负责整个应用程序的多个方面。您可以使用用户定义的网络和共享卷连接多个容器。

容器的主要过程负责管理它启动的所有过程。在某些情况下，主流程设计不当，并且在容器退出时无法优雅地处理“收获”（停止）子流程。如果您的进程属于此类别，则可以`--init`在运行容器时使用该选项。该`--init`标志将一个微小的`init`进程作为主要进程插入到容器中，并在容器退出时处理所有进程的收割。以这种方式处理此类流程优于使用成熟的初始化流程（如`sysvinit`，`upstart`）或`systemd`处理容器内的流程生命周期。

如果您需要在一个容器中运行多个服务，则可以通过几种不同的方式来完成此操作。

- 将所有命令放入包装器脚本中，其中包含测试和调试信息。以您的身份运行包装脚本`CMD`。这是一个非常幼稚的例子。首先，包装脚本：
```shell
#!/bin/bash

# Start the first process
./my_first_process -D
status=$?
if [ $status -ne 0 ]; then
  echo "Failed to start my_first_process: $status"
  exit $status
fi

# Start the second process
./my_second_process -D
status=$?
if [ $status -ne 0 ]; then
  echo "Failed to start my_second_process: $status"
  exit $status
fi

# Naive check runs checks once a minute to see if either of the processes exited.
# This illustrates part of the heavy lifting you need to do if you want to run
# more than one service in a container. The container exits with an error
# if it detects that either of the processes has exited.
# Otherwise it loops forever, waking up every 60 seconds

while sleep 60; do
  ps aux |grep my_first_process |grep -q -v grep
  PROCESS_1_STATUS=$?
  ps aux |grep my_second_process |grep -q -v grep
  PROCESS_2_STATUS=$?
  # If the greps above find anything, they exit with 0 status
  # If they are not both 0, then something is wrong
  if [ $PROCESS_1_STATUS -ne 0 -o $PROCESS_2_STATUS -ne 0 ]; then
    echo "One of the processes has already exited."
    exit 1
  fi
done
```
接下来，`Dockerfile`：
```Dockerfile
FROM ubuntu:latest
COPY my_first_process my_first_process
COPY my_second_process my_second_process
COPY my_wrapper_script.sh my_wrapper_script.sh
CMD ./my_wrapper_script.sh
```

- 如果您有一个主要流程需要首先启动并保持运行，但是您暂时需要运行其他一些流程（可能与主要流程进行交互），则可以使用`bash`的作业控制来简化这一过程。首先，包装脚本：
```shell
#!/bin/bash

# turn on bash's job control
set -m

# Start the primary process and put it in the background
./my_main_process &

# Start the helper process
./my_helper_process

# the my_helper_process might need to know how to wait on the
# primary process to start before it does its work and returns


# now we bring the primary process back into the foreground
# and leave it there
```
```Dockerfile
fg %1
FROM ubuntu:latest
COPY my_main_process my_main_process
COPY my_helper_process my_helper_process
COPY my_wrapper_script.sh my_wrapper_script.sh
CMD ./my_wrapper_script.sh
```

- 使用类似的流程管理器`supervisord`。这是一种中等重量的方法，要求您将`supervisord`镜像及其配置打包（或在包含的镜像基础上`supervisord`）及其管理的不同应用程序一起打包。然后启动`supervisord`，它将为您管理流程。下面是使用这种方法，它假定预先编写的一个例子`Dockerfile supervisord.conf`，`my_first_process`和`my_second_process`文件都存在于相同的目录中`Dockerfile`。
```Dockerfile
FROM ubuntu:latest
RUN apt-get update && apt-get install -y supervisor
RUN mkdir -p /var/log/supervisor
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY my_first_process my_first_process
COPY my_second_process my_second_process
CMD ["/usr/bin/supervisord"]
```