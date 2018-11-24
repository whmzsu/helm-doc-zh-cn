# 快速入门
本指南介绍如何快速开始使用 Helm。

## 前提条件
需要准备以下前提条件才能成功且安全地使用 Helm。

1. 一个 Kubernetes 集群
2. 确定使用哪种安装安全配置（如果有的话）
3. 安装和配置 Helm 和集群端服务 Tiller。

### 安装 Kubernetes 或有权访问群集
- 必须已安装 Kubernetes。对于 Helm 的最新版本，我们推荐最新的 Kubernetes 稳定版本，在大多数情况下它是次新版本。
- 应该有一个本地配置好的 `kubectl`。

** 注意：** 1.6 之前的 Kubernetes 版本对于基于角色的访问控制（RBAC），要么有限制，或者不支持。

Helm 将通过 Kubernetes 配置文件（通常是 `$HOME/.kube/config`）来确定在哪里安装 Tiller 。这个配置文件也是 kubectl 使用的文件。

要找出 Tiller 将安装到哪个集群，可以运行 `kubectl config current-context` 或 `kubectl cluster-info`。

```bash
$ kubectl config current-context
my-cluster
```
### 了解集群配置的安全上下文
与所有强大的工具一样，需要确保为你的场景正确安装它。

如果你在完全控制的群集上使用 Helm，如 minikube 或专用网络中的不考虑共享的群集，则默认安装（不采用安全配置）很合适，并且是最容易的。要在无需额外安全措施的场景下安装 Helm，请参考 [安装 Helm](# 安装 Helm)，然后 [初始化 Helm](# 初始化 Helm 并安装 Tiller)。

但是，如果集群暴露于更大的网络中，或者集群与他人共享 - 生产集群属于此类别 - 则必须采取额外步骤来确保安装安全，以防止不小心或恶意的操作者损坏集群或其集群数据。在生产环境和其他多租户方案中，要使用安全配置安装 Helm，请参阅 [Helm 安全安装](securing_installation-zh_cn.md)。

如果群集启用了基于角色的访问控制（RBAC），在继续之前配置 [服务帐户 (service account) 和规则](rbac-zh_cn.md)。

## 安装 Helm
下载 Helm 客户端的二进制版本。可以使用类似工具如 `homebrew`，或查看 [官方发布页面](https://github.com/helm/helm/releases)。

有关更多详细信息或其他选项，请参阅 [安装指南](install-zh_cn.md)。

## 初始化 Helm 并安装 Tiller
有了 Helm 安装文件，就可以初始化本地 CLI，并将 Tiller 安装到 Kubernetes 集群中：

```bash
$ helm init
```
这会将 Tiller 安装到对应的 Kubernetes 群集中, 集群同 `kubectl config current-context`。

** 提示：** 想要安装到不同的群集中？使用 --kube-context 参数。

** 提示：** 如果要升级 Tiller，请运行 helm init --upgrade。

默认情况下，安装 Tiller 时，没有启用身份验证。要了解有关为 Tiller 配置增强 TLS 身份验证的更多信息，请参阅 [Tiller TLS 指南](tiller_ssl-zh_cn.md)。

## 安装示例 Chart
要安装一个 chart，可以运行 `helm install` 命令。Helm 有几种方法来查找和安装 chart，但最简单的方法是使用其中一个官方 `stable` 稳定版本的 chart。

```bash
$ helm repo update               ＃确保我们获得最新的 chart 清单
$ helm install stable/mysql
Released smile-penguin
NAME:   wintering-rodent
LAST DEPLOYED: Thu Oct 18 14:21:18 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME                    AGE
wintering-rodent-mysql  0s

==> v1/ConfigMap
wintering-rodent-mysql-test  0s

==> v1/PersistentVolumeClaim
wintering-rodent-mysql  0s

==> v1/Service
wintering-rodent-mysql  0s

==> v1beta1/Deployment
wintering-rodent-mysql  0s

==> v1/Pod(related)

NAME                                    READY  STATUS   RESTARTS  AGE
wintering-rodent-mysql-6986fd6fb-988x7  0/1    Pending  0         0s


NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
wintering-rodent-mysql.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default wintering-rodent-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h wintering-rodent-mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/wintering-rodent-mysql 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}


```
在上面的例子中，stable/mysql 已经安装，安装版本的 release 的名字是 `wintering-rodent`。通过运行 `helm inspect stable/mysql` 可以简单了解这个 MySQL chart 的功能。

无论何时安装 chart，都会创建一个新 release 版本。所以一个 chart 可以多次安装到同一个群集中。而且每个都可以独立管理和升级。

`helm install` 命令功能非常丰富，具有很多强大功能。要了解更多信息，请查看 [使用 Helm 指南](using_helm-zh_cn.md)

## 了解安装的 release

很容易通过如下命令查看已使用 Helm 安装的 release：

```bash
$ helm ls
+NAME            	REVISION	UPDATED                 	STATUS  	CHART       	APP VERSION	NAMESPACE
+wintering-rodent	1       	Thu Oct 18 15:06:58 2018	DEPLOYED	mysql-0.10.1	5.7.14     	default
```

## 卸载安装的 release

要卸载安装的 release，请使用以下 `helm delete` 命令：

```bash
$ helm delete wintering-rodent
release "wintering-rodent" deleted
```

`wintering-rodent` release 将从 Kubernetes 卸载，但仍然可以查询有关该 release 的信息：

```bash
$ helm status wintering-rodent
LAST DEPLOYED: Thu Oct 18 14:21:18 2018
NAMESPACE: default
STATUS: DELETED

NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
wintering-rodent-mysql.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default wintering-rodent-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h wintering-rodent-mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/wintering-rodent-mysql 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}

```

由于 Helm 在删除它们之后也会跟踪该 release，因此可以审核群集的历史记录，甚至可以取消删除动作（使用 `helm rollback`）。

## 阅读帮助文本

要了解有关 Helm 命令的更多信息，请使用 `helm help` 或键入一个后跟 `-h` 标志的命令：

```bash
$ helm get -h
```
