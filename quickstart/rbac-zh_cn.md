# RBAC-基于角色的访问控制

在Kubernetes中，最佳的做法是，为特定的应用程序的服务帐户授予角色,确保应用程序在指定的范围内运行。要详细了解服务帐户权限请阅读[官方Kubernetes文档](https://kubernetes.io/docs/admin/authorization/rbac/#service-account-permissions).

Bitnami写了一个在集群中配置RBAC的[指导](https://docs.bitnami.com/kubernetes/how-to/configure-rbac-in-your-kubernetes-cluster/)，可让你了解RBAC基础知识。

本指南面向希望对Helm限制如下权限的用户:
1. Tiller将资源安装到特定namespace能力
2. 授权Helm客户端对Tiller实例的访问

## Tiller和基于角色的访问控制


可以在配置Helm时使用`--service-account <NAME>`参数将服务帐户添加到Tiller 。前提条件是必须创建一个角色绑定，来指定预先设置的角色[role](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole)和服务帐户[service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) 名称。

在前提条件下，并且有了一个具有正确权限的服务帐户，就可以像这样运行一个命令来初始化Tiller： `helm init --service-account <NAME>`

### Example: 服务账户带有cluster-admin 角色权限

```console
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
apiVersion: rbac.authorization.k8s.io/v1beta1
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

_Note: cluster-admin角色是在Kubernetes集群中默认创建的，因此不必再显式地定义它。._

```console
$ kubectl create -f rbac-config.yaml
serviceaccount "tiller" created
clusterrolebinding "tiller" created
$ helm init --service-account tiller
```

### 在特定namespace中部署Tiller，并仅限于在该namespace中部署资源

在上面的例子中，我们让Tiller管理访问整个集群。当然，Tiller正常工作并不一定要为它设置集群管理员访问权限。我们可以指定Role和RoleBinding来将Tiller的范围限制为特定的namespace，而不是指定ClusterRole或ClusterRoleBinding。

```console
$ kubectl create namespace tiller-world
namespace "tiller-world" created
$ kubectl create serviceaccount tiller --namespace tiller-world
serviceaccount "tiller" created
```

定义允许Tiller管理namespace `tiller-world` 中所有资源的角色 ，文件`role-tiller.yaml`:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tiller-manager
  namespace: tiller-world
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
```

```console
$ kubectl create -f role-tiller.yaml
role "tiller-manager" created
```

文件 `rolebinding-tiller.yaml`,

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
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

```console
$ kubectl create -f rolebinding-tiller.yaml
rolebinding "tiller-binding" created
```

之后，运行`helm init`来在`tiller-world` namespace中安装Tiller 。

```console
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

### Example: 在一个namespace中部署Tiller，并限制它在另一个namespace部署资源

在上面的例子中，我们让Tiller管理它部署所在的namespace。现在，让我们限制Tiller的范围，将资源部署在不同的namespace中！

下面例子中，让我们在`myorg-system` namespace中安装Tiller，并允许Tiller在`myorg-users` namespace中部署资源。

```console
$ kubectl create namespace myorg-system
namespace "myorg-system" created
$ kubectl create serviceaccount tiller --namespace myorg-system
serviceaccount "tiller" created
```

在`role-tiller.yaml`中，定义了一个允许Tiller管理所有`myorg-users`资源的角色：

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tiller-manager
  namespace: myorg-users
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
```

```console
$ kubectl create -f role-tiller.yaml
role "tiller-manager" created
```

将 service account 与那个role绑定. `rolebinding-tiller.yaml`,

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
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

```console
$ kubectl create -f rolebinding-tiller.yaml
rolebinding "tiller-binding" created
```
我们还需要授予Tiller访问权限来读取`myorg-system`中的configmaps，以便它可以存储release信息。如 `role-tiller-myorg-system.yaml`:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: myorg-system
  name: tiller-manager
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["configmaps"]
  verbs: ["*"]
```

```console
$ kubectl create -f role-tiller-myorg-system.yaml
role "tiller-manager" created
```

相应的role 绑定. 如 `rolebinding-tiller-myorg-system.yaml`:

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
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

```console
$ kubectl create -f rolebinding-tiller-myorg-system.yaml
rolebinding "tiller-binding" created
```

## Helm 和基于角色的访问控制

在pod中运行Helm客户端时，为了让Helm客户端与Tiller实例进行通信，需要授予某些特权。具体来说，Helm客户端需要能够创建pods，转发端口并能够在Tiller运行的namespace中列出pod（这样它才可以找到Tiller）。

### Example: 在一个namespace中部署helm，与在另一个namespace中与Tiller交互

在这个例子中，我们将假设Tiller在名为`tiller-world` 的namespace中运行，并且Helm客户端在`helm-world`中运行。默认情况下，Tiller在`kube-system` namespace中运行。

如 `helm-user.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: helm
  namespace: helm-world
---
apiVersion: rbac.authorization.k8s.io/v1beta1
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
apiVersion: rbac.authorization.k8s.io/v1beta1
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

```console
$ kubectl create -f helm-user.yaml
serviceaccount "helm" created
role "tiller-user" created
rolebinding "tiller-user-binding" created
```
