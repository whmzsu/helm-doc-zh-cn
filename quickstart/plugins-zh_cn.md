# 插件指南

Helm 2.1.0 引入了客户端 Helm 插件_plugin_的概念。插件是一种可以通过 helm CLI 访问的工具，但它不是内置 Helm 代码库的一部分。

现有的插件可以在相关部分 [related](../related-zh_cn.md#Helm 插件) 找到或者通过搜索 [Github](https://github.com/search?q=topic%3Ahelm-plugin&type=Repositories)。

本指南介绍了如何使用和创建插件。

## 概述

Helm 插件是与 Helm 无缝集成的附加工具。它们提供了扩展 Helm 核心功能集的方法，但不需要将每个新功能都通过 Go 语言写入并添加到核心工具中。

Helm 插件具有以下功能：

- 可以在 Helm 安装中添加和删除它们，而不会影响核心 Helm 工具。
- 它们可以用任何编程语言编写。
- 他们与 Helm 集成，并出现在 helm help 和其他地方。

Helm 插件放置在 $(helm home)/plugins。

Helm 插件模型部分建模在​​Git 的插件模型上。为此，有时可能会听到 helm 称为瓷层 _porcelain_，插件是管道 _plumbing_。这是揭示 Helm 提供用户体验和顶级处理逻辑，而插件则是执行所需操作的 “细节工作” 的简略说法。

## 安装插件

使用 `$ helm plugin install <path|url>` 命令安装插件。可以将路径设置为本地文件系统上的插件或远程 VCS repo 的 URL。`helm plugin install` 命令克隆或复制该插件的路径 / URL 到给定的 `$(helm home)/plugins`

```bash
$ helm plugin install https://github.com/technosophos/helm-template
```

如果你有一个插件 tar 分发版，只需将插件解压到 $(helm home)/plugins 目录中即可。

也可以通过直接从 URL 安装 tarball 插件 `helm plugin install http://domain/path/to/plugin.tar.gz`

## 构建插件

在很多方面，插件类似于 chart。每个插件都有一个顶级目录，然后是一个 `plugin.yaml` 文件。

```
$(helm home)/plugins/
  |- keybase/
      |
      |- plugin.yaml
      |- keybase.sh

```

在上面的例子中，keybase 插件包含在名为 keybase 的目录中。它有两个文件：（`plugin.yaml` 必需）和一个可执行脚本 `keybase.sh`（可选）。

插件的核心是一个简单的 YAML 文件 `plugin.yaml`。这是一个插件的一个插件 YAML，它增加了对 Keybase 操作的支持：

```
name: "keybase"
version: "0.1.0"
usage: "Integrate Keybase.io tools with Helm"
description: |-
  This plugin provides Keybase services to Helm.
ignoreFlags: false
useTunnel: false
command: "$HELM_PLUGIN_DIR/keybase.sh"
```

`name` 是插件的名称。当 Helm 执行插件时，这是它将使用的名称（例如，`helm NAME` 将调用此插件）。

_`name` 应该匹配目录名称。_ 在我们上面的例子中，这意味着插件 `name: keybase` 应该在一个名为 keybase 的目录中。

`name` 的限制：

- `name` 不能一个现有的 helm 顶级命令重复。
- name 必须限制为 ASCII az，AZ，0-9 `_` 和 `-。

`version` 是插件的 SemVer 2 版本。 `usage` 和 `description` 都用于生成命令的帮助文本。

`ignoreFlags` 告诉 H​​elm 不会将参数传递给插件。所以，如果一个插件被 `helm myplugin --foo` 调用，并且 `ignoreFlags: true`，那么 `--foo` 将被忽略。

`useTunnel` 指示插件需要一个隧道去连接 Tiller。这在任何时候插件与 Tiller 对接都应该设置为 true 。它会使 Helm 打开一个隧道，然后 ``$TILLER_HOST` 为该隧道设置正确的本地地址。不用担心：如果 Helm 由于 Tiller 在本地运行而检测到隧道是不必啊哟的，它就不会创建隧道。

最后，也是最重要的是，`command`，是这个插件在调用时会执行的命令。在执行插件之前会插入环境变量。上面的模式说明了指出插件程序所在位置的首选方式。

有一些使用插件命令的策略：

- 如果插件包含可执行文件 `command:`，则应将可执行文件打包到插件目录中。
- `command:` 将在执行前展开任何环境变量。`$HELM_PLUGIN_DIR` 将指向插件目录。
- 该命令本身不在 shell 中执行。所以你不能在一个 shell 脚本上运行。
- Helm 将大量配置注入到环境变量中。查看环境以查看可用信息。
- Helm 对插件的语言没有任何设限。你可以用你喜欢的任何方式来写。
- 命令负责执行具体的帮助文本 `-h` 和 `--help`。helm 将使用 usage 和 description 对 helm help 和 helm help myplugin 进行处理，但不会处理 `helm myplugin --help`。

## 下载器插件

默认情况下，Helm 可以使用 HTTP/S 获取图表。从 Helm 2.4.0 开始，插件可以从任意源下载 chart。

插件应在 plugin.yaml 文件（顶层）中声明这个特殊功能：

```
downloaders:
- command: "bin/mydownloader"
  protocols:
  - "myprotocol"
  - "myprotocols"
```

如果安装了这样的插件，Helm 可以通过调用 `command` 指定的协议方案与存储库 repo 进行交互。特殊存储库应与常规存储库类似添加：特殊存储库 `helm repo add favorite myprotocol://example.com/` 的规则与常规存储库的规则相同：Helm 必须能够下载 index.yaml 文件以发现并缓存可用 charts 列表。

定义的命令将使用以下方案调用： `command certFile keyFile caFile full-URL`。SSL 凭证来自存储在 `$HELM_HOME/repository/repositories.yaml` 其中的 repo 定义, 。下载器插件将原始内容转储到 stdout 并在 stderr 上报告错误。

## 环境变量

当 Helm 执行插件时，它将外部环境传递给插件，并且还会注入一些其他环境变量。

类似 `KUBECONFIG` 的变量将为插件设置，如果他们设置在外部环境变量中。

保证以下变量设置：

- HELM_PLUGIN：插件目录的路径
- HELM_PLUGIN_NAME：插件的名称，正如 `helm` 所调用的。所以 `helm myplug` 会有简称 `myplug`。
- HELM_PLUGIN_DIR：包含该插件的目录。
- HELM_BIN：`helm` 命令的路径（由用户执行）。
- HELM_HOME：Helm 的 home 的路径。
- HELM_PATH_*：重要 Helm 文件和目录的路径存储在前缀为 `HELM_PATH` 的环境变量中。
- TILLER_HOST：Tiller 的 `domain:port`。如果创建隧道，则会指向隧道的本地端点。否则，它会指向 ``$HELM_HOST`，`--host` 或默认主机（按照优先级的规则）。

虽然 `HELM_HOST` 可以设置，但不能保证它会指向正确的 Tiller 实例。这是为了允许插件开发人员在插件本身需要手动配置连接时以其原始状态进行访问 `HELM_HOST` 。

## 关于 `useTunnel`

如果插件指定 `useTunnel: true`，Helm 将执行以下操作（按顺序）：

1. 解析全局标志和环境
2. 创建隧道
3. 设置 `TILLER_HOST`
4. 执行插件
5. 关闭隧道

命令退出后，隧道即被删除。因此，一个进程要使用该隧道，它不能是后台进程。

## 关于参数标记解析

在执行插件时，Helm 会解析全局标志以供自己使用。其中一些参数标志不会传递给插件。

- `--debug`：如果已指定，`$HELM_DEBUG` 则设为 `1`
- `--home`：这被转换为 `$HELM_HOME`
- `--host`：这被转换为 `$HELM_HOST`
- `--kube-context`：将丢弃。如果你的插件使用 `useTunnel`，这是用来为你设置隧道的。

`-h`和`--help`，插件应该显示帮助文本，然后退出。在所有其他情况下，插件可以根据需要使用参数标志。
