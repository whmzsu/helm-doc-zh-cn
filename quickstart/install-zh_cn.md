# 安装

Helm有两个部分：Helm客户端（helm）和Helm服务器（Tiller）。本指南介绍如何安装客户端，然后继续演示两种安装服务端的方法。

**重要提示**：如果你负责的群集是在受控的环境，尤其是在共享资源时，强烈建议使用安全配置安装Tiller。有关指导，请参阅[安全Helm安装](securing_installation-zh_cn.md)。

## 安装Helm客户端

Helm客户端可以从源代码安装，也可以从预构建的二进制版本安装。

### 二进制版本

每一个版本[release](https://github.com/kubernetes/helm/releases)Helm提供多种操作系统的二进制版本。这些二进制版本可以手动下载和安装。

1. 下载你[想要的版本](https://github.com/kubernetes/helm/releases)
2. 解压缩（`tar -zxvf helm-v2.0.0-linux-amd64.tgz`）
3. `helm`在解压后的目录中找到二进制文件，并将其移动到所需的位置（`mv linux-amd64/helm /usr/local/bin/helm`）

到这里，你应该可以运行客户端了：`helm help`。

### 通过homebrew（macOS）

Kubernetes社区的成员为Homebrew贡献了Helm。这个通常是最新的。

```
brew install kubernetes-helm
```
（注意：emacs-helm也是一个软件，这是一个不同的项目。）

### 从Chocolatey（Windows）

Kubernetes社区的成员为 Chocolatey贡献了Helm包。这个软件包通常是最新的。

```
choco install kubernetes-helm
```

## 从脚本

Helm现在有一个安装shell脚本，将自动获取最新版本的Helm客户端并在本地安装。

可以获取该脚本，然后在本地执行它。这种方法也有文档指导，以便可以在运行之前仔细阅读并理解它在做什么。

```
$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

```
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
```
也可以做到这一点。

### 从金丝雀(Canary )构建

“Canary”版本是从最新的主分支构建的Helm软件的版本。它们不是正式版本，可能不稳定。但是，他们提供了测试最新功能的机会。

"Canary"版本Helm二进制文件存储在Kubernetes Helm GCS存储中。以下是常见构建的链接：

- [Linux AMD64](https://kubernetes-helm.storage.googleapis.com/helm-canary-linux-amd64.tar.gz)
- [macOS AMD64](https://kubernetes-helm.storage.googleapis.com/helm-canary-darwin-amd64.tar.gz)
- [Experimental Windows AMD64](https://kubernetes-helm.storage.googleapis.com/helm-canary-windows-amd64.zip)

### 源代码方式（Linux，macOS）

从源代码构建Helm的工作稍微多一些，但如果你想测试最新的（预发布）Helm版本，那么这是最好的方法。

你必须有一个安装Go工作环境 。

```console
$ cd $GOPATH
$ mkdir -p src/k8s.io
$ cd src/k8s.io
$ git clone https://github.com/kubernetes/helm.git
$ cd helm
$ make bootstrap build
```

该`bootstrap`目标将尝试安装依赖，重建 vendor/树，并验证配置。

该`build`目标编译`helm`并将其放置在`bin/helm`目录。Tiller也会编译，并且被放置在`bin/tiller`目录。

## 安装Tiller

Helm的服务器端部分Tiller通常运行在Kubernetes集群内部。但是对于开发，它也可以在本地运行，并配置为与远程Kubernetes群集通信。

### 快捷群集内安装

安装`tiller`到群集中最简单的方法就是运行 `helm init`。这将验证`helm`本地环境设置是否正确（并在必要时进行设置）。然后它会连接到`kubectl`默认连接的任何集群（`kubectl config view`）。一旦连接，它将安装`tiller`到 `kube-system`命名空间中。

`helm init`以后，可以运行`kubectl get pods --namespace kube-system`并看到Tiller正在运行。

你可以通过参数运行`helm init`:

- `--canary-image` 参数安装金丝雀版本
- `--tiller-image` 安装特定的镜像（版本）
- `--kube-context` 使用安装到特定群集
- `--tiller-namespace` 用一个特定的命名空间(namespace)安装

一旦安装了Tiller，运行helm version会显示客户端和服务器版本。（如果它仅显示客户端版本， helm则无法连接到服务器,使用`kubectl`查看是否有任何 tiller Pod正在运行。）

除非设置`--tiller-namespace`或`TILLER_NAMESPACE`参数，否则Helm将在命名空间`kube-system`中查找Tiller 。

### 安装Tiller金丝雀版本

Canary 镜像是从master分支建立的。他们可能不稳定，但他们为您提供测试最新功能的机会。

安装Canary 镜像最简单的方法是helm init与 --canary-image参数一起使用：

```console
$ helm init --canary-image
```

这将使用最近构建的容器镜像。您可以随时使用` kubectl`删除`kube-system`名称空间中的Tiller deployment来卸载Tiller。

### 本地运行Tiller

对于开发而言，有时在本地运行Tiller更容易，将其配置为连接到远程Kubernetes群集。

上面介绍了构建部署Tiller的过程。

一旦tiller构建部署完成，只需启动它：

```console
$ bin/tiller
Tiller running on :44134
```

当Tiller在本地运行时，它将尝试连接到由`kubectl`配置的Kubernetes群集。（运行kubectl config view以查看是哪个群集。）

必须告知helm连接到这个新的本地Tiller主机，而不是连接到群集中的一个。有两种方法可以做到这一点。第一种是在命令行上指定`--host`选项。第二个是设置`$HELM_HOST`环境变量。

```console
$ export HELM_HOST=localhost:44134
$ helm version # Should connect to localhost.
Client: &version.Version{SemVer:"v2.0.0-alpha.4", GitCommit:"db...", GitTreeState:"dirty"}
Server: &version.Version{SemVer:"v2.0.0-alpha.4", GitCommit:"a5...", GitTreeState:"dirty"}
```

注意，即使在本地运行，Tiller也会将安装的release配置存储在Kubernetes内的ConfigMaps中。

## 升级Tiller

从Helm 2.2.0开始，Tiller可以升级使用`helm init --upgrade`。

对于旧版本的Helm或手动升级，可以使用`kubectl`修改Tiller容器镜像：

```console
$ export TILLER_TAG=v2.0.0-beta.1        # Or whatever version you want
$ kubectl --namespace=kube-system set image deployments/tiller-deploy tiller=gcr.io/kubernetes-helm/tiller:$TILLER_TAG
deployment "tiller-deploy" image updated
```

设置`TILLER_TAG=canary`将获得master版本的最新快照。

## 删除或重新安装Tiller

由于Tiller将其数据存储在Kubernetes ConfigMaps中，因此可以安全地删除并重新安装Tiller，而无需担心丢失任何数据。推荐删除Tiller的方法是使用`kubectl delete deployment tiller-deploy --namespace kube-system`或更简洁使用`helm reset`。

然后可以从客户端重新安装Tiller：

```console
$ helm init
```

## 高级用法

helm init 提供了额外的参数，用于在安装之前修改Tiller的deployment manifest。

### 使用 `--node-selectors`

`--node-selectors`参数允许我们指定调度Tiller Pod所需的节点标签。

下面的例子将在nodeSelector属性下创建指定的标签。

```
helm init --node-selectors "beta.kubernetes.io/os"="linux"
```

已安装的deployment manifest将包含我们的节点选择器标签。

```
...
spec:
  template:
    spec:
      nodeSelector:
        beta.kubernetes.io/os: linux
...
```

### 使用 --override

`--override`允许指定Tiller的deployment manifest的属性。与在Helm其他地方`--set`使用的命令不同，`helm init --override`修改最终manifest的指定属性（没有"values"文件）。因此，可以为deployment manifest中的任何有效属性指定任何有效值。

#### 覆盖注释

在下面的示例中，我们使用--override添加修订版本属性并将其值设置为1。

```
helm init --override metadata.annotations."deployment\.kubernetes\.io/revision"="1"
```

输出：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
...
```

#### 覆盖亲和性
在下面的例子中，我们为节点设置了亲和性属性。`--override`可以组合来修改同一列表项的不同属性。

```
helm init --override "spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[0].weight"="1" --override "spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[0].preference.matchExpressions[0].key"="e2e-az-name"
```

指定的属性组合到“preferredDuringSchedulingIgnoredDuringExecution”属性的第一个列表项中。

```
...
spec:
  strategy: {}
  template:
    ...
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - preference:
              matchExpressions:
              - key: e2e-az-name
                operator: ""
            weight: 1
...
```

### 使用 --output

`--output`参数允许我们跳过安装Tiller的deployment
manifest，并以JSON或YAML格式简单地将deployment
manifest输出到标准输出stdout。然后可以使用`jq`类似工具修改输出，并使用`kubectl`手动安装。

在下面的例子中，我们helm init用--output json 参数执行。

```
helm init --output json
```

Tiller安装被跳过，manifest以JSON格式输出到stdout。

```json
"apiVersion": "extensions/v1beta1",
"kind": "Deployment",
"metadata": {
    "creationTimestamp": null,
    "labels": {
        "app": "helm",
        "name": "tiller"
    },
    "name": "tiller-deploy",
    "namespace": "kube-system"
},
...
```
### 存储后端

默认情况下，tiller将安装release信息存储在其运行的名称空间中的ConfigMaps中。从Helm 2.7.0开始，现在有一个Secrets用于存储安装release信息的beta存储后端。添加了这个功能是为和Kubernetes的加密Secret一起，保护chart的安全性。

要启用secrets后端，需要使用以下选项启动Tiller：

```shell
helm init --override 'spec.template.spec.containers[0].command'='{/tiller,--storage=secret}'
```

目前，如果您想从默认后端切换到secrets后端，必须自行为此进行迁移配置信息。当这个后端从beta版本毕业时，将会有更正式的移徙方法。

## 总结

在大多数情况下，安装和获取预先构建的helm二进制代码及`helm init`一样简单。这个文档提供而了一些用例给那些想要用Helm做更复杂的事情的人。

一旦成功安装了Helm Client和Tiller，可以继续下一步使用Helm来管理charts。
