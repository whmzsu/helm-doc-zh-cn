# Helm 的出处与完整性验证

Helm拥有可帮助chart用户验证chart包完整性和出处的工具。使用基于PKI，GnuPG和备受尊敬的软件包管理器的行业标准工具，Helm可以生成并验证签名文件。

## 概述

完整性通过将chart与其出处记录进行比较来确定。出处记录存储在出处文件provenance中，并与打包的chart一起存储。例如，如果chart包被命名`myapp-1.2.3.tgz`，其出处文件将是`myapp-1.2.3.tgz.prov`。

出处文件在打包时生成（`helm package --sign ...`），并且可以通过多个命令检查，比如`helm install --verify`。

## 工作流程

本节描述了有效使用出处数据的可用的工作流程。

前提条件：

- 二进制（非ASCII）格式的有效PGP密钥对
- helm命令行工具
- GnuPG命令行工具（可选）
- Keybase命令行工具（可选）
-
**注意：**如果PGP私钥有密码，支持`--sign`选项的任何命令系统将提示输入密码以查看。

创建chart：

```
$ helm create mychart
Creating mychart
```

准备打包后，将`--sign`参数加到`helm package`。另外，指定已知签名密钥和包含相应私钥的密钥环keyring：

```
$ helm package --sign --key 'helm signing key' --keyring path/to/keyring.secret mychart
```

**提示：**对于GnuPG用户，密钥环已存在`~/.gnupg/secring.gpg`。可以使用gpg --list-secret-keys列出拥有的密钥。

**警告：** GnuPG v2在默认位置在`〜/ .gnupg / pubring.kbx`，使用新格式'kbx'存储密钥keyring。请使用以下命令将钥匙keyring转换为传统的gpg格式：

```
$ gpg --export-secret-keys >~/.gnupg/secring.gpg
```

到这里，应该可用看到mychart-0.1.0.tgz和mychart-0.1.0.tgz.prov。这两个文件最终应该上传到想要的chart库。

可以使用`helm verify`以下方式验证chart：

```
$ helm verify mychart-0.1.0.tgz
```

验证失败如下所示样例：

```
$ helm verify topchart-0.1.0.tgz
Error: sha256 sum does not match for topchart-0.1.0.tgz: "sha256:1939fbf7c1023d2f6b865d137bbb600e0c42061c3235528b1e8c82f4450c12a7" != "sha256:5a391a90de56778dd3274e47d789a2c84e0e106e1a37ef8cfa51fd60ac9e623a"
```

要在安装过程中进行验证，请使用该`--verify`标志。

```
$ helm install --verify mychart-0.1.0.tgz
```
如果密钥keyring（包含与签名chart关联的公钥）不在默认位置，则可能需要`--keyring PATH`像`helm package`示例中那样指向密钥keyring。

如果验证失败，在chart被推到Tiller之前中止安装终止。

### 使用Keybase.io凭据
该Keybase.io服务可以很容易建立信任链的密码身份。密钥库凭证可用于对chart进行签名。

前提条件：

- 已配置的Keybase.io帐户
- GnuPG在本地安装
- keybaseCLI本地安装

#### 签署软件包

第一步是将keybase密钥导入到本地GnuPG密钥keyring中：

```
$ keybase pgp export -s | gpg --import
```

这会将Keybase密钥转换为OpenPGP格式，然后将其本地导入到`~/.gnupg/secring.gpg`文件中。

可以通过运行`gpg --list-secret-keys`进行仔细检查。

```
$ gpg --list-secret-keys  /Users/mattbutcher/.gnupg/secring.gpg
-------------------------------------
sec   2048R/1FC18762 2016-07-25
uid                  technosophos (keybase.io/technosophos) <technosophos@keybase.io>
ssb   2048R/D125E546 2016-07-25
```

注意，密钥有一个标识符字符串：

```
technosophos (keybase.io/technosophos) <technosophos@keybase.io>
```

这是钥匙的全名。

接下来，可以使用`helm package`打包和签名chart。`--key`确保至少使用该名称字符串的一部分。


```
$ helm package --sign --key technosophos --keyring ~/.gnupg/secring.gpg mychart
```

结果，package命令应该生成一个.tgz文件和一个.tgz.prov 文件。

#### 验证软件包

还可以使用类似的技术来验证由其他人的Keybase密钥签名的chart。假设想验证签名的软件包keybase.io/technosophos。请使用该keybase工具：

```
$ keybase follow technosophos
$ keybase pgp pull
```

上面的第一条命令跟踪用户`technosophos`。接下来`keybase pgp pull `，将关注的所有帐户的OpenPGP密钥下载到GnuPG密钥环（`~/.gnupg/pubring.gpg1`）中

到此，可以使用`helm verify`或带有`--verify`参数的任何命令：

```
$ helm verify somechart-1.2.3.tgz
```

### 可能无法验证的原因

下面是失败的常见原因。

- prov文件丢失或损坏。这表明某些内容配置错误或原始维护人员未创建出处文件。
- 用于签署文件的密钥不在钥匙keyring中。这表明签名chart的组织不是已经信任的人员。
- prov文件的验证失败。这表明chart或出处数据有问题。
- prov文件中的文件哈希与压缩包文件的哈希不匹配。这表明压缩包已被篡改。
- 如果验证失败，则有理由怀疑该软件包有问题。

## Provenance文件
Provenance文件包含chart的YAML文件以及几条验证信息。Provenance文件被设计自动生成。

添加了以下几个出处数据：

- 包含chart文件（Chart.yaml）可以让人员和工具轻松查看chart内容。
- 包括chart包（.tgz文件）的签名（SHA256，就像Docker）一样，可用于验证chart包的完整性。
- 整个文件使用PGP使用的算法进行签名（参见[ http://keybase.io ]，这是一种使加密签名和验证变得容易的新方法）。

这样的组合给了用户以下保证：

- 包本身没有被篡改（校验和包tgz）。
- 已知发布此包的组织（通过GnuPG / PGP签名）。

该文件的格式如下所示：

```
-----BEGIN PGP SIGNED MESSAGE-----
name: nginx
description: The nginx web server as a replication controller and service pair.
version: 0.5.1
keywords:
  - https
  - http
  - web server
  - proxy
source:
- https://github.com/foo/bar
home: http://nginx.com

...
files:
        nginx-0.5.1.tgz: “sha256:9f5270f50fc842cfcb717f817e95178f”
-----BEGIN PGP SIGNATURE-----
Version: GnuPG v1.4.9 (GNU/Linux)

iEYEARECAAYFAkjilUEACgQkB01zfu119ZnHuQCdGCcg2YxF3XFscJLS4lzHlvte
WkQAmQGHuuoLEJuKhRNo+Wy7mhE7u1YG
=eifq
-----END PGP SIGNATURE-----
```

注意，YAML部分包含两个文档（由分隔`...\n`）。首先是Chart.yaml。第二个是校验和，一个文件名到SHA-256摘要的映射（上显示的值是假的/被截断的）

签名块是一个标准的PGP签名，它提供了[防篡改](http://www.rossde.com/PGP/pgp_signatures.html)功能。

## Chart库

Chart库用作Helm chart的集中场所。

Chart存储库必须能够通过特定的请求通过HTTP为provenance文件提供服务，并且必须使它们在与chart相同的URI路径下可用。

例如，如果软件包的基本URL是`https://example.com/charts/mychart-1.2.3.tgz`, provenance文件（如果存在）必须可以通过`https://example.com/charts/mychart-1.2.3.tgz.prov`访问。

从最终用户的角度来看，`helm install --verify myrepo/mychart-1.2.3 ``应该无需额外的配置或操作即可下载chart和provenance文件。

## 确认权力和鉴权

在处理信任链系统时，能够确认签名者的权力很重要。或者，简单地说，上述系统取决于相信签名人员的事实。这反过来又意味着你需要信任签名者的公钥。

Kubernetes Helm的设计决策之一是Helm项目不会将自己插入信任链中作为必要的组分。我们不希望成为所有chart签名者的“证书颁发机构”。相反，我们强烈支持分散模式，这是我们选择OpenPGP作为基础技术的原因之一。所以说到确认权威时，我们在Helm 2.0.0中对这个步骤或多或少未明确定义。

但是，对于那些有兴趣使用rpovenance系统的人，我们有一些建议：

- [Keybase](https://keybase.io) 平台提供了可靠信息的公开集中存放。
- 可以使用Keybase存储密钥或获取其他公钥。
- Keybase也有很多可用的文档
- 虽然我们还没有对它进行测试，但Keybase的“安全网站”功能可用于服务Helm chart。
- Kubernetes chart项目 正在设法解决这个管方chart库[official Kubernetes Charts project](https://github.com/kubernetes/charts)问题。
- 这里有一个[很长的issue](https://github.com/kubernetes/charts/issues/23)，详细介绍了当前的想法。
- 基本思想原则是官方的“chart reviewer”用她或他的钥匙签名chart，然后将得到的Provence文件上传到chart存储库。
- 关于`有效签名密钥列表可包含在index.yaml存储库文件中`的想法已经有了一些工作进展。

最后，信任链是Helm的一个发展特征，一些社区成员已经提出了将OSI模型的一部分用于签名。这是Helm团队的一个开放性调查。如果你有兴趣，请参与其中。
