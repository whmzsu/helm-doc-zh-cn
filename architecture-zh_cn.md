# Kubernetes Helm 架构

本文档介绍了 Helm 高层次体系结构。

## Helm 的目的
Helm 是管理称为 chart 的 Kubernetes 包的工具。Helm 可以做到以下几点：

- 从头开始创建新 chart
- 将 chart 打包成 chart 归档（tgz）文件
- 与存储 chart 的 chart 存储库交互
- 安装或卸载 chart 到现有的 Kubernetes 集群中
- 管理用 Helm 安装的 chart 的 release 周期

对于 Helm，有三个重要的概念：

- chart 是创建 Kubernetes 应用程序实例所需的一系列信息。
- 配置包含可以合并到打包 chart 以创建可发布对象的配置信息。
- Release 是一个的运行实例的 chart，具有特定的组合配置。

## 组件
Helm 有两个主要部分：

Helm Client 是最终用户的命令行客户端。客户端负责以下部分：

- 本地 chart 开发
- 管理存储库
- 与 Tiller 服务交互
- 发送要安装的 chart
- 查询有关发布的信息
- 请求升级或卸载现有 release

**Tiller Server** 是一个集群内服务，与 Helm 客户端进行交互，并与 Kubernetes API 服务进行交互。服务负责以下内容：

- 监听来自 Helm 客户端的传入请求
- 结合 chart 和配置来构建发布
- 将 chart 安装到 Kubernetes 中，然后跟踪后续 release
- 通过与 Kubernetes 交互来升级和卸载 chart

简而言之，客户端负责管理 chart，而服务端负责管理 release。

## 部署

Helm 客户端使用 Go 编程语言编写，并使用 gRPC 协议套件与 Tiller 服务进行交互。

Tiller 服务也用 Go 编写。它提供了一个与客户端连接的 gRPC 服务，它使用 Kubernetes 客户端库与 Kubernetes 进行通信。目前，该库使用 REST + JSON。

Tiller 服务将信息存储在位于 Kubernetes 内的 ConfigMaps 中。它不需要自己的数据库。

如有可能，配置文件用YAML编写。
