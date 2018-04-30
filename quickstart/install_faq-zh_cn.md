# 安装FAQ

本节跟踪安装或开始使用Helm时遇到的一些经常遇到的问题。

**欢迎你的帮助** 来更好的提供此文档。要添加，更正或删除信息，提出问题[issue](https://github.com/kubernetes/helm/issues)或向我们发送PR请求。

## 下载

我想知道更多关于我的下载选项。

**问：我无法获得最新Helm的GitHub发布。他们在哪？**

答：我们不再使用GitHub发布版本。二进制文件现在存储在 GCS公共存储区中[GCS public bucket](https://kubernetes-helm.storage.googleapis.com)。

**问：为什么没有Debian/Fedora/... Helm的原生的软件包？**

我们很乐意提供这些信息，或者指向可靠的提供商。如果你对帮助感兴趣，我们很乐意。这就是Homebrew式的开始。

**问：你为什么要提供一个curl ...|bash脚本？**

答：我们的repo库（`scripts/get`）中有一个脚本可以作为`curl ..|bash`脚本执行。这些传输全部受HTTPS保护，并且脚本会对其获取的包进行一些审计。但是，脚本具有任何shell脚本的所有常见危险。

我们提供它是因为它很有用，但我们建议用户先仔细阅读脚本。并且，我们真正喜欢的是Helm的的打包版本。

## 安装

我正在尝试安装Helm/Tiller，但有些地方出了问题。

**问：我如何将Helm客户端文件放在~/.helm以外的地方？**

设置`$HELM_HOME`环境变量，然后运行`helm init`：

```bash
export HELM_HOME=/some/path
helm init --client-only
```

注意，如果你有现有的repo存储库，则需要通过`helm repo add...`.重新添加它们。

**问：我如何配置Helm，但不安装Tiller？**

答：默认情况下，helm init将确认本​​地$HELM_HOME配置，然后在群集上安装Tiller。要本地配置，但不安装Tiller，请使用`helm init --client-only`。

**问：如何在集群上手动安装Tiller？**

答：Tiller是作为Kubernetes deployment安装的。您可以通过运行`helm init --dry-run --debug`获取manifest，然后通过kubectl手动安装 。建议不要删除或更改该deployment中的标签labels，因为它们有时支持脚本和工具需要用到。

**问：为什么安装Tiller期间报错误Error response from daemon: target is unknown？**

答：有用户报告无法在使用Docker 1.13.0的Kubernetes实例上安装Tiller。造成这种情况的根本原因是Docker中的一个错误，它使得一个版本与早期版本的Docker推送到Docker注册表的镜像不兼容。

该问题在发布后不久就已修复，并在Docker 1.13.1-RC1和更高版本中提供。

## 入门
我成功安装了Helm/Tiller，但我使用时碰到问题。

**问：使用Helm时，收到错误“客户端传输中断”**

```
E1014 02:26:32.885226   16143 portforward.go:329] an error occurred forwarding 37008 -> 44134: error forwarding port 44134 to pod tiller-deploy-2117266891-e4lev_kube-system, uid : unable to do port forwarding: socat not found.
2016/10/14 02:26:32 transport: http2Client.notifyError got notified that the client transport was broken EOF.
Error: transport is closing
```

答：这通常表明Kubernetes未设置为允许端口转发。

通常情况下，缺少的部分是socat。如果您正在运行CoreOS，我们被告知它可能在安装时配置错误。CoreOS团队建议阅读以下内容：

https://coreos.com/kubernetes/docs/latest/kubelet-wrapper.html

以下是一些解决的问题案例，可以帮助您开始使用：

- https://github.com/kubernetes/helm/issues/1371
- https://github.com/kubernetes/helm/issues/966

**Q：使用Helm时,报错误"lookup XXXXX on 8.8.8.8:53: no such host"**

```
Error: Error forwarding ports: error upgrading connection: dial tcp: lookup kube-4gb-lon1-02 on 8.8.8.8:53: no such host
```

答：我们在Ubuntu和Kubeadm多节点群集中有这个问题。问题原因是节点期望某些DNS记录可以通过全局DNS获得。在上游解决此问题之前，可以按照以下方式解决该问题。在每个控制平面节点上：

1. 添加条目到`/etc/hosts`，将主机名映射到其 public IP
2. 安装`dnsmasq`（例如`apt install -y dnsmasq`）
3. 删除k8s api服务容器（kubelet会重新创建它）
4. 然后`systemctl restart docker`（或重新启动节点）请/etc/resolv.conf更改
请参阅此问题以获取更多信息：https://github.com/kubernetes/helm/issues/1455

**问：在GKE（Google Container Engine）上，报错"No SSH tunnels currently open"**

```
Error: Error forwarding ports: error upgrading connection: No SSH tunnels currently open. Were the targets able to accept an ssh-key for user "gke-[redacted]"?
```

错误消息的另一个形式是：

```
Unable to connect to the server: x509: certificate signed by unknown authority

```

答：这个问题是你的本地Kubernetes配置文件必须具有正确的凭据。

在GKE上创建集群时，它将提供凭证，包括SSL证书和证书颁发机构信息。这些需要存储在一个Kubernetes配置文件中（默认：`~/.kube/config`，这样`kubectl`和`helm`可以访问它们）。

**问：当我运行Helm命令时，出现有关隧道tunnel或代理proxy的错误**

答：Helm使用Kubernetes代理服务连接到Tiller服务器。如果命令`kubectl proxy`不适用，Helm也不行。通常，错误与缺失的`socat`服务有关。

**问：Tiller 崩溃**

当我在Helm上运行命令时，Tiller崩溃时会出现如下错误：

```
Tiller is listening on :44134
Probes server is listening on :44135
Storage driver is ConfigMap
Cannot initialize Kubernetes connection: the server has asked for the client to provide credentials 2016-12-20 15:18:40.545739 I | storage.go:37: Getting release "bailing-chinchilla" (v1) from storage
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x8053d5]

goroutine 77 [running]:
panic(0x1abbfc0, 0xc42000a040)
        /usr/local/go/src/runtime/panic.go:500 +0x1a1
k8s.io/helm/vendor/k8s.io/kubernetes/pkg/client/unversioned.(*ConfigMaps).Get(0xc4200c6200, 0xc420536100, 0x15, 0x1ca7431, 0x6, 0xc42016b6a0)
        /home/ubuntu/.go_workspace/src/k8s.io/helm/vendor/k8s.io/kubernetes/pkg/client/unversioned/configmap.go:58 +0x75
k8s.io/helm/pkg/storage/driver.(*ConfigMaps).Get(0xc4201d6190, 0xc420536100, 0x15, 0xc420536100, 0x15, 0xc4205360c0)
        /home/ubuntu/.go_workspace/src/k8s.io/helm/pkg/storage/driver/cfgmaps.go:69 +0x62
k8s.io/helm/pkg/storage.(*Storage).Get(0xc4201d61a0, 0xc4205360c0, 0x12, 0xc400000001, 0x12, 0x0, 0xc420200070)
        /home/ubuntu/.go_workspace/src/k8s.io/helm/pkg/storage/storage.go:38 +0x160
k8s.io/helm/pkg/tiller.(*ReleaseServer).uniqName(0xc42002a000, 0x0, 0x0, 0xc42016b800, 0xd66a13, 0xc42055a040, 0xc420558050, 0xc420122001)
        /home/ubuntu/.go_workspace/src/k8s.io/helm/pkg/tiller/release_server.go:577 +0xd7
k8s.io/helm/pkg/tiller.(*ReleaseServer).prepareRelease(0xc42002a000, 0xc42027c1e0, 0xc42002a001, 0xc42016bad0, 0xc42016ba08)
        /home/ubuntu/.go_workspace/src/k8s.io/helm/pkg/tiller/release_server.go:630 +0x71
k8s.io/helm/pkg/tiller.(*ReleaseServer).InstallRelease(0xc42002a000, 0x7f284c434068, 0xc420250c00, 0xc42027c1e0, 0x0, 0x31a9, 0x31a9)
        /home/ubuntu/.go_workspace/src/k8s.io/helm/pkg/tiller/release_server.go:604 +0x78
k8s.io/helm/pkg/proto/hapi/services._ReleaseService_InstallRelease_Handler(0x1c51f80, 0xc42002a000, 0x7f284c434068, 0xc420250c00, 0xc42027c190, 0x0, 0x0, 0x0, 0x0, 0x0)
        /home/ubuntu/.go_workspace/src/k8s.io/helm/pkg/proto/hapi/services/tiller.pb.go:747 +0x27d
k8s.io/helm/vendor/google.golang.org/grpc.(*Server).processUnaryRPC(0xc4202f3ea0, 0x28610a0, 0xc420078000, 0xc420264690, 0xc420166150, 0x288cbe8, 0xc420250bd0, 0x0, 0x0)
        /home/ubuntu/.go_workspace/src/k8s.io/helm/vendor/google.golang.org/grpc/server.go:608 +0xc50
k8s.io/helm/vendor/google.golang.org/grpc.(*Server).handleStream(0xc4202f3ea0, 0x28610a0, 0xc420078000, 0xc420264690, 0xc420250bd0)
        /home/ubuntu/.go_workspace/src/k8s.io/helm/vendor/google.golang.org/grpc/server.go:766 +0x6b0
k8s.io/helm/vendor/google.golang.org/grpc.(*Server).serveStreams.func1.1(0xc420124710, 0xc4202f3ea0, 0x28610a0, 0xc420078000, 0xc420264690)
        /home/ubuntu/.go_workspace/src/k8s.io/helm/vendor/google.golang.org/grpc/server.go:419 +0xab
created by k8s.io/helm/vendor/google.golang.org/grpc.(*Server).serveStreams.func1
        /home/ubuntu/.go_workspace/src/k8s.io/helm/vendor/google.golang.org/grpc/server.go:420 +0xa3
```

答：请检查Kubernetes的安全设置。

Tiller中的崩溃几乎总是由于未能与Kubernetes API服务器进行协商而导致的结果（此时，Tiller功能不正常，因此崩溃并退出）。

通常，这是认证失败的结果，因为运行Tiller的Pod没有正确的令牌token。

要解决这个问题，你需要修改Kubernetes配置。确保--service-account-private-key-file从controller-manager和 --service-account-key-file从API服务器指向同一个X509 RSA密钥。

## 升级

我的Helm原来工作正常，然后我升级了。现在它工作不正常。

**问：升级后，我收到错误“Client version is incompatible”。怎么问题？**

Tiller和Helm必须协商一个通用版本，以确保他们可以安全地进行通信而不会违反API假设。该错误意味着版本差异太大而无法安全地继续。通常，需要为此手动升级Tiller。

该安装指南[Installation Guide](install.md)大约有安全Helm升级和Tiller权威信息。

版本号的规则如下：

- 预发布版本与其他一切不兼容。Alpha.1与...不相容Alpha.2。
- 修补程序版本兼容：1.2.3与1.2.4兼容
- 少量修订不兼容：1.2.0与1.3.0不兼容，但我们可能在未来放宽这一限制。
- 主要版本不兼容：1.0.0与2.0.0不兼容。

## 卸载

我正在尝试删除某些东西。

**问：当我删除Tiller deployment时，为何所有安装的release信息还在集群里？**

安装release信息存储在kube-system名称空间内的ConfigMaps中。需要手动删除它们以删除记录或使用helm delete --purge。

问：我想删除我的本地Helm。它的所有文件在哪里？

包括helm二进制文件，Helm存储了一些文件在$HELM_HOME，默认位于~/.helm。
