# 在 Helm 和 Tiller 之间使用 SSL

本文讲述了如何在 Helm 和 Tiller 之间创建强 SSL/TLS 连接。这里强调的是创建一个内部 CA，并使用 SSL 的加密和身份识别功能。

> 在 Helm 2.3.0 中引入了对基于 TLS 的身份验证的支持

配置 SSL 是一个高级主题，需要你已了解 Helm 和 Tiller。

## 概述

Tiller 认证模型使用客户端 SSL 证书。Tiller 自己使用证书认证授权验证这些证书。同样，客户端还通过证书授权验证 Tiller 的身份。

有许多可能的设置证书和权限的配置，但我们在这里覆盖的方法适用于大多数情况。

> 从 Helm 2.7.2 开始，Tiller 要求客户端证书由其 CA 验证。在之前的版本中，Tiller 使用了允许自签名证书的较弱验证策略。

在本指南中，我们将展示如何：

- 创建用于为 Tiller 客户端和服务端颁发证书的私有 CA.
- 为 Tiller 创建证书
- 为 Helm 客户端创建一个证书
- 创建一个使用该证书的 Tiller 实例
- 配置 Helm 客户端以使用 CA 和客户端证书

在本指南结束时，你应该有一个正在运行的 Tiller 实例，它只接受来自可以通过 SSL 证书进行身份验证的客户端的连接。

## 生成证书认证授权和证书

生成 SSL CA 的一种方法是通过 `openssl` 命令行工具。线上提供了许多指南和最佳实践文档。此文档着重于在少量时间内准备好配置。对于生产配置，我们建议读者阅读官方文档 [the official documentation](https://www.openssl.org) 并咨询其他资源。

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

以上将生成一个密钥和一个 CA. 请注意，这两个文件非常重要。尤其是 key 文件要特别注意处理。

通常，你需要生成中间签名密钥。为了简洁起见，我们将使用我们的根 CA 签署密钥。

### 生成证书

我们将生成两个证书，每个证书代表一种证书类型：

- 一个证书是用于 Tiller 的。每个 tiller 主机需要一个。
- 一个证书是给用户的。每个 helm 用户需要一个。

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

接下来，生成 Helm 客户端的密钥：

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

接下来，我们需要从这些密钥创建证书。对于每个证书，这有两个步骤，创建 CSR，然后创建证书。


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

我们为 Helm 客户端证书重复这一步骤：

```bash
$ openssl req -key helm.key.pem -new -sha256 -out helm.csr.pem
# Answer the questions with your client user's info
```

（在极少数情况下，我们必须在生成请求时添加标志 `-nodes`。）

现在我们使用我们创建的 CA 证书对每个 CSR 进行签名（调整 days 参数以满足你的要求）：

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

## 创建自定义 Tiller 安装

Helm 全面支持创建 SSL 配置的部署。通过指定几个标志，`helm init` 命令可以创建一个新的 Tiller 安装，并完成所有 SSL 配置。

要看看这将产生什么，运行这个命令：

```bash
$ helm init --dry-run --debug --tiller-tls --tiller-tls-cert ./tiller.cert.pem --tiller-tls-key ./tiller.key.pem --tiller-tls-verify --tls-ca-cert ca.cert.pem
```

输出将显示一个 Deployment，一个 Secret 和一个 Service。SSL 信息将预先加载到 Secret 中，Deployment 将在启动时挂载到 pod。

如果要定制 manifest，可以将该输出保存到文件中，然后用 kubectl create 它将其加载到群集中。

> 我们强烈建议在集群上启用 RBAC 并 使用 RBAC 添加服务帐户 [service accounts](rbac.md)。

另外，可以删除 `--dry-run` 和 `--debug` 标志。我们还建议将 Tiller 放入非系统 namespace（`--tiller-namespace=something`）并启用服务帐户（`--service-account=somename`）。但是对于这个例子，我们将继续使用基础配置：

```bash
$ helm init --tiller-tls --tiller-tls-cert ./tiller.cert.pem --tiller-tls-key ./tiller.key.pem --tiller-tls-verify --tls-ca-cert ca.cert.pem
```

在一两分钟内它就应该准备好了。我们可以像这样检查 Tiller：

```bash
$ kubectl -n kube-system get deployment
NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
... other stuff
tiller-deploy   1         1         1            1           2m
```

如果出现问题，可能需要使用 `kubectl get pods -n kube-system` 以找出问题所在。通过 SSL/TLS 支持，最常见的问题都与不正确生成的 TLS 证书有关，或意外更换证书和密钥。

此时，运行基本的 Helm 命令时应该会报错：


```bash
$ helm ls
Error: transport is closing
```

这是因为您的 Helm 客户端没有正确的证书来向 Tiller 进行身份验证。

## 配置 Helm 客户端

Tiller 服务现在运行通过 TLS 保护。现在需要配置 Helm 客户端来执行 TLS 操作。

对于快速测试，我们可以手动指定我们的配置。我们将运行一个普通的 Helm 命令（`helm ls`），但启用 SSL/TLS。

```bash
helm ls --tls --tls-ca-cert ca.cert.pem --tls-cert helm.cert.pem --tls-key helm.key.pem
```

此配置将发送我们的客户端证书以确认身份，使用客户端密钥进行加密，并使用 CA 证书验证远程 Tiller 的身份。

尽管如此，键入长命令很麻烦。快捷方法是将密钥，证书和 CA 移入 `$HELM_HOME`：

```bash
$ cp ca.cert.pem $(helm home)/ca.pem
$ cp helm.cert.pem $(helm home)/cert.pem
$ cp helm.key.pem $(helm home)/key.pem
```

有了这个，你可以简单地运行 helm ls --tls 以启用 TLS。

### 故障排除

* 运行命令，报错 `Error: transport is closing`*

这几乎总是由于配置错误导致客户端缺少证书（`--tls-cert`）或证书不正确。

* 我使用证书，但得到 `Error: remote error: tls: bad certificate`*

这意味着 Tiller 的 CA 无法验证你的证书。在上面的例子中，我们使用一个 CA 来生成客户端和服务端证书。在这些例子中，CA 已经签署了客户的证书。然后，我们将该 CA 加载到 Tiller。因此，当客户端证书发送到服务器时，Tiller 会根据 CA 检查客户端证书。

* 如果我使用 `--tls-verify` 客户端，报错 `Error: x509: certificate is valid for tiller-server, not localhost`*

如果打算 --tls-verify 在客户端上使用，则需要确保 Helm 连接的主机名与证书上的主机名匹配。在某些情况下，这很尴尬，因为 Helm 将通过本地主机 localhost 连接，或者 FQDN 不可用于公共解析。

* 如果我在客户端使用 `--tls-verify` , 返回报错信息 `Error: x509: cannot validate certificate for 127.0.0.1 because it doesn't contain any IP SANs`*


默认情况下，Helm 客户端通过隧道（即 kube 代理）127.0.0.1 连接到 Tiller。 在 TLS 握手期间，通常提供主机名（例如 example.com），对证书进行检查，包括附带的信息。 但是，由于通过隧道，目标是 IP 地址。因此，要验证证书，必须在 Tiller 证书中将 IP 地址 127.0.0.1 列为 IP 附带备用名称（IP SAN： IP subject alternative name）。

例如，要在生成 Tiller 证书时将 127.0.0.1 列为 IP SAN：

```bash
$ echo subjectAltName=IP:127.0.0.1 > extfile.cnf
$ openssl x509 -req -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -in tiller.csr.pem -out tiller.cert.pem -days 365 -extfile extfile.cnf
```


* 如果我在客户端使用 `--tls-verify`，报错 `Error: x509: certificate has expired or is not yet valid`*

你的 Helm 证书已过期，需要使用你的私钥和 CA 签署新证书（并考虑增加天数）

如果你的 Tiller 证书已经过期，你需要签署一个新的证书，使用 base64 对它进行编码并更新 Tiller Secret： `kubectl edit secret tiller-secret`

## 参考
https://github.com/denji/golang-tls

https://www.openssl.org/docs/

https://jamielinux.com/docs/openssl-certificate-authority/sign-server-and-client-certificates.html
