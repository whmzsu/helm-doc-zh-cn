# 插件指南

Helm 2.1.0引入了客户端Helm 插件_plugin_的概念。插件是一种可以通过helm CLI 访问的工具，但它不是内置Helm代码库的一部分。

现有的插件可以在相关部分[related](related.md#helm-plugins)找到或者通过搜索[Github](https://github.com/search?q=topic%3Ahelm-plugin&type=Repositories)。

本指南介绍了如何使用和创建插件。

## 概述

Helm插件是与Helm无缝集成的附加工具。它们提供了扩展Helm核心功能集的方法，但不需要将每个新功能都通过Go语言写入并添加到核心工具中。

Helm插件具有以下功能：

- 可以在Helm安装中添加和删除它们，而不会影响核心Helm工具。
- 它们可以用任何编程语言编写。
- 他们与Helm集成，并出现在helm help和其他地方。

Helm插件放置在$(helm home)/plugins。

Helm插件模型部分建模在​​Git的插件模型上。为此，有时可能会听到helm称为瓷层 _porcelain_，插件是管道 _plumbing_。这是揭示Helm提供用户体验和顶级处理逻辑，而插件则是执行所需操作的“细节工作”的简略说法。

## 安装插件

使用 `$ helm plugin install <path|url>`命令安装插件。可以将路径设置为本地文件系统上的插件或远程VCS repo的URL。`helm plugin install`命令克隆或复制该插件的路径/URL到给定的`$(helm home)/plugins`

```bash
$ helm plugin install https://github.com/technosophos/helm-template
```

如果你有一个插件tar分发版，只需将插件解压到 $(helm home)/plugins目录中即可。

也可以通过直接从URL安装tarball插件`helm plugin install` http://domain/path/to/plugin.tar.gz

## 构建插件

在很多方面，插件类似于chart。每个插件都有一个顶级目录，然后是一个`plugin.yaml` 文件。

```
$(helm home)/plugins/
  |- keybase/
      |
      |- plugin.yaml
      |- keybase.sh

```

在上面的例子中，keybase插件包含在名为keybase的目录中。它有两个文件：（`plugin.yaml`必需）和一个可执行脚本`keybase.sh`（可选）。

插件的核心是一个简单的YAML文件`plugin.yaml`。这是一个插件的一个插件YAML，它增加了对Keybase操作的支持：

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

`name`是插件的名称。当Helm执行插件时，这是它将使用的名称（例如，`helm NAME`将调用此插件）。

_`name`应该匹配目录名称。_ 在我们上面的例子中，这意味着插件`name: keybase`应该在一个名为keybase的目录中。

`name`的限制：

- `name`不能一个现有的helm顶级命令重复。
- name必须限制为ASCII az，AZ，0-9 `_`和`-。

`version`是插件的SemVer 2版本。 `usage`和`description`都用于生成命令的帮助文本。

`ignoreFlags`告诉H​​elm 不会将参数传递给插件。所以，如果一个插件被`helm myplugin --foo`调用，并且`ignoreFlags: true`，那么`--foo` 将被忽略。

`useTunnel`指示插件需要一个隧道去连接Tiller。这在任何时候插件与Tiller对接都应该设置为true 。它会使Helm打开一个隧道，然后``$TILLER_HOST`为该隧道设置正确的本地地址。不用担心：如果Helm由于Tiller在本地运行而检测到隧道是不必啊哟的，它就不会创建隧道。

最后，也是最重要的是，`command`，是这个插件在调用时会执行的命令。在执行插件之前会插入环境变量。上面的模式说明了指出插件程序所在位置的首选方式。

有一些使用插件命令的策略：

- 如果插件包含可执行文件`command:`，则应将可执行文件打包到插件目录中。
- `command:`将在执行前展开任何环境变量。`$HELM_PLUGIN_DIR`将指向插件目录。
- 该命令本身不在shell中执行。所以你不能在一个shell脚本上运行。
- Helm将大量配置注入到环境变量中。查看环境以查看可用信息。
- Helm对插件的语言没有任何设限。你可以用你喜欢的任何方式来写。
- 命令负责执行具体的帮助文本`-h`和`--help`。helm将使用usage和description对helm help和helm help myplugin进行处理，但不会处理`helm myplugin --help`。

## 下载器插件

默认情况下，Helm可以使用HTTP/S获取图表。从Helm 2.4.0开始，插件可以从任意源下载chart。

插件应在plugin.yaml文件（顶层）中声明这个特殊功能：

```
downloaders:
- command: "bin/mydownloader"
  protocols:
  - "myprotocol"
  - "myprotocols"
```

如果安装了这样的插件，Helm可以通过调用`command`指定的协议方案与存储库repo进行交互。特殊存储库应与常规存储库类似添加：特殊存储库`helm repo add favorite myprotocol://example.com/` 的规则与常规存储库的规则相同：Helm必须能够下载index.yaml文件以发现并缓存可用charts列表。

定义的命令将使用以下方案调用： `command certFile keyFile caFile full-URL`。SSL凭证来自存储在`$HELM_HOME/repository/repositories.yaml`其中的repo定义, 。下载器插件将原始内容转储到stdout并在stderr上报告错误。

## 环境变量

当Helm执行插件时，它将外部环境传递给插件，并且还会注入一些其他环境变量。

类似`KUBECONFIG`的变量将为插件设置，如果他们设置在外部环境变量中。

保证以下变量设置：

- HELM_PLUGIN：插件目录的路径
- HELM_PLUGIN_NAME：插件的名称，正如`helm`所调用的。所以 `helm myplug`会有简称`myplug`。
- HELM_PLUGIN_DIR：包含该插件的目录。
- HELM_BIN：`helm`命令的路径（由用户执行）。
- HELM_HOME：Helm的home的路径。
- HELM_PATH_*：重要Helm文件和目录的路径存储在前缀为`HELM_PATH`的环境变量中。
- TILLER_HOST：Tiller的`domain:port`。如果创建隧道，则会指向隧道的本地端点。否则，它会指向``$HELM_HOST`，`--host`或默认主机（按照优先级的规则）。

虽然`HELM_HOST` 可以设置，但不能保证它会指向正确的Tiller实例。这是为了允许插件开发人员在插件本身需要手动配置连接时以其原始状态进行访问`HELM_HOST` 。

## 关于 useTunnel

如果插件指定`useTunnel: true`，Helm将执行以下操作（按顺序）：

1. 解析全局标志和环境
2. 创建隧道
3. 设置TILLER_HOST
4. 执行插件
5. 关闭隧道
command 退出后，隧道即被移除。因此，假定一个进程要使用该隧道，它不能是后台进程，。

## 关于参数标记解析

在执行插件时，Helm会解析全局标志以供自己使用。其中一些参数标志不会传递给插件。

- `--debug`：如果已指定，`$HELM_DEBUG`则设为`1`
- `--home`：这被转换为 `$HELM_HOME`
- `--host`：这被转换为 `$HELM_HOST`
- `--kube-context`：将丢弃。如果你的插件使用`useTunnel`，这是用来为你设置隧道的。

`-h`和`--help`，插件应该显示帮助文本，然后退出。在所有其他情况下，插件可以根据需要使用参数标志。
