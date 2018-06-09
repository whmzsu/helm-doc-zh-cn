# 快速入门
本指南介绍如何快速开始使用Helm。

## 前提条件
需要准备以下前提条件才能成功且安全地使用Helm。

1. 一个Kubernetes集群
2. 确定使用哪种安装安全配置（如果有的话）
3. 安装和配置Helm和集群端服务Tiller。

### 安装Kubernetes或有权访问群集
- 必须已安装Kubernetes。对于Helm的最新版本，我们推荐最新的Kubernetes稳定版本，在大多数情况下它是次新版本。
- 应该有一个本地配置好的`kubectl`。

**注意：**1.6之前的Kubernetes版本对于基于角色的访问控制（RBAC），要么有限制，或者不支持。

Helm将通过Kubernetes配置文件（通常是`$HOME/.kube/config`）来确定在哪里安装Tiller 。这个配置文件也是kubectl使用的文件。

要找出Tiller将安装到哪个集群，可以运行 `kubectl config current-context`或`kubectl cluster-info`。

```bash
$ kubectl config current-context
my-cluster
```
### 了解集群配置的安全上下文
与所有强大的工具一样，需要确保为你的场景正确安装它。

如果你在完全控制的群集上使用Helm，如minikube或专用网络中的不考虑共享的群集，则默认安装（不采用安全配置）很合适，并且是最容易的。要在无需额外安全措施的场景下安装Helm，请参考[安装Helm](#安装Helm)，然后[初始化Helm](#初始化Helm并安装Tiller)。

但是，如果集群暴露于更大的网络中，或者集群与他人共享 - 生产集群属于此类别 - 则必须采取额外步骤来确保安装安全，以防止不小心或恶意的操作者损坏集群或其集群数据。在生产环境和其他多租户方案中，要使用安全配置安装Helm，请参阅[Helm安全安装](securing_installation-zh_cn.md)。

如果群集启用了基于角色的访问控制（RBAC），在继续之前配置[服务帐户(service account)和规则](rbac-zh_cn.md)。

## 安装Helm
下载Helm客户端的二进制版本。可以使用类似工具如`homebrew`，或查看[官方发布页面](https://github.com/kubernetes/helm/releases)。

有关更多详细信息或其他选项，请参阅[安装指南](install-zh_cn.md)。

## 初始化Helm并安装Tiller
有了Helm安装文件，就可以初始化本地CLI，并将Tiller安装到Kubernetes集群中：

```bash
$ helm init
```
这会将Tiller安装到对应的Kubernetes群集中,集群同`kubectl config current-context`。

**提示：** 想要安装到不同的群集中？使用 --kube-context 参数。

**提示：** 如果要升级Tiller，请运行helm init --upgrade。

默认情况下，安装Tiller时，没有启用身份验证。要了解有关为Tiller配置增强TLS身份验证的更多信息，请参阅 [Tiller TLS指南](tiller_ssl-zh_cn.md)。

## 安装示例Chart
要安装一个chart，可以运行`helm install`命令。Helm有几种方法来查找和安装chart，但最简单的方法是使用其中一个官方`stable`稳定版本的chart。

```bash
$ helm repo update               ＃确保我们获得最新的chart列表
$ helm install stable / mysql
Released smile-penguin
```
在上面的例子中，stable/mysql 已经安装，安装版本的release的名字是smiling-penguin。通过运行`helm inspect stable/mysql`可以简单了解该MySQL chart的功能。

无论何时安装chart，都会创建一个新release版本。所以一个chart可以多次安装到同一个群集中。而且每个都可以独立管理和升级。

`helm install`命令功能非常丰富，具有很多强大功能。要了解更多信息，请查看[使用Helm指南](using_helm-zh_cn.md)

## 了解安装的release
很容易通过如下命令查看已使用Helm安装的内容：

```bash
$ helm ls
NAME             VERSION   UPDATED                   STATUS    CHART
smiling-penguin  1         Wed Sep 28 12:59:46 2016  DEPLOYED  mysql-0.1.0
```

## 卸载安装的release
要卸载安装的release，请使用以下`helm delete`命令：

```bash
$ helm delete smiling-penguin
Removed smiling-penguin
```

smiling-penguin release将从Kubernetes 卸载，但仍然可以查询有关该release的信息：

```bash
$ helm status smiling-penguin
Status: DELETED
...
```

由于Helm在删除它们之后也会跟踪release，因此可以审核群集的历史记录，甚至可以取消删除动作（使用`helm rollback`）。

## 阅读帮助文本
要了解有关Helm命令的更多信息，请使用helm help或键入一个后跟该-h标志的命令：

```bash
$ helm get -h
```
