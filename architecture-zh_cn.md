# Kubernetes Helm 架构

本文档介绍了Helm高层次体系结构。

## Helm的目的
Helm是管理称为chart的 Kubernetes包的工具。Helm可以做到以下几点：

- 从头开始创建新chart
- 将chart打包成chart归档（tgz）文件
- 与存储chart的chart存储库交互
- 安装或卸载chart到现有的Kubernetes集群中
- 管理用Helm安装的chart的release周期

对于Helm，有三个重要的概念：

- chart是创建Kubernetes应用程序实例所需的一系列信息。
- 配置包含可以合并到打包chart以创建可发布对象的配置信息。
- Release是一个的运行实例的chart，具有特定的组合配置。

## 组件
Helm有两个主要部分：

Helm Client是最终用户的命令行客户端。客户端负责以下部分：

- 本地chart开发
- 管理存储库
- 与Tiller服务交互
- 发送要安装的chart
- 查询有关发布的信息
- 请求升级或卸载现有release

**Tiller Server**是一个集群内服务，与Helm客户端进行交互，并与Kubernetes API服务进行交互。服务负责以下内容：

- 监听来自Helm客户端的传入请求
- 结合chart和配置来构建发布
- 将chart安装到Kubernetes中，然后跟踪后续release
- 通过与Kubernetes交互来升级和卸载chart

简而言之，客户端负责管理chart，而服务端负责管理release。

## 部署

Helm客户端使用Go编程语言编写，并使用gRPC协议套件与Tiller服务进行交互。

Tiller服务也用Go编写。它提供了一个与客户端连接的gRPC服务，它使用Kubernetes客户端库与Kubernetes进行通信。目前，该库使用REST + JSON。

Tiller服务将信息存储在位于Kubernetes内的ConfigMaps中。它不需要自己的数据库。

如有可能，配置文件用YAML编写。
