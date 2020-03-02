# 配置Docker以使用代理服务器

如果您的容器需要使用`HTTP`，`HTTPS`或`FTP`代理服务器，则可以通过不同的方式对其进行配置：

在`Docker 17.07`及更高版本中，您可以配置`Docker`客户端以将代理信息自动传递到容器。

在`Docker 17.06`及更低版本中，您必须在容器内设置适当的环境变量。您可以在生成映像时执行此操作（这会使映像的便携性降低），或者在创建或运行容器时执行此操作。

## 配置Docker客户端
1. 在`Docker`客户端上，在启动容器的用户的主目录中创建或编辑`〜/.docker/config.json`文件。添加如下所示的`JSON`，如有必要，用`httpsProxy`或`ftpProxy`替换代理的类型，并替换代理服务器的地址和端口。您可以同时配置多个代理服务器。

通过将`noProxy`密钥设置为一个或多个逗号分隔的`IP`地址或主机，您可以有选择地排除主机或范围通过代理服务器。如示例中所示，支持将`*`字符用作通配符。
```json
{
 "proxies":
 {
   "default":
   {
     "httpProxy": "http://127.0.0.1:3001",
     "httpsProxy": "http://127.0.0.1:3001",
     "noProxy": "*.test.example.com,.example2.com"
   }
 }
}
```
保存文件。

2. 创建或启动新容器时，环境变量将在容器内自动设置。

## 使用环境变量
在生成镜像时，或者在创建或运行容器时使用`--env`标志，可以将以下一个或多个变量设置为适当的值。 此方法使映像的可移植性降低，因此，如果您具有`Docker 17.07`或更高版本，则应配置`Docker`客户端。

| 变量 | Dockerfile例子 | docker run例子 |
| --- | --- | --- |
| `HTTP_PROXY`	 | `ENV HTTP_PROXY "http://127.0.0.1:3001"` |	`--env HTTP_PROXY="http://127.0.0.1:3001"` |
| `HTTPS_PROXY` |	`ENV HTTPS_PROXY "https://127.0.0.1:3001"` |	`--env HTTPS_PROXY="https://127.0.0.1:3001"` |
| `FTP_PROXY` |	`ENV FTP_PROXY "ftp://127.0.0.1:3001"` |	`--env FTP_PROXY="ftp://127.0.0.1:3001"` |
| `NO_PROXY` |	`ENV NO_PROXY "*.test.example.com,.example2.com"` |	`--env NO_PROXY="*.test.example.com,.example2.com" \` |