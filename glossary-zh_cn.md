# Helm 词汇表

Helm 使用一些特殊术语来描述体系结构的组件。

## Chart

包含足以将一组 Kubernetes 资源安装到 Kubernetes 集群中的信息的 Helm 软件包。

Chart 包含 Chart.yaml 文件以及模板，默认值（values.yaml）和依赖关系。

Chart 是在定义良好的目录结构中开发的，然后打包成一个称为 chart 压缩包的压缩格式。

## Chart 压缩包

一个 chart 压缩包是一个 tar 打包和 gzip 压缩（签名可选）的 chart。

## Chart 依赖（Subcharts）

Chart 可能依赖于其他 chart。有两种方式可能会出现依赖性：

- 软依赖性：如果没有在集群中安装另一个 chart，chart 可能无法正常工作。Helm 不为这种情况提供工具。在这种情况下，依赖关系可以单独管理。
- 硬性依赖性：chart 可能包含（在其 `charts/` 目录内）其所依赖的另一个 chart。在这种情况下，安装 chart 将安装它的所有依赖关系。在这种情况下，chart 及其依赖关系作为集合进行管理。

当一个 chart 打包（通过 `helm package`）时，它的所有硬依赖关系都与它捆绑在一起。

## Chart 版本

根据 [SemVer 2
spec](https://semver.org) 规范对 chart 进行版本控制。每个 chart 上都需要一个版本号。

## Chart.yaml

有关 chart 的信息存储在名为 Chart.yaml 的特殊文件中。每个 chart 都必须有这个文件。

## Helm（和 helm）

Helm 是 Kubernetes 的软件包管理员。由于操作系统软件包管理器可以轻松在 OS 上安装工具，因此 Helm 可以轻松将应用程序和资源安装到 Kubernetes 群集中。

虽然 Helm 是该项目的名称，命令行客户端也被命名 helm。按照惯例，当谈到这个项目时，Helm 被大写。在谈到客户时，掌舵是小写的。

## Helm Home (HELM_HOME)

Helm 客户端将信息存储在称为 helm home 的本地目录中 。默认情况下，这是在 `$HOME/.helm` 目录中。

该目录包含配置和缓存数据，并由 `helm init` 创建。





## Kube Config（KUBECONFIG）

Helm 客户端通过使用 Kube 配置文件格式的文件来了解 Kubernetes 集群。默认情况下，Helm 尝试在 `kubectl` 创建的地方找到这个文件（`$HOME/.kube/config`）。

## Lint（Linting）

Lint chart 是验证它遵循约定和 Helm chart 标准的要求。Helm 提供了执行此操作的工具，特别是 `helm lint` 命令。

## 出处（出处文件）

Helm chart 可能伴随着一个出处文件，该文件提供关于 chart 来自哪里以及它包含什么的信息。

出处文件是 Helm 安全的一部分。出处包含 chart 压缩文件，Chart.yaml 数据和签名块（OpenPGP“clearsign” 块）的加密哈希。当与钥匙串结合使用时，这为 chart 用户提供了以下功能：

- 验证 chart 是否由可信方签署
- 验证 chart 文件没有被篡改
- 验证 chart 元数据（Chart.yaml）的内容
- 快速将 chart 与其出处数据进行匹配

Provenance 文件具有. prov 扩展名，可以从 chart 存储库服务器或任何其他 HTTP 服务器提供。

## Release

当安装 chart 时，Tiller（Helm 服务器）创建一个 Release 来跟踪该安装。

单个 chart 可以多次安装到同一个群集中，并创建许多不同的 release。例如，可以通过 `helm install` 以不同的 release 名称运行三次来安装三个 PostgreSQL 数据库。

（在 2.0.0-Alpha.1 之前，release 被称为 deployment，但这造成了与 Kubernetes Deployment 类型的混淆。）

## Release 版本号

单个版本可以多次更新。顺序计数器用于在 release 更改时跟踪 release。通过 `helm install` 第一次安装后，release 版本的版本号为 1. 每次发布 release 升级或回滚时，版本号都会增加。

## 回滚

Release 可以升级到更新 chart 或配置。但是，由于发布历史已存储，release 版本也可以回滚到以前的版本号。这是通过 `helm rollback` 命令完成的。

重要的是，回滚版本将获得新版本号。


Operation | Release Number
----------|---------------
install   | release 1
upgrade   | release 2
upgrade   | release 3
rollback 1| release 4（但运行与 release 1 相同的配置）

上表说明了如何在安装，升级和回滚都会增加版本号。

## Tiller

Tiller 是 Helm 的集群组件。它直接与 Kubernetes API 服务器交互以安装，升级，查询和删除 Kubernetes 资源。它还存储代表 release 的对象。

## Repository（Repo，Chart Repository）

Helm chart 可以存储在专用的 HTTP 服务器上，称为 chart 存储库（存储库或库）。

chart 存储库服务器是一个简单的 HTTP 服务器，可以提供 index.yaml 描述一批 chart 的文件，并提供有关每个 chart 可从哪里下载的信息。（许多 chart 存储库同时保存 chart 以及 index.yaml 文件。）

Helm 客户端可以指向零个或多个 chart 存储库。默认情况下，Helm 客户端指向 stable 官方 Kubernetes chart 存储库。

## Values（值文件，values.yaml）

Values 提供了一种用自己的信息覆盖模板默认值的方法。

Helm chart 是 “参数化” 的，这意味着 chart 开发人员可能会公开可在安装时被覆盖的配置。例如，chart 可能会公开 username 允许为服务设置用户名的字段。

这些暴露的变量在 Helm 说法中被称为值 values。

values可以在`helm install`和`helm upgrade`操作时设置，或通过直接，或通过上传values.yaml 文件。
