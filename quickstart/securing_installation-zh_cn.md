# 安全安装
Helm是一款强大而灵活的Kubernetes软件包管理和运维工具。使用默认安装命令`helm init-` 可以快速轻松地安装它和 **Tiller**，与Helm相对应的服务端组件。

但是，默认安装没有启用任何安全配置。使用这种类型的安装在下面的场景下是完全合适的，在没有安全问题或几乎没有安全问题的群集时可以使用这种安装方式，例如使用Minikube进行本地开发，或者使用在专用网络中，安全性良好且无数据共享或无其他用户或团队。如果是这种情况，那么默认安装很合适，但请记住：权力越大，责任越大。决定使用默认安装时始终要注意相应的安全问题。

## 谁需要安全配置？
对于以下类型的集群，我们强烈建议使用正确的安全配置应用于Helm和Tiller，以确保集群，集群中的数据以及它所连接的网络的安全性。

- 暴露于不受控制的网络环境的群集：不受信任的网络参与者可以访问群集，也可以访问网络环境的不受信任的应用程序。
- 许多人使用的群集 - 多租户群集 - 作为共享环境
- 有权访问或使用高价值数据或任何类型网络的群集

通常，像这样的环境被称为 _生产等级_ 或 _生产质量_ 的环境，因为任何因滥用集群而对任何公司造成的损害对于客户，对公司本身或者两者都是深远的。一旦损害风险变得足够高，无论实际风险如何，都需要确保集群的安全完整性。

要为环境正确配置安装，必须：

- 了解群集的安全上下文
- 选择合适的helm安装的最佳实践

以下假定有一个Kubernetes配置文件（一个kubeconfig文件），或者有一个用于访问群集的文件。

## 了解群集的安全上下文
helm init将Tiller安装到kube-system名称空间中的集群中，而不应用任何RBAC规则。这适用于本地开发和其他私人场景，因为它可以让立即开始工作。它还使你能够继续使用没有基于角色的访问控制（RBAC）支持的Kubernetes群集来运行Helm，直到可以将工作负载移动到更新的Kubernetes版本。

在Tiller安全安装时，需要考虑四个主要方面：

1. 基于角色的访问控制或RBAC
2. Tiller的gRPC端点及Helm的使用情况
3. Tiller的release信息
4. Helm harts

### RBAC
Kubernetes的最新版本采用基于角色的访问控制（或RBAC）系统（与现代操作系统一样），以帮助缓解证书被滥用或存在错误时可能造成的损害。即使在身份被劫持的情况下，这个身份在受控空间也只有这么多的权限。这有效地增加了一层安全性，以限制使用该身份进行攻击的范围。

Helm和Tiller在安装，删除和修改逻辑应用程序时，可以包含许多服务交互。因此，它的使用通常涉及整个集群的操作，在多租户集群中意味着Tiller安装必须非常小心才能访问整个集群，以防止不正确的安全活动。

特定用户和团队 - 开发人员，运维人员，系统和网络管理员 - 需要他们自己的群集分区，以便他们可以使用Helm和Tiller，而不会冒着集群其他分区的风险。这需要启用RBAC的Kubernetes集群，并配置Tiller的RBAC权限。有关在Kubernetes中使用RBAC的更多信息，请参阅使用RBAC授权[Using RBAC Authorization](rbac-zh_cn.md)。

#### Tiller和用户权限
当前情况下的Tiller不提供将用户凭据映射到Kubernetes内的特定权限的方法。当Tiller在集群内部运行时，它将使用其服务帐户的权限运行。如果没有服务帐户名称提供给Tiller，它将使用该名称空间的默认服务帐户运行。这意味着该服务器上的所有Tiller操作均使用Tiller pod的凭据和权限执行。

为了合适的限制Tiller本身的功能，标准Kubernetes RBAC机制必须配置到Tiller上，包括角色和角色绑定，这些角色明确的限制了Tiller实例可以安装什么以及在哪里安装。

这种情况在未来可能会改变。社区有几种方法可以解决这个问题，采用客户端权限而不是Tiller权限的情况下，活动的权限取决于Pod身份工作组，已经解决了的安全的一般性问题。

### Tiller gRPC端点和TLS
在默认安装中，Tiller提供的gRPC端点在集群内部（不在集群外部）可用，不需要应用认证配置。如果不应用身份验证，集群中的任何进程都可以使用gRPC端点在集群内执行操作。在本地或安全的专用群集中，这可以实现快速使用并且是合适的。（当在集群外部运行时，Helm通过Kubernetes API服务器进行身份验证，以达到Tiller，利用现有的Kubernetes身份验证支持。）

共享和生产群集 - 大多数情况下 - 应至少使用Helm 2.7.2，并为每个Tiller gRPC端点配置TLS，以确保群集内gRPC端点的使用仅适用于该端点的正确身份验证标识。这样做可以在任意数量的namespace中部署任意数量的Tiller实例，任何gRPC端点未经授权不可使用。使用Helm `init` --tiller-tls-verify选择安装启用TLS的Tiller,并验证远程证书，所有其他Helm命令都应该使用该--tls选项。

有关正确配置并使用TLS的Tiller和Helm的正确步骤的更多信息，请参阅Helm和Tiller使用SSL[Using SSL between Helm and Tiller](tiller_ssl.md)。

当Helm客户端从群集外部连接时，Helm客户端和API服务器之间的安全性由Kubernetes本身管理。你可能需要确保这个链接是安全的。请注意，如果使用上面建议的TLS配置，则Kubernetes API服务器也无法访问客户端和Tiller之间的未加密消息。

### Tiller Release信息
由于历史原因，Tiller将其release信息存储在ConfigMaps中。我们建议将默认设置更改为Secrets。

Secrets是Kubernetes用于保存被认为是敏感的配置数据的可接受的方法。尽管secrets本身并不提供很多保护，但Kubernetes集群管理软件经常将它们与其他对象区别开来。因此，我们建议使用secrets来存储release信息。

启用此功能目前需要在Tiller部署时设置参数`--storage=secret`。这需要直接修改deployment或使用`helm init --override=...`，因为当前没有helm init 参数可供执行此操作。有关更多信息，请参阅 [Using --override](install.md#using---override)。

### 关于chart
由于Helm的相对生命周期，Helm chart生态系统的发展并没有考虑到整个集群的控制，这在开发人员来说，是完全合理的。但是，chart是一种不仅可以安装可能已经验证或可能未验证的容器的包，它也可以安装到多个namespace中。

与所有共享的软件一样，在受控或共享的环境中，必须在安装之前验证自己安装的所有软件。如果已经通过TLS配置安装了Tiller，并且只有一个或部分namespace的权限，某些chart可能无法安装 - 在这些环境中，这正是你想要的。如果需要使用chart，可能必须与创建者一起工作或自行修改它，以便在应用了适当的RBAC规则的多租户群集中安全地使用它。`helm template`命令在本地呈现chart并显示输出。

一旦通过检查，可以使用Helm的工具来确保使用的chart的出处和完整性[ensure the provenance and integrity of charts](provenance.md)。

### gRPC工具和安全Tiller配置
许多非常有用的工具直接使用gRPC接口，并且已经针对默认安装构建 - 它们提供了集群范围的访问 - 一旦应用了安全配置后就可能工作不正常。RBAC策略由你或集群运维人员控制，并且可以针对该工具进行调整，或者可以将该工具配置为，在应用于Tiller的特定RBAC策略的约束范围内来正常工作。如果gRPC端点受到保护，则可能需要执行相同的操作：为了使用特定的Tiller实例，这些工具需要自己的安全TLS配置。RBAC策略和gRPC工具一起配置的安全gRPC端点的组合，使你能够按照自己的需要控制群集环境。

## Helm和Tiller安全最佳实践
以下指导原则重申了Helm和Tiller安全并正确使用它们的最佳方法。

1. 创建一个启用了RBAC的集群
2. 配置每个Tiller gRPC端点以使用单独的TLS证书
3. Release信息应该使用Kubernetes Secret
4. 为每个用户，团队或其他具有`--service-account`参数，role和RoleBindings的组织安装一个Tiller
5. `helm init`使用--tiller-tls-verify，其他Helm命令`--tls`来强制验证

如果遵循这些步骤，则helm init命令可能如下所示：

```bash
$ helm init \
--tiller-tls \
--tiller-tls-verify \
--tiller-tls-cert=cert.pem \
--tiller-tls-key=key.pem \
--tls-ca-cert=ca.pem \
--service-account=accountname
```

此命令将通过gRPC进行强身份验证，并应用RBAC策略的服务帐户安装启动Tiller。
