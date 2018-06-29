# 安装

Helm 有两个部分：Helm 客户端（helm）和 Helm 服务端（Tiller）。本指南介绍如何安装客户端，然后继续演示两种安装服务端的方法。

** 重要提示 **：如果你负责的群集是在受控的环境，尤其是在共享资源时，强烈建议使用安全配置安装 Tiller。有关指导，请参阅 [安全 Helm 安装](securing_installation-zh_cn.md)。

## 安装 Helm 客户端

Helm 客户端可以从源代码安装，也可以从预构建的二进制版本安装。

### 从二进制版本

每一个版本 [release](https://github.com/kubernetes/helm/releases)Helm 提供多种操作系统的二进制版本。这些二进制版本可以手动下载和安装。

1. 下载你 [想要的版本](https://github.com/kubernetes/helm/releases)
2. 解压缩（`tar -zxvf helm-v2.0.0-linux-amd64.tgz`）
3. `helm` 在解压后的目录中找到二进制文件，并将其移动到所需的位置（`mv linux-amd64/helm /usr/local/bin/helm`）

到这里，你应该可以运行客户端了：`helm help`。

### 通过 homebrew（macOS）

Kubernetes 社区的成员为 Homebrew 贡献了 Helm。这个通常是最新的。

```
brew install kubernetes-helm
```
（注意：emacs-helm 也是一个软件，这是一个不同的项目。）

### 从 Chocolatey（Windows）

Kubernetes 社区的成员为 Chocolatey 贡献了 Helm 包。这个软件包通常是最新的。

```
choco install kubernetes-helm
```

## 从脚本

Helm 现在有一个安装 shell 脚本，将自动获取最新版本的 Helm 客户端并在本地安装。

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

### 从金丝雀 (Canary) 构建

“Canary” 版本是从最新的主分支构建的 Helm 软件的版本。它们不是正式版本，可能不稳定。但是，他们提供了测试最新功能的机会。

"Canary" 版本 Helm 二进制文件存储在 Kubernetes Helm GCS 存储中。以下是常见构建的链接：

- [Linux AMD64](https://kubernetes-helm.storage.googleapis.com/helm-canary-linux-amd64.tar.gz)
- [macOS AMD64](https://kubernetes-helm.storage.googleapis.com/helm-canary-darwin-amd64.tar.gz)
- [Experimental Windows AMD64](https://kubernetes-helm.storage.googleapis.com/helm-canary-windows-amd64.zip)

### 源代码方式（Linux，macOS）

从源代码构建 Helm 的工作稍微多一些，但如果你想测试最新的（预发布）Helm 版本，那么这是最好的方法。

你必须有一个安装 Go 工作环境 。

```bash
$ cd $GOPATH
$ mkdir -p src/k8s.io
$ cd src/k8s.io
$ git clone https://github.com/kubernetes/helm.git
$ cd helm
$ make bootstrap build
```

该 `bootstrap` 目标将尝试安装依赖，重建 vendor / 树，并验证配置。

该 `build` 目标编译 `helm` 并将其放置在 `bin/helm` 目录。Tiller 也会编译，并且被放置在 `bin/tiller` 目录。

## 安装 Tiller

Helm 的服务器端部分 Tiller 通常运行在 Kubernetes 集群内部。但是对于开发，它也可以在本地运行，并配置为与远程 Kubernetes 群集通信。

### 快捷群集内安装

安装 `tiller` 到群集中最简单的方法就是运行 `helm init`。这将验证 `helm` 本地环境设置是否正确（并在必要时进行设置）。然后它会连接到 `kubectl` 默认连接的任何集群（`kubectl config view`）。一旦连接，它将安装 `tiller` 到 `kube-system` 命名空间中。

`helm init` 以后，可以运行 `kubectl get pods --namespace kube-system` 并看到 Tiller 正在运行。

你可以通过参数运行 `helm init`:

- `--canary-image` 参数安装金丝雀版本
- `--tiller-image` 安装特定的镜像（版本）
- `--kube-context` 使用安装到特定群集
- `--tiller-namespace` 用一个特定的命名空间 (namespace) 安装

一旦安装了 Tiller，运行 helm version 会显示客户端和服务器版本。（如果它仅显示客户端版本， helm 则无法连接到服务器, 使用 `kubectl` 查看是否有任何 tiller Pod 正在运行。）

除非设置 `--tiller-namespace` 或 `TILLER_NAMESPACE` 参数，否则 Helm 将在命名空间 `kube-system` 中查找 Tiller 。

### 安装 Tiller 金丝雀版本

Canary 镜像是从 master 分支建立的。他们可能不稳定，但他们提供测试最新功能的机会。

安装 Canary 镜像最简单的方法是 helm init 与 --canary-image 参数一起使用：

```bash
$ helm init --canary-image
```

这将使用最近构建的容器镜像。可以随时使用 ` kubectl` 删除 `kube-system` 名称空间中的 Tiller deployment 来卸载 Tiller。

### 本地运行 Tiller

对于开发而言，有时在本地运行 Tiller 更容易，将其配置为连接到远程 Kubernetes 群集。

上面介绍了构建部署 Tiller 的过程。

一旦 tiller 构建部署完成，只需启动它：

```bash
$ bin/tiller
Tiller running on :44134
```

当 Tiller 在本地运行时，它将尝试连接到由 `kubectl` 配置的 Kubernetes 群集。（运行 kubectl config view 以查看是哪个群集。）

必须告知 helm 连接到这个新的本地 Tiller 主机，而不是连接到群集中的一个。有两种方法可以做到这一点。第一种是在命令行上指定 `--host` 选项。第二个是设置 `$HELM_HOST` 环境变量。

```bash
$ export HELM_HOST=localhost:44134
$ helm version # Should connect to localhost.
Client: &version.Version{SemVer:"v2.0.0-alpha.4", GitCommit:"db...", GitTreeState:"dirty"}
Server: &version.Version{SemVer:"v2.0.0-alpha.4", GitCommit:"a5...", GitTreeState:"dirty"}
```

注意，即使在本地运行，Tiller 也会将安装的 release 配置存储在 Kubernetes 内的 ConfigMaps 中。

## 升级 Tiller

从 Helm 2.2.0 开始，Tiller 可以升级使用 `helm init --upgrade`。

对于旧版本的 Helm 或手动升级，可以使用 `kubectl` 修改 Tiller 容器镜像：

```bash
$ export TILLER_TAG=v2.0.0-beta.1        # Or whatever version you want
$ kubectl --namespace=kube-system set image deployments/tiller-deploy tiller=gcr.io/kubernetes-helm/tiller:$TILLER_TAG
deployment "tiller-deploy" image updated
```

设置 `TILLER_TAG=canary` 将获得 master 版本的最新快照。

## 删除或重新安装 Tiller

由于 Tiller 将其数据存储在 Kubernetes ConfigMaps 中，因此可以安全地删除并重新安装 Tiller，而无需担心丢失任何数据。推荐删除 Tiller 的方法是使用 `kubectl delete deployment tiller-deploy --namespace kube-system` 或更简洁使用 `helm reset`。

然后可以从客户端重新安装 Tiller：

```bash
$ helm init
```

## 高级用法

helm init 提供了额外的参数，用于在安装之前修改 Tiller 的 deployment manifest。

### 使用 `--node-selectors`

`--node-selectors` 参数允许我们指定调度 Tiller Pod 所需的节点标签。

下面的例子将在 nodeSelector 属性下创建指定的标签。

```
helm init --node-selectors "beta.kubernetes.io/os"="linux"
```

已安装的 deployment manifest 将包含我们的节点选择器标签。

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

`--override` 允许指定 Tiller 的 deployment manifest 的属性。与在 Helm 其他地方 `--set` 使用的命令不同，`helm init --override` 修改最终 manifest 的指定属性（没有 "values" 文件）。因此，可以为 deployment manifest 中的任何有效属性指定任何有效值。

#### 覆盖注释

在下面的示例中，我们使用 --override 添加修订版本属性并将其值设置为 1。

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
在下面的例子中，我们为节点设置了亲和性属性。`--override` 可以组合来修改同一列表项的不同属性。

```
helm init --override "spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[0].weight"="1" --override "spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[0].preference.matchExpressions[0].key"="e2e-az-name"
```

指定的属性组合到 “preferredDuringSchedulingIgnoredDuringExecution” 属性的第一个列表项中。

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

`--output` 参数允许我们跳过安装 Tiller 的 deployment
manifest，并以 JSON 或 YAML 格式简单地将 deployment
manifest 输出到标准输出 stdout。然后可以使用 `jq` 类似工具修改输出，并使用 `kubectl` 手动安装。

在下面的例子中，我们 helm init 用 --output json 参数执行。

```
helm init --output json
```

Tiller 安装被跳过，manifest 以 JSON 格式输出到 stdout。

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

默认情况下，tiller 将安装 release 信息存储在其运行的名称空间中的 ConfigMaps 中。从 Helm 2.7.0 开始，现在有一个 Secrets 用于存储安装 release 信息的 beta 存储后端。添加了这个功能是为和 Kubernetes 的加密 Secret 一起，保护 chart 的安全性。

要启用 secrets 后端，需要使用以下选项启动 Tiller：

```bash
helm init --override 'spec.template.spec.containers[0].command'='{/tiller,--storage=secret}'
```

目前，如果想从默认后端切换到 secrets 后端，必须自行为此进行迁移配置信息。当这个后端从 beta 版本毕业时，将会有更正式的移徙方法。

## 总结

在大多数情况下，安装和获取预先构建的 helm 二进制代码和 `helm init` 一样简单。这个文档提供而了一些用例给那些想要用 Helm 做更复杂的事情的人。

一旦成功安装了Helm Client和Tiller，可以继续下一步使用Helm来管理charts。
