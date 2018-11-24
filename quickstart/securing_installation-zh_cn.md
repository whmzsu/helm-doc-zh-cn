# 安全安装
Helm 是一款强大而灵活的 Kubernetes 软件包管理和运维工具。使用默认安装命令 `helm init-` 可以快速轻松地安装它和 **Tiller**，与 Helm 相对应的服务端组件。

但是，默认安装没有启用任何安全配置。使用这种类型的安装在下面的场景下是完全合适的，在没有安全问题或几乎没有安全问题的群集时可以使用这种安装方式，例如使用 Minikube 进行本地开发，或者使用在专用网络中，安全性良好且无数据共享或无其他用户或团队。如果是这种情况，那么默认安装很合适，但请记住：权力越大，责任越大。决定使用默认安装时始终要注意相应的安全问题。

## 谁需要安全配置？
对于以下类型的集群，我们强烈建议使用正确的安全配置应用于 Helm 和 Tiller，以确保集群，集群中的数据以及它所连接的网络的安全性。

- 暴露于不受控制的网络环境的群集：不受信任的网络参与者可以访问群集，也可以访问网络环境的不受信任的应用程序。
- 许多人使用的群集 - 多租户群集 - 作为共享环境
- 有权访问或使用高价值数据或任何类型网络的群集

通常，像这样的环境被称为 _生产等级_ 或 _生产质量_ 的环境，因为任何因滥用集群而对任何公司造成的损害对于客户，对公司本身或者两者都是深远的。一旦损害风险变得足够高，无论实际风险如何，都需要确保集群的安全完整性。

要为环境正确配置安装，必须：

- 了解群集的安全上下文
- 选择合适的 helm 安装的最佳实践

以下假定有一个 Kubernetes 配置文件（一个 kubeconfig 文件），或者有一个用于访问群集的文件。

## 了解群集的安全上下文
helm init 将 Tiller 安装到 kube-system 名称空间中的集群中，而不应用任何 RBAC 规则。这适用于本地开发和其他私人场景，因为它可以让立即开始工作。它还使你能够继续使用没有基于角色的访问控制（RBAC）支持的 Kubernetes 群集来运行 Helm，直到可以将工作负载移动到更新的 Kubernetes 版本。

在 Tiller 安全安装时，需要考虑四个主要方面：

1. 基于角色的访问控制或 RBAC
2. Tiller 的 gRPC 端点及 Helm 的使用情况
3. Tiller 的 release 信息
4. Helm harts

### RBAC
Kubernetes 的最新版本采用基于角色的访问控制（[RBAC] (https://en.wikipedia.org/wiki/Role-based_access_control) ）系统（与现代操作系统一样），以帮助缓解证书被滥用或存在错误时可能造成的损害。即使在身份被劫持的情况下，这个身份在受控空间也只有这么多的权限。这有效地增加了一层安全性，以限制使用该身份进行攻击的范围。

Helm 和 Tiller 在安装，删除和修改逻辑应用程序时，可以包含许多服务交互。因此，它的使用通常涉及整个集群的操作，在多租户集群中意味着 Tiller 安装必须非常小心才能访问整个集群，以防止不正确的安全活动。

特定用户和团队 - 开发人员，运维人员，系统和网络管理员 - 需要他们自己的群集分区，以便他们可以使用 Helm 和 Tiller，而不会冒着集群其他分区的风险。这需要启用 RBAC 的 Kubernetes 集群，并配置 Tiller 的 RBAC 权限。有关在 Kubernetes 中使用 RBAC 的更多信息，请参阅使用 RBAC 授权 [Using RBAC Authorization](rbac-zh_cn.md)。

#### Tiller 和用户权限
当前情况下的 Tiller 不提供将用户凭据映射到 Kubernetes 内的特定权限的方法。当 Tiller 在集群内部运行时，它将使用其服务帐户的权限运行。如果没有服务帐户名称提供给 Tiller，它将使用该名称空间的默认服务帐户运行。这意味着该服务器上的所有 Tiller 操作均使用 Tiller pod 的凭据和权限执行。

为了合适的限制 Tiller 本身的功能，标准 Kubernetes RBAC 机制必须配置到 Tiller 上，包括角色和角色绑定，这些角色明确的限制了 Tiller 实例可以安装什么以及在哪里安装。

这种情况在未来可能会改变。社区有几种方法可以解决这个问题，采用客户端权限而不是 Tiller 权限的情况下，活动的权限取决于 Pod 身份工作组，已经解决了的安全的一般性问题。

### Tiller gRPC 端点和 TLS
在默认安装中，Tiller 提供的 gRPC 端点在集群内部（不在集群外部）可用，不需要应用认证配置。如果不应用身份验证，集群中的任何进程都可以使用 gRPC 端点在集群内执行操作。在本地或安全的专用群集中，这可以实现快速使用并且是合适的。（当在集群外部运行时，Helm 通过 Kubernetes API 服务器进行身份验证，以达到 Tiller，利用现有的 Kubernetes 身份验证支持。）

The following two sub-sections describe options of how to setup Tiller so there isn't an unauthenticated endpoint (i.e. gRPC) in your cluster.

#### Enabling TLS

(Note that out of the two options, this is the recommended one for Helm 2.)
共享和生产群集 - 大多数情况下 - 应至少使用 Helm 2.7.2，并为每个 Tiller gRPC 端点配置 TLS，以确保群集内 gRPC 端点的使用仅适用于该端点的正确身份验证标识。这样做可以在任意数量的 namespace 中部署任意数量的 Tiller 实例，任何 gRPC 端点未经授权不可使用。使用 Helm `init` 和 `--tiller-tls-verify` 选择安装启用 TLS 的 Tiller, 并验证远程证书，所有其他 Helm 命令都应该使用该 `--tls` 选项。

有关正确配置并使用 TLS 的 Tiller 和 Helm 的正确步骤的更多信息，请参阅下面的章节 [Best Practices](#Helm 和 Tiller 安全最佳实践) 以及 Helm 和 Tiller 使用 SSL[在 Helm 和 Tiller 之间使用 SSL](tiller_ssl-zh_cn.md)。

当 Helm 客户端从群集外部连接时，Helm 客户端和 API 服务器之间的安全性由 Kubernetes 本身管理。你可能需要确保这个链接是安全的。请注意，如果使用上面建议的 TLS 配置，则 Kubernetes API 服务器也无法访问客户端和 Tiller 之间的加密消息。

#### Running Tiller Locally

与上面的章节 [Enabling TLS](#Enabling TLS) 相反, 本节不涉及在集群中运行分蘖服务器 pod（就其价值而言，它符合当前情况 [helm v3 proposal](https://github.com/helm/community/blob/master/helm-v3/000-helm-v3.md)), 因此没有 gRPC 端点（因此不需要创建和管理 TLS 证书来保护每个 gRPC 端点）。

步骤:
 * 获取最新安装包 [GitHub release page](https://github.com/helm/helm/releases), 解压缩，并将 `helm` and `tiller` 放到你的路径 `$PATH`.
 * "服务端": 运行 `tiller --storage=secret`. (`tiller` 默认监听 ":44134" 通过 `--listen` 参数.)
 * "客户端": 在另一个终端 (同一台运行 `tiller` 的机器上): 运行 `export HELM_HOST=:44134`, 然后运行 `helm`.

### Tiller Release 信息
由于历史原因，Tiller 将其 release 信息存储在 ConfigMaps 中。我们建议将默认设置更改为 Secrets。

Secrets 是 Kubernetes 用于保存被认为是敏感的配置数据的可接受的方法。尽管 secrets 本身并不提供很多保护，但 Kubernetes 集群管理软件经常将它们与其他对象区别开来。因此，我们建议使用 secrets 来存储 release 信息。

启用此功能目前需要在 Tiller 部署时设置参数 `--storage=secret`。这需要直接修改 deployment 或使用 `helm init --override 'spec.template.spec.containers[0].command'='{/tiller,--storage=secret}'`，因为当前没有 helm init 参数可供执行此操作。

### 关于 chart
由于 Helm 的相对生命周期，Helm chart 生态系统的发展并没有考虑到整个集群的控制，这在开发人员来说，是完全合理的。但是，chart 是一种不仅可以安装可能已经验证或可能未验证的容器的包，它也可以安装到多个 namespace 中。

与所有共享的软件一样，在受控或共享的环境中，必须在安装之前验证自己安装的所有软件。如果已经通过 TLS 配置安装了 Tiller，并且只有一个或部分 namespace 的权限，某些 chart 可能无法安装 - 在这些环境中，这正是你想要的。如果需要使用 chart，可能必须与创建者一起工作或自行修改它，以便在应用了适当的 RBAC 规则的多租户群集中安全地使用它。`helm template` 命令在本地呈现 chart 并显示输出。

一旦通过检查，可以使用 Helm 的工具来确保使用的 chart 的出处和完整性 [ensure the provenance and integrity of charts](provenance.md)。

### gRPC 工具和安全 Tiller 配置
许多非常有用的工具直接使用 gRPC 接口，并且已经针对默认安装构建 - 它们提供了集群范围的访问 - 一旦应用了安全配置后就可能工作不正常。RBAC 策略由你或集群运维人员控制，并且可以针对该工具进行调整，或者可以将该工具配置为，在应用于 Tiller 的特定 RBAC 策略的约束范围内来正常工作。如果 gRPC 端点受到保护，则可能需要执行相同的操作：为了使用特定的 Tiller 实例，这些工具需要自己的安全 TLS 配置。RBAC 策略和 gRPC 工具一起配置的安全 gRPC 端点的组合，使你能够按照自己的需要控制群集环境。

## Helm 和 Tiller 安全最佳实践
以下指导原则重申了 Helm 和 Tiller 安全并正确使用它们的最佳方法。

1. 创建一个启用了 RBAC 的集群
2. 配置每个 Tiller gRPC 端点以使用单独的 TLS 证书
3. Release 信息应该使用 Kubernetes Secret
4. 为每个用户，团队或其他具有 `--service-account` 参数，role 和 RoleBindings 的组织安装一个 Tiller
5. `helm init` 使用 --tiller-tls-verify，其他 Helm 命令 `--tls` 来强制验证

如果遵循这些步骤，则 helm init 命令可能如下所示：

```bash
$ helm init \
--override 'spec.template.spec.containers[0].command'='{/tiller,--storage=secret}' \
--tiller-tls \
--tiller-tls-verify \
--tiller-tls-cert=cert.pem \
--tiller-tls-key=key.pem \
--tls-ca-cert=ca.pem \
--service-account=accountname
```

此命令将通过gRPC进行强身份验证，release信息存储在Kubernetes Secret，并使用RBAC策略的服务帐户安装启动Tiller。
