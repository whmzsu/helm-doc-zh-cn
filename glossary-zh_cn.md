# Helm词汇表

Helm使用一些特殊术语来描述体系结构的组件。

## Chart

包含足以将一组Kubernetes资源安装到Kubernetes集群中的信息的Helm软件包。

Chart包含Chart.yaml文件以及模板，默认值（values.yaml）和依赖关系。

Chart是在定义良好的目录结构中开发的，然后打包成一个称为chart压缩包的压缩格式。

## Chart压缩包

一个chart压缩包是一个tar打包和gzip压缩（签名可选）的chart。

## Chart依赖（Subcharts）

Chart可能依赖于其他chart。有两种方式可能会出现依赖性：

- 软依赖性：如果没有在集群中安装另一个chart，chart可能无法正常工作。Helm不为这种情况提供工具。在这种情况下，依赖关系可以单独管理。
- 硬性依赖性：chart可能包含（在其`charts/`目录内）其所依赖的另一个chart。在这种情况下，安装chart将安装它的所有依赖关系。在这种情况下，chart及其依赖关系作为集合进行管理。

当一个chart打包（通过`helm package`）时，它的所有硬依赖关系都与它捆绑在一起。

## Chart版本

根据[SemVer 2
spec](http://semver.org)规范对chart进行版本控制。每个chart上都需要一个版本号。

## Chart.yaml

有关chart的信息存储在名为Chart.yaml的特殊文件中。每个chart都必须有这个文件。

## Helm（和helm）

Helm是Kubernetes的软件包管理员。由于操作系统软件包管理器可以轻松在OS上安装工具，因此Helm可以轻松将应用程序和资源安装到Kubernetes群集中。

虽然Helm是该项目的名称，命令行客户端也被命名helm。按照惯例，当谈到这个项目时，Helm被大写。在谈到客户时，掌舵是小写的。

## Helm Home (HELM_HOME)

Helm客户端将信息存储在称为helm home的本地目录中 。默认情况下，这是在`$HOME/.helm`目录中。

该目录包含配置和缓存数据，并由`helm init`创建。





## Kube Config（KUBECONFIG）

Helm客户端通过使用Kube配置文件格式的文件来了解Kubernetes集群。默认情况下，Helm尝试在`kubectl`创建的地方找到这个文件（`$HOME/.kube/config`）。

## Lint（Linting）

Lint chart是验证它遵循约定和Helm chart标准的要求。Helm提供了执行此操作的工具，特别是`helm lint`命令。

## 出处（出处文件）

Helm chart可能伴随着一个出处文件，该文件提供关于chart来自哪里以及它包含什么的信息。

出处文件是Helm安全的一部分。出处包含chart压缩文件，Chart.yaml数据和签名块（OpenPGP“clearsign”块）的加密哈希。当与钥匙串结合使用时，这为chart用户提供了以下功能：

- 验证chart是否由可信方签署
- 验证chart文件没有被篡改
- 验证chart元数据（Chart.yaml）的内容
- 快速将chart与其出处数据进行匹配

Provenance文件具有.prov扩展名，可以从chart存储库服务器或任何其他HTTP服务器提供。

## Release

当安装chart时，Tiller（Helm服务器）创建一个Release来跟踪该安装。

单个chart可以多次安装到同一个群集中，并创建许多不同的release。例如，可以通过`helm install`以不同的release名称运行三次来安装三个PostgreSQL数据库。

（在2.0.0-Alpha.1之前，release被称为deployment，但这造成了与Kubernetes Deployment类型的混淆。）

## Release版本号

单个版本可以多次更新。顺序计数器用于在release更改时跟踪release。通过`helm install`第一次安装后，release版本的版本号为 1.每次发布release升级或回滚时，版本号都会增加。

## 回滚

Release可以升级到更新chart或配置。但是，由于发布历史已存储，release版本也可以回滚到以前的版本号。这是通过`helm rollback`命令完成的。

重要的是，回滚版本将获得新版本号。


Operation | Release Number
----------|---------------
install   | release 1
upgrade   | release 2
upgrade   | release 3
rollback 1| release 4（但运行与release 1相同的配置）

上表说明了如何在安装，升级和回滚都会增加版本号。

## Tiller

Tiller是Helm的集群组件。它直接与Kubernetes API服务器交互以安装，升级，查询和删除Kubernetes资源。它还存储代表release的对象。

## Repository（Repo，Chart Repository）

Helm chart可以存储在专用的HTTP服务器上，称为chart存储库（存储库或库）。

chart存储库服务器是一个简单的HTTP服务器，可以提供index.yaml描述一批chart的文件，并提供有关每个chart可从哪里下载的信息。（许多chart存储库同时保存chart以及index.yaml文件。）

Helm客户端可以指向零个或多个chart存储库。默认情况下，Helm客户端指向stable官方Kubernetes chart存储库。

## Values（值文件，values.yaml）

Values提供了一种用自己的信息覆盖模板默认值的方法。

Helm chart是“参数化”的，这意味着chart开发人员可能会公开可在安装时被覆盖的配置。例如，chart可能会公开username允许为服务设置用户名的字段。

这些暴露的变量在Helm说法中被称为值values。

values可以在`helm install`和`helm upgrade`操作时设置，或通过直接，或通过上传values.yaml 文件。
