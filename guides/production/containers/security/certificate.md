# 使用证书验证存储库客户端

在使用`HTTPS`运行`Docker`中，您了解到，默认情况下，`Docker`通过非联网的`Unix`套接字运行，并且必须启用`TLS才`能使`Docker`客户端和守护程序通过`HTTPS`安全地进行通信。`TLS`确保注册表端点的真实性，并且加密往返于注册表的流量。

本文演示了如何确保`Docker`注册表服务器和`Docker`守护程序（注册表服务器的客户端）之间的流量经过加密，并使用基于证书的客户端服务器身份验证进行了正确身份验证。

我们向您展示如何为注册表安装证书颁发机构（`CA`）根证书，以及如何设置客户端`TLS`证书进行验证。

## 了解配置
通过`/etc/docker/certs.d`使用与注册表的主机名相同的名称创建目录，以 配置自定义证书` localhos`t。所有`*.crt`文件都以`CA`根目录的形式添加到此目录中。

> 注意：从`Docker 1.13`开始，在`Linux上`，任何根证书颁发机构都与系统默认值合并，包括作为主机的根`CA`设置。在早期版本的`Docker`和适用于`Windows Server`的`Docker Enterprise Edition`上，仅当未配置自定义根证书时才使用系统默认证书。

一个或多个`<filename>.key/cert`对的存在向`Docker`指示需要访问所需存储库的自定义证书。

> 注意：如果存在多个证书，则按字母顺序尝试每个证书。如果出现`4xx`级或`5xx`级身份验证错误，`Docker`将继续尝试下一个证书。

下面说明了带有自定义证书的配置：
```shell
    /etc/docker/certs.d/        <-- Certificate directory
    └── localhost:5000          <-- Hostname:port
       ├── client.cert          <-- Client certificate
       ├── client.key           <-- Client key
       └── ca.crt               <-- Certificate authority that signed
                                    the registry certificate
```
前面的示例是特定于操作系统的，仅用于说明目的。您应该查阅操作系统文档，以创建操作系统提供的捆绑式证书链。

## 创建客户端证书
使用`OpenSSL genrsa`和`req`命令首先生成`RSA`密钥，然后使用密钥创建证书。
```shell
$ openssl genrsa -out client.key 4096
$ openssl req -new -x509 -text -key client.key -out client.cert
```
> 注意：这些`TLS`命令仅在`Linux`上生成一组有效的证书。`macOS`中的`OpenSSL`版本与`Docker`要求的证书类型不兼容。

## 故障排除技巧
`Docker`守护程序将`.crt`文件解释为`CA`证书，并将`.cert`文件解释为客户端证书。如果`CA`证书被意外赋予了扩展名 `.cert`而不是正确的`.crt`扩展名，则`Docker`守护程序会记录以下错误消息：
```shell
Missing key KEY_NAME for client certificate CERT_NAME. CA certificates should use the extension .crt.
```
如果在没有端口号的情况下访问`Docker`注册表，请勿将端口添加到目录名称中。下面显示了默认端口`443上`注册表的配置，该端口可通过进行访问`docker login my-https.registry.example.com`：
```shell
    /etc/docker/certs.d/
    └── my-https.registry.example.com          <-- Hostname without port
       ├── client.cert
       ├── client.key
       └── ca.crt
```