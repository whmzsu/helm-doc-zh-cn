# RBAC - 基于角色的访问控制

在 Kubernetes 中，确保应用程序在指定的范围内运行, 最佳的做法是，为特定的应用程序的服务帐户授予角色。要详细了解服务帐户权限请阅读 [官方 Kubernetes 文档](https://kubernetes.io/docs/admin/authorization/rbac/#service-account-permissions).

Bitnami 写了一个在集群中配置 RBAC 的 [指导](https://docs.bitnami.com/kubernetes/how-to/configure-rbac-in-your-kubernetes-cluster/)，可让你了解 RBAC 基础知识。

本指南面向希望对 Helm 限制如下权限的用户:
1. Tiller 将资源安装到特定 namespace 能力
2. 授权 Helm 客户端对 Tiller 实例的访问

## Tiller 和基于角色的访问控制


可以在配置 Helm 时使用 `--service-account <NAME>` 参数将服务帐户添加到 Tiller 。前提条件是必须创建一个角色绑定，来指定预先设置的角色 [role](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) 和服务帐户 [service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) 名称。

在前提条件下，并且有了一个具有正确权限的服务帐户，就可以像这样运行一个命令来初始化 Tiller： `helm init --service-account <NAME>`

### Example: 服务账户带有 cluster-admin 角色权限

```bash
$ kubectl create serviceaccount tiller --namespace kube-system
serviceaccount "tiller" created
```

文件 `rbac-config.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v11
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

_Note: cluster-admin 角色是在 Kubernetes 集群中默认创建的，因此不必再显式地定义它。_

```bash
$ kubectl create -f rbac-config.yaml
serviceaccount "tiller" created
clusterrolebinding "tiller" created
$ helm init --service-account tiller --history-max 200
```

### 在特定 namespace 中部署 Tiller，并仅限于在该 namespace 中部署资源

在上面的例子中，我们让 Tiller 管理访问整个集群。当然，Tiller 正常工作并不一定要为它设置集群管理员访问权限。我们可以指定 Role 和 RoleBinding 来将 Tiller 的范围限制为特定的 namespace，而不是指定 ClusterRole 或 ClusterRoleBinding。

```bash
$ kubectl create namespace tiller-world
namespace "tiller-world" created
$ kubectl create serviceaccount tiller --namespace tiller-world
serviceaccount "tiller" created
```

定义允许 Tiller 管理 namespace `tiller-world` 中所有资源的角色 ，文件 `role-tiller.yaml`:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v11
metadata:
  name: tiller-manager
  namespace: tiller-world
rules:
- apiGroups: ["","extensions","apps"]
  resources: ["*"]
  verbs: ["*"]
```

```bash
$ kubectl create -f role-tiller.yaml
role "tiller-manager" created
```

文件 `rolebinding-tiller.yaml`,

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v11
metadata:
  name: tiller-binding
  namespace: tiller-world
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: tiller-world
roleRef:
  kind: Role
  name: tiller-manager
  apiGroup: rbac.authorization.k8s.io
```

```bash
$ kubectl create -f rolebinding-tiller.yaml
rolebinding "tiller-binding" created
```

之后，运行 `helm init` 来在 `tiller-world` namespace 中安装 Tiller 。

```bash
$ helm init --service-account tiller --tiller-namespace tiller-world
$HELM_HOME has been configured at /Users/awesome-user/.helm.

Tiller (the Helm server side component) has been installed into your Kubernetes Cluster.
Happy Helming!

$ helm install nginx --tiller-namespace tiller-world --namespace tiller-world
NAME:   wayfaring-yak
LAST DEPLOYED: Mon Aug  7 16:00:16 2017
NAMESPACE: tiller-world
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod
NAME                  READY  STATUS             RESTARTS  AGE
wayfaring-yak-alpine  0/1    ContainerCreating  0         0s
```

### Example: 在一个 namespace 中部署 Tiller，并限制它在另一个 namespace 部署资源

在上面的例子中，我们让 Tiller 管理它部署所在的 namespace。现在，让我们限制 Tiller 的范围，将资源部署在不同的 namespace 中！

下面例子中，让我们在 `myorg-system` namespace 中安装 Tiller，并允许 Tiller 在 `myorg-users` namespace 中部署资源。

```bash
$ kubectl create namespace myorg-system
namespace "myorg-system" created
$ kubectl create serviceaccount tiller --namespace myorg-system
serviceaccount "tiller" created
```

在 `role-tiller.yaml` 中，定义了一个允许 Tiller 管理所有 `myorg-users` 资源的角色：

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v11
metadata:
  name: tiller-manager
  namespace: myorg-users
rules:
- apiGroups: ["","extensions","apps"]
  resources: ["*"]
  verbs: ["*"]
```

```bash
$ kubectl create -f role-tiller.yaml
role "tiller-manager" created
```

将 service account 与那个 role 绑定. `rolebinding-tiller.yaml`,

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v11
metadata:
  name: tiller-binding
  namespace: myorg-users
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: myorg-system
roleRef:
  kind: Role
  name: tiller-manager
  apiGroup: rbac.authorization.k8s.io
```

```bash
$ kubectl create -f rolebinding-tiller.yaml
rolebinding "tiller-binding" created
```
我们还需要授予 Tiller 访问权限来读取 `myorg-system` 中的 configmaps，以便它可以存储 release 信息。如 `role-tiller-myorg-system.yaml`:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v11
metadata:
  namespace: myorg-system
  name: tiller-manager
rules:
- apiGroups: ["","extensions","apps"]
  resources: ["configmaps"]
  verbs: ["*"]
```

```bash
$ kubectl create -f role-tiller-myorg-system.yaml
role "tiller-manager" created
```

相应的 role 绑定. 如 `rolebinding-tiller-myorg-system.yaml`:

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v11
metadata:
  name: tiller-binding
  namespace: myorg-system
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: myorg-system
roleRef:
  kind: Role
  name: tiller-manager
  apiGroup: rbac.authorization.k8s.io
```

```bash
$ kubectl create -f rolebinding-tiller-myorg-system.yaml
rolebinding "tiller-binding" created
```

## Helm 和基于角色的访问控制

在 pod 中运行 Helm 客户端时，为了让 Helm 客户端与 Tiller 实例进行通信，需要授予某些特权。具体来说，Helm 客户端需要能够创建 pods，转发端口并能够在 Tiller 运行的 namespace 中列出 pod（这样它才可以找到 Tiller）。

### Example: 在一个 namespace 中部署 Helm，与在另一个 namespace 中与 Tiller 交互

在这个例子中，我们将假设 Tiller 在名为 `tiller-world` 的 namespace 中运行，并且 Helm 客户端在 `helm-world` 的 namespace 中运行。默认情况下，Tiller 在 `kube-system` namespace 中运行。

如 `helm-user.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: helm
  namespace: helm-world
---
apiVersion: rbac.authorization.k8s.io/v11
kind: Role
metadata:
  name: tiller-user
  namespace: tiller-world
rules:
- apiGroups:
  - ""
  resources:
  - pods/portforward
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
---
apiVersion: rbac.authorization.k8s.io/v11
kind: RoleBinding
metadata:
  name: tiller-user-binding
  namespace: tiller-world
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: tiller-user
subjects:
- kind: ServiceAccount
  name: helm
  namespace: helm-world
```

```bash
$ kubectl create -f helm-user.yaml
serviceaccount "helm" created
role "tiller-user" created
rolebinding "tiller-user-binding" created
```
