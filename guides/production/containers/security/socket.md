# 保护`Docker`守护进程套接字

默认情况下，`Docker`通过非网络`UNIX`套接字运行。它还可以选择使用`HTTP`套接字进行通信。

如果需要以安全的方式通过网络访问`Docker`，可以通过指定该`tlsverify`标志并将`Docke`r的`tlscacert`标志指向 受信任的`CA`证书来启用`TLS` 。

在守护程序模式下，它仅允许来自由该`CA`签名的证书验证的客户端的连接。在客户端模式下，它仅连接到具有该`CA`签名的证书的服务器。

> 进阶主题

> 使用`TLS`和管理`CA`是一个高级主题。在生产中使用`OpenSSL`，`x509`和`TLS`之前，请先熟悉一下它们。

## 使用`OpenSS`L创建一个`CA`，服务器和客户端密钥
> 注意：将`$HOST`以下示例中的所有实例替换为`Docker`守护程序主机的`DNS`名称。

首先，在`Docker`守护程序的主机上，生成`CA`私钥和公钥：
```shell
$ openssl genrsa -aes256 -out ca-key.pem 4096
Generating RSA private key, 4096 bit long modulus
............................................................................................................................................................................................++
........++
e is 65537 (0x10001)
Enter pass phrase for ca-key.pem:
Verifying - Enter pass phrase for ca-key.pem:

$ openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
Enter pass phrase for ca-key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:Queensland
Locality Name (eg, city) []:Brisbane
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Docker Inc
Organizational Unit Name (eg, section) []:Sales
Common Name (e.g. server FQDN or YOUR name) []:$HOST
Email Address []:Sven@home.org.au
```
现在您已经有了`CA`，您可以创建服务器密钥和证书签名请求（`CSR`）。确保“公用名”与您用于连接`Docker`的主机名匹配：

> 注意：将`$HOST`以下示例中的所有实例替换为`Docker`守护程序主机的`DNS`名称。
```shell
$ openssl genrsa -out server-key.pem 4096
Generating RSA private key, 4096 bit long modulus
.....................................................................++
.................................................................................................++
e is 65537 (0x10001)

$ openssl req -subj "/CN=$HOST" -sha256 -new -key server-key.pem -out server.csr
```
接下来，我们将用`CA`签署公钥：

由于可以通过`IP`地址和`DNS`名称建立`TLS`连接，因此在创建证书时需要指定`IP`地址。例如，允许使用`10.10.10.20`和`127.0.0.1`进行连接：
```shell
$ echo subjectAltName = DNS:$HOST,IP:10.10.10.20,IP:127.0.0.1 >> extfile.cnf
```
将`Docker`守护程序密钥的扩展使用属性设置为仅用于服务器身份验证：
```shell
$ echo extendedKeyUsage = serverAuth >> extfile.cnf
```
现在，生成签名证书：
```shell
$ openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -extfile extfile.cnf
Signature ok
subject=/CN=your.host.com
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```
授权插件提供了更细粒度的控制，以补充来自相互`TLS`的身份验证。除了以上文档中描述的其他信息外，在`Docker`守护程序上运行的授权插件还会接收用于连接`Docker`客户端的证书信息。

对于客户端身份验证，创建客户端密钥和证书签名请求：

> 注意：为了简化接下来的几个步骤，您也可以在`Docker`守护程序的主机上执行此步骤。
```shell
$ openssl genrsa -out key.pem 4096
Generating RSA private key, 4096 bit long modulus
.........................................................++
................++
e is 65537 (0x10001)

$ openssl req -subj '/CN=client' -new -key key.pem -out client.csr
```
为了使密钥适合客户端身份验证，请创建一个新的扩展配置文件：
```shell
$ echo extendedKeyUsage = clientAuth > extfile-client.cnf
```
现在，生成签名证书：
```shell
$ openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out cert.pem -extfile extfile-client.cnf
Signature ok
subject=/CN=client
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```
生成后`cert.pem`，`server-cert.pem`您可以安全地删除两个证书签名请求和扩展名配置文件：
```shell
$ rm -v client.csr server.csr extfile.cnf extfile-client.cnf
```
默认`umask`密钥为`022`，您的密钥对于您和您的组是全球可读的且可写的。

为了保护您的钥匙免遭意外损坏，请删除其写权限。要使它们仅供您阅读，请按以下方式更改文件模式：
```shell
$ chmod -v 0400 ca-key.pem key.pem server-key.pem
```
证书可能是世界可读的，但是您可能希望删除写访问权限以防止意外损坏：
```shell
$ chmod -v 0444 ca.pem server-cert.pem cert.pem
```
现在，您可以使`Docker`守护程序仅接受来自提供`CA`信任的证书的客户端的连接：
```shell
$ dockerd --tlsverify --tlscacert=ca.pem --tlscert=server-cert.pem --tlskey=server-key.pem \
  -H=0.0.0.0:2376
```
要连接到`Docker`并验证其证书，请提供您的客户端密钥，证书和受信任的`CA`：

在客户端计算机上运行

此步骤应在`Docker`客户端计算机上运行。这样，您需要将`CA`证书，服务器证书和客户端证书复制到该计算机上。

> 注意：将`$HOST`以下示例中的所有实例替换为`Docker`守护程序主机的`DNS`名称。
```shell
$ docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem \
  -H=$HOST:2376 version
```
> 注意：基于`TLS`的`Docker`应该在`TCP`端口`2376`上运行。

> 警告：如上例所示，使用证书身份验证时，您不需要`docker`使用`sudo`或`docker`组运行客户端。这意味着拥有密钥的任何人都可以向您的`Docker`守护程序提供任何指令，从而使他们对托管该守护程序的机器具有`root`访问权限。像使用`root`密码一样保护这些密钥！

## 默认安全
如果要默认保护`Docker`客户端连接的安全性，可以将文件移动到`.docker`主目录`--- `的目录中，并同时设置` DOCKER_HOS`T和`DOCKER_TLS_VERIFY`变量（而不是在每次调用时传递 `-H=tcp://$HOST:2376和--tlsverify`）。
```shell
$ mkdir -pv ~/.docker
$ cp -v {ca,cert,key}.pem ~/.docker

$ export DOCKER_HOST=tcp://$HOST:2376 DOCKER_TLS_VERIFY=1
```
`Docker`现在默认情况下安全连接：
```shell
$ docker ps
```
## 其他模式
如果您不想进行完整的双向身份验证，则可以通过混合标志以各种其他模式运行`Docker`。

### 守护程序模式
- `tlsverify`，`tlscacert`，`tlscert`，`tlskey`设置：验证客户端
- `tls`，`tlscert`，`tlskey`：不验证客户端

### 客户端模式
- `tls`：根据公共/默认`CA`池对服务器进行身份验证
- `tlsverify`，`tlscacert`：根据给定的`CA`验证服务器
- `tls`，`tlscert`，`tlskey`：根据给定的`CA`验证使用客户端证书，不验证服务器
- `tlsverify`，`tlscacert`，`tlscert`，`tlskey`：身份验证使用客户端证书，验证服务器基于给定`CA`

如果找到，客户端将发送其客户端证书，因此您只需将密钥放入即可`~/.docker/{ca,cert,key}.pem`。或者，如果要将密钥存储在其他位置，则可以使用环境变量指定该位置`DOCKER_CERT_PATH`。
```shell
$ export DOCKER_CERT_PATH=~/.docker/zone1/
$ docker --tlsverify ps
```

### 使用`curl`连接到安全`Docker`端口
要用于`curl`发出测试`API`请求，您需要使用三个额外的命令行标志：
```shell
$ curl https://$HOST:2376/images/json \
  --cert ~/.docker/cert.pem \
  --key ~/.docker/key.pem \
  --cacert ~/.docker/ca.pem
```