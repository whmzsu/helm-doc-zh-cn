# 在Helm和Tiller之间使用SSL

本文讲述了如何在Helm和Tiller之间创建强SSL/TLS连接。这里强调的是创建一个内部CA，并使用SSL的加密和身份识别功能。

> 在Helm 2.3.0中引入了对基于TLS的身份验证的支持

配置SSL是一个高级主题，并假定已了解Helm和Tiller。

## 概述

Tiller认证模型使用客户端SSL证书。Tiller自己使用证书颁发机构验证这些证书。同样，客户端还通过证书授权验证Tiller的身份。

有许多可能的设置证书和权限的配置，但我们在这里覆盖的方法适用于大多数情况。

> 从Helm 2.7.2开始，Tiller 要求客户端证书由其CA验证。在之前的版本中，Tiller使用了允许自签名证书的较弱验证策略。

在本指南中，我们将展示如何：

- 创建用于为Tiller客户端和服务端颁发证书的私有CA.
- 为Tiller创建证书
- 为Helm客户端创建一个证书
- 创建一个使用该证书的Tiller实例
- 配置Helm客户端以使用CA和客户端证书

在本指南结束时，你应该有一个正在运行的Tiller实例，它只接受来自可以通过SSL证书进行身份验证的客户端的连接。

## 生成证书颁发机构和证书

生成SSL CA的一种方法是通过`openssl`命令行工具。线上提供了许多指南和最佳实践文档。此文档着重于在少量时间内准备好配置。对于生产配置，我们建议读者阅读官方文档[the official documentation](https://www.openssl.org)并咨询其他资源。

### 生成证书授权

生成证书授权的最简单方法是运行两个命令：

```bash
$ openssl genrsa -out ./ca.key.pem 4096
$ openssl req -key ca.key.pem -new -x509 -days 7300 -sha256 -out ca.cert.pem -extensions v3_ca
Enter pass phrase for ca.key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:CO
Locality Name (eg, city) []:Boulder
Organization Name (eg, company) [Internet Widgits Pty Ltd]:tiller
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:tiller
Email Address []:tiller@example.com
```

请注意，上面输入的数据是样例数据。你应该根据自己的规格进行定制。

以上将生成一个密钥和一个CA. 请注意，这两个文件非常重要。尤其是key文件要特别注意处理。

通常，你需要生成中间签名密钥。为了简洁起见，我们将使用我们的根CA签署密钥。

### 生成证书

我们将生成两个证书，每个证书代表一种证书类型：

- 一个证书是用于Tiller的。每个tiller主机需要一个。
- 一个证书是给用户的。每个helm用户需要一个。

由于生成这些命令的命令是相同的，我们将同时创建。名字将表明他们的目标用处。

首先，Tiller 密钥：

```bash
$ openssl genrsa -out ./tiller.key.pem 4096
Generating RSA private key, 4096 bit long modulus
..........................................................................................................................................................................................................................................................................................................................++
............................................................................++
e is 65537 (0x10001)
Enter pass phrase for ./tiller.key.pem:
Verifying - Enter pass phrase for ./tiller.key.pem:
```

接下来，生成Helm客户端的密钥：

```bash
$ openssl genrsa -out ./helm.key.pem 4096
Generating RSA private key, 4096 bit long modulus
.....++
......................................................................................................................................................................................++
e is 65537 (0x10001)
Enter pass phrase for ./helm.key.pem:
Verifying - Enter pass phrase for ./helm.key.pem:
```

同样，对于生产用途，将为每个用户生成一个客户端证书。

接下来，我们需要从这些密钥创建证书。对于每个证书，这有两个步骤，创建CSR，然后创建证书。


```bash
$ openssl req -key tiller.key.pem -new -sha256 -out tiller.csr.pem
Enter pass phrase for tiller.key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:CO
Locality Name (eg, city) []:Boulder
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Tiller Server
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:tiller-server
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

我们为Helm客户端证书重复这一步骤：

```bash
$ openssl req -key helm.key.pem -new -sha256 -out helm.csr.pem
# Answer the questions with your client user's info
```

（在极少数情况下，我们必须在生成请求时添加标志`-nodes`。）

现在我们使用我们创建的CA证书对每个CSR进行签名（调整days参数以满足你的要求）：

```bash
$ openssl x509 -req -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -in tiller.csr.pem -out tiller.cert.pem -days 365
Signature ok
subject=/C=US/ST=CO/L=Boulder/O=Tiller Server/CN=tiller-server
Getting CA Private Key
Enter pass phrase for ca.key.pem:
```

再次为客户证书：

```bash
$ openssl x509 -req -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -in helm.csr.pem -out helm.cert.pem  -days 365
```

到此，对我们来说重要的文件是这些：

```
# The CA. Make sure the key is kept secret.
ca.cert.pem
ca.key.pem
# The Helm client files
helm.cert.pem
helm.key.pem
# The Tiller server files.
tiller.cert.pem
tiller.key.pem
```

现在我们准备好继续下一步。

## 创建自定义Tiller安装

Helm全面支持创建SSL配置的部署。通过指定几个标志，`helm init`命令可以创建一个新的Tiller安装，并完成所有SSL配置。

要看看这将产生什么，运行这个命令：

```bash
$ helm init --dry-run --debug --tiller-tls --tiller-tls-cert ./tiller.cert.pem --tiller-tls-key ./tiller.key.pem --tiller-tls-verify --tls-ca-cert ca.cert.pem
```

输出将显示一个Deployment，一个Secret和一个Service。SSL信息将预先加载到Secret中，Deployment将在启动时挂载到pod。

如果要定制manifest，可以将该输出保存到文件中，然后用kubectl create它将其加载到群集中。

> 我们强烈建议在集群上启用RBAC并 使用RBAC添加服务帐户[service accounts](rbac.md)。

另外，可以删除`--dry-run`和`--debug`标志。我们还建议将Tiller放入非系统namespace（`--tiller-namespace=something`）并启用服务帐户（`--service-account=somename`）。但是对于这个例子，我们将继续使用基础配置：

```bash
$ helm init --tiller-tls --tiller-tls-cert ./tiller.cert.pem --tiller-tls-key ./tiller.key.pem --tiller-tls-verify --tls-ca-cert ca.cert.pem
```

在一两分钟内它就应该准备好了。我们可以像这样检查Tiller：

```bash
$ kubectl -n kube-system get deployment
NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
... other stuff
tiller-deploy   1         1         1            1           2m
```

如果出现问题，可能需要使用`kubectl get pods -n kube-system`以找出问题所在。通过SSL/TLS支持，最常见的问题都与不正确生成的TLS证书有关，或意外更换证书和密钥。

此时，运行基本的Helm命令时应该会报错：


```bash
$ helm ls
Error: transport is closing
```

这是因为您的Helm客户端没有正确的证书来向Tiller进行身份验证。

## 配置Helm客户端

Tiller服务现在运行通过TLS保护。现在需要配置Helm客户端来执行TLS操作。

对于快速测试，我们可以手动指定我们的配置。我们将运行一个普通的Helm命令（`helm ls`），但启用SSL/TLS。

```bash
helm ls --tls --tls-ca-cert ca.cert.pem --tls-cert helm.cert.pem --tls-key helm.key.pem
```

此配置将发送我们的客户端证书以确认身份，使用客户端密钥进行加密，并使用CA证书验证远程Tiller的身份。

尽管如此，键入长命令很麻烦。快捷方法是将密钥，证书和CA移入`$HELM_HOME`：

```bash
$ cp ca.cert.pem $(helm home)/ca.pem
$ cp helm.cert.pem $(helm home)/cert.pem
$ cp helm.key.pem $(helm home)/key.pem
```

有了这个，你可以简单地运行helm ls --tls以启用TLS。

### 故障排除

*运行命令，报错 `Error: transport is closing`*

这几乎总是由于配置错误导致客户端缺少证书（`--tls-cert`）或证书不正确。

*我使用证书，但得到 `Error: remote error: tls: bad certificate`*

这意味着Tiller的CA无法验证你的证书。在上面的例子中，我们使用一个CA来生成客户端和服务端证书。在这些例子中，CA已经签署了客户的证书。然后，我们将该CA加载到Tiller。因此，当客户端证书发送到服务器时，Tiller会根据CA检查客户端证书。

*如果我使用`--tls-verify`客户端，报错`Error: x509: certificate is valid for tiller-server, not localhost`*

如果打算--tls-verify在客户端上使用，则需要确保Helm连接的主机名与证书上的主机名匹配。在某些情况下，这很尴尬，因为Helm将通过本地主机连接，或者FQDN不可用于公共解析。

*如果我在客户端使用`--tls-verify`，报错`Error: x509: certificate has expired or is not yet valid`*

你的Helm证书已过期，需要使用你的私钥和CA签署新证书（并考虑增加天数）

如果你的Tiller证书已经过期，你需要签署一个新的证书，使用base64对它进行编码并更新Tiller Secret： `kubectl edit secret tiller-secret`

## 参考
https://github.com/denji/golang-tls
https://www.openssl.org/docs/ https://jamielinux.com/docs/openssl-certificate-authority/sign-server-and-client-certificates.html
