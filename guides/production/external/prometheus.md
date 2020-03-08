# 使用`Prometheus`收集`Docker`指标

`Prometheus`是一个开源系统监视和警报工具包。您可以将`Docker`配置为`Prometheus`目标。本主题向您展示如何配置`Docker`，设置`Prometheus`以作为`Docker`容器运行以及如何使用`Prometheus`监视`Docker`实例。

> 警告：可用的度量标准和这些度量标准的名称正在开发中，并可能随时更改。

当前，您只能监视`Docker`本身。您目前无法使用`Docker`目标监视您的应用程序。

## 配置
要将`Docker`守护程序配置为`Prometheus`目标，您需要指定 `metrics-address`。最好的方法是通过`daemon.json`，默认情况下，它位于以下位置之一。如果文件不存在，请创建它。

- `Linux`：`/etc/docker/daemon.json`
- `Windows Server`：`C:\ProgramData\docker\config\daemon.json`
- 适用于`Mac`的`Docker`桌面/适用于`Windows`的`Docker`桌面：单击工具栏中的`Docker`图标，选择首选项，然后选择守护程序。单击高级。

如果文件当前为空，请粘贴以下内容：
```json
{
  "metrics-addr" : "127.0.0.1:9323",
  "experimental" : true
}
```

如果文件不为空，请添加这两个键，确保结果文件为有效`JSON`。请注意，,除最后一行外，每行均以(`,`)结尾。

保存文件，或者对于`Mac`的`Docker Desktop`或`Windows`的`Docker Desktop`，保存配置。重新启动`Docker`。

`Docker`现在在端口`9323`上公开了与`Prometheus`兼容的指标。

## 配置和运行`Prometheus`
`Prometheus`作为`Docker`服务在`Docker``Swarm`上运行。

> 先决条件

> 1. 使用`docker swarm init` 一个管理器以及`docker swarm join`其他管理器和工作器节点，将一个或多个`Docker`引擎加入到`Docker``Swarm`中。

> 2. 您需要互联网连接才能提取`Prometheus`图像。

复制以下配置文件之一，并将其​​保存到`/tmp/prometheus.yml`（`Linux`或`Mac`）或`C:\tmp\prometheus.yml`（`Windows`）。这是一个常规的`Prometheus`配置文件，除了在文件底部添加了`Docker`作业定义外。`Mac`版`Docker Desktop`和`Windows`版`Docker Desktop`需要稍有不同的配置。

适用于`Linux`的`Docker`
```shell
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'docker'
         # metrics_path defaults to '/metrics'
         # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9323']
```
接下来，使用此配置启动单副本`Prometheus`服务。

适用于`Linux`的`Docker`
```shell
$ docker service create --replicas 1 --name my-prometheus \
    --mount type=bind,source=/tmp/prometheus.yml,destination=/etc/prometheus/prometheus.yml \
    --publish published=9090,target=9090,protocol=tcp \
    prom/prometheus
```
验证`Docker`目标是否在`http://localhost:9090/targets/`中列出。

![](images/prometheus-graph_idle.png)

如果使用`Mac`的`Docker`桌面或`Windows`的`Docker`桌面，则无法直接访问端点`URL`。

## 使用`Prometheus` 
创建一个图形。单击`Prometheus UI`中的“ 图形”链接。从执行按钮右侧的组合框中选择一个指标，然后点击 执行。以下屏幕截图显示了的图形 `engine_daemon_network_actions_seconds_count`。

![](images/prometheus-targets.png)

上图显示了一个非常空闲的`Docker`实例。如果您正在运行活动的工作负载，则图表的外观可能会有所不同。

为了使图更有趣，可通过启动一项服务来创建一些网络操作，这些服务仅需不间断地`ping Docker`的10个任务（您可以将`ping`目标更改为您喜欢的任何对象）：
```shell
$ docker service create \
  --replicas 10 \
  --name ping_service \
  alpine ping docker.com
```
等待几分钟（默认的抓取间隔为15秒），然后重新加载图形。

![](images/prometheus-graph_load.png)

准备就绪后，请停止并删除该`ping_service`服务，以使您不会无缘无故地用`ping`泛洪主机。
```shell
$ docker service remove ping_service
```
等待几分钟，您应该会看到图表回落到空闲水平。
