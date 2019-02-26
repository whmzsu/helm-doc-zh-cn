# Helm 的出处与完整性验证

Helm 拥有可帮助 chart 用户验证 chart 包完整性和出处的工具。使用基于 PKI，GnuPG 和备受尊敬的软件包管理器的行业标准工具，Helm 可以生成并验证签名文件。

## 概述

完整性通过将 chart 与其出处记录进行比较来确定。出处记录存储在出处文件 provenance 中，并与打包的 chart 一起存储。例如，如果 chart 包被命名 `myapp-1.2.3.tgz`，其出处文件将是 `myapp-1.2.3.tgz.prov`。

出处文件在打包时生成（`helm package --sign ...`），并且可以通过多个命令检查，比如 `helm install --verify`。

## 工作流程

本节描述了有效使用出处数据的可用的工作流程。

前提条件：

- 二进制（非 ASCII）格式的有效 PGP 密钥对
- helm 命令行工具
- GnuPG >=2.1 命令行工具（可选）
- Keybase 命令行工具（可选）
-
** 注意：** 如果 PGP 私钥有密码，支持 `--sign` 选项的任何命令系统将提示输入密码。可以设置 HELM_KEY_PASSPHRASE 环境变量避免每次输入.

** 注意：** GnuPG 的密钥文件格式在 2.1 版中已更改。在该版本之前，没有必要从 GnuPG 中导出密钥，可以将 Helm 指向你的 `* .gpg` 文件。使用 2.1 时，引入了新的 `.kbx` 格式，Helm 不支持这种格式。

创建 chart：

```
$ helm create mychart
Creating mychart
```

准备打包后，将 `--sign` 参数加到 `helm package`。另外，指定已知签名密钥和包含相应私钥的密钥环 keyring：

```
$ helm package --sign --key 'helm signing key' --keyring path/to/keyring.secret mychart
```

** 提示：** 对于 GnuPG 用户，密钥环已存在 `~/.gnupg/secring.kbx`。可以使用 gpg --list-secret-keys 列出拥有的密钥。

** 警告：** GnuPG v2.1 在默认位置在 `〜/ .gnupg / pubring.kbx`，使用新格式'kbx'存储密钥 keyring。请使用以下命令将钥匙 keyring 转换为传统的 gpg 格式：

```
$ gpg --export-secret-keys >~/.gnupg/secring.gpg
```

到这里，应该可用看到 mychart-0.1.0.tgz 和 mychart-0.1.0.tgz.prov。这两个文件最终应该上传到想要的 chart 库。

可以使用 `helm verify` 以下方式验证 chart：

```
$ helm verify mychart-0.1.0.tgz
```

验证失败如下所示样例：

```
$ helm verify topchart-0.1.0.tgz
Error: sha256 sum does not match for topchart-0.1.0.tgz: "sha256:1939fbf7c1023d2f6b865d137bbb600e0c42061c3235528b1e8c82f4450c12a7" != "sha256:5a391a90de56778dd3274e47d789a2c84e0e106e1a37ef8cfa51fd60ac9e623a"
```

要在安装过程中进行验证，请使用该 `--verify` 标志。

```
$ helm install --verify mychart-0.1.0.tgz
```
如果密钥 keyring（包含与签名 chart 关联的公钥）不在默认位置，则可能需要 `--keyring PATH` 像 `helm package` 示例中那样指向密钥 keyring。

如果验证失败，在 chart 被推到 Tiller 之前中止安装终止。

### 使用 Keybase.io 凭据
该 Keybase.io 服务可以很容易建立信任链的密码身份。密钥库凭证可用于对 chart 进行签名。

前提条件：

- 已配置的 Keybase.io 帐户
- GnuPG 在本地安装
- keybaseCLI 本地安装

#### 签署软件包

第一步是将 keybase 密钥导入到本地 GnuPG 密钥 keyring 中：

```
$ keybase pgp export -s > secring.gpg
```

这会将 Keybase 密钥转换为 OpenPGP 格式，然后将其本地导入到 secring.gpg 文件中。

可以通过运行 `gpg --list-secret-keys` 进行仔细检查。

```
$ gpg --list-secret-keys  /Users/mattbutcher/.gnupg/secring.gpg
-------------------------------------
sec   2048R/1FC18762 2016-07-25
uid                  technosophos (keybase.io/technosophos) <technosophos@keybase.io>
ssb   2048R/D125E546 2016-07-25
```

提示: 如果你想添加一个 Keybase key 到已存在的 keyring, 你需要执行 `keybase pgp export -s | gpg --import && gpg --export-secret-keys --outfile secring.gpg`

你的密钥有一个标识符字符串：

```
technosophos (keybase.io/technosophos) <technosophos@keybase.io>
```

这是钥匙的全名。

接下来，可以使用 `helm package` 打包和签名 chart。`--key` 确保至少使用该名称字符串的一部分。


```
$ helm package --sign --key technosophos --keyring ~/.gnupg/secring.gpg mychart
```

结果，package 命令应该生成一个. tgz 文件和一个. tgz.prov 文件。

#### 验证软件包

还可以使用类似的技术来验证由其他人的 Keybase 密钥签名的 chart。假设想验证签名的软件包 keybase.io/technosophos。请使用该 keybase 工具：

```
$ keybase follow technosophos
$ keybase pgp pull
```

上面的第一条命令跟踪用户 `technosophos`。接下来 `keybase pgp pull `，将关注的所有帐户的 OpenPGP 密钥下载到 GnuPG 密钥环（`~/.gnupg/pubring.gpg1`）中

到此，可以使用 `helm verify` 或带有 `--verify` 参数的任何命令：

```
$ helm verify somechart-1.2.3.tgz
```

### 可能无法验证的原因

下面是失败的常见原因。

- prov 文件丢失或损坏。这表明某些内容配置错误或原始维护人员未创建出处文件。
- 用于签署文件的密钥不在钥匙 keyring 中。这表明签名 chart 的组织不是已经信任的人员。
- prov 文件的验证失败。这表明 chart 或出处数据有问题。
- prov 文件中的文件哈希与压缩包文件的哈希不匹配。这表明压缩包已被篡改。
- 如果验证失败，则有理由怀疑该软件包有问题。

## Provenance 文件
Provenance 文件包含 chart 的 YAML 文件以及几条验证信息。Provenance 文件被设计自动生成。

添加了以下几个出处数据：

- 包含 chart 文件（Chart.yaml）可以让人员和工具轻松查看 chart 内容。
- 包括 chart 包（.tgz 文件）的签名（SHA256，就像 Docker）一样，可用于验证 chart 包的完整性。
- 整个文件使用 PGP 使用的算法进行签名（参见 [https://keybase.io]，这是一种使加密签名和验证变得容易的新方法）。

这样的组合给了用户以下保证：

- 包本身没有被篡改（校验和包 tgz）。
- 已知发布此包的组织（通过 GnuPG / PGP 签名）。

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
home: https://nginx.com

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

注意，YAML 部分包含两个文档（由分隔 `...\n`）。首先是 Chart.yaml。第二个是校验和，一个文件名到 SHA-256 摘要的映射（上显示的值是假的 / 被截断的）

签名块是一个标准的 PGP 签名，它提供了 [防篡改](https://www.rossde.com/PGP/pgp_signatures.html) 功能。

## Chart 库

Chart 库用作 Helm chart 的集中场所。

Chart 存储库必须能够通过特定的请求通过 HTTP 为 provenance 文件提供服务，并且必须使它们在与 chart 相同的 URI 路径下可用。

例如，如果软件包的基本 URL 是 `https://example.com/charts/mychart-1.2.3.tgz`, provenance 文件（如果存在）必须可以通过 `https://example.com/charts/mychart-1.2.3.tgz.prov` 访问。

从最终用户的角度来看，`helm install --verify myrepo/mychart-1.2.3 ` 应该无需额外的配置或操作即可下载 chart 和 provenance 文件。

## 确认认证和鉴权

在处理信任链系统时，能够确认签名者的认证很重要。或者，简单地说，上述系统取决于相信签名人员的事实。这反过来又意味着你需要信任签名者的公钥。

Kubernetes Helm 的设计决策之一是 Helm 项目不会将自己插入信任链中作为必要的组分。我们不希望成为所有 chart 签名者的 “证书颁发机构”。相反，我们强烈支持分散模式，这是我们选择 OpenPGP 作为基础技术的原因之一。所以说到确认认证时，我们在 Helm 2.0.0 中对这个步骤或多或少未明确定义。

但是，对于那些有兴趣使用 povenance 系统的人，我们有一些建议：

- [Keybase](https://keybase.io) 平台提供了可靠信息的公开集中存放。
- 可以使用 Keybase 存储密钥或获取其他公钥。
- Keybase 也有很多可用的文档
- 虽然我们还没有对它进行测试，但 Keybase 的 “安全网站” 功能可用于服务 Helm chart。
- Kubernetes chart 项目 正在设法解决这个管方 chart 库 [official Kubernetes Charts project](https://github.com/kubernetes/charts) 问题。
- 这里有一个 [很长的 issue](https://github.com/kubernetes/charts/issues/23)，详细介绍了当前的想法。
- 基本思想原则是官方的 “chart reviewer” 用她或他的钥匙签名 chart，然后将得到的 Provence 文件上传到 chart 存储库。
- 关于 ` 有效签名密钥列表可包含在 index.yaml 存储库文件中 ` 的想法已经有了一些工作进展。

最后，信任链是Helm的一个发展特征，一些社区成员已经提出了将OSI模型的一部分用于签名。这是Helm团队的一个开放性调查。如果你有兴趣，请参与其中。
