# Kubernetes 各发行版本指南
本文档描述有关在各 Kubernetes 发行版本环境中使用 Helm 的信息。

我们尝试为此文档添加更多详细信息。如果可以，请通过 Pull Requests 提供。

## MiniKube
Helm 已经过测试并且已知可以与 minikube 一起使用。它不需要额外的配置。

## scripts/local-cluster 和 Hyperkube
通过配置 Hyperkube scripts/local-cluster.sh 已知可以工作。对于原始的 Hyperkube，可能需要进行一些手动配置。

## GKE
已知 Google 的 GKE 托管 Kubernetes 平台与 Helm 一起工作，并且不需要额外的配置。

## Ubuntu 与'kubeadm'
kubeadm 构建的 Kubernetes 已知可用于以下 Linux 发行版：

- Ubuntu 16.04
- Fedora 发布 25

某些版本的 Helm（v2.0.0-beta2）要求 `export KUBECONFIG=/etc/kubernetes/admin.conf` 或创建一个 `~/.kube/config` 文件。

## CoreOS 提供的 Container Linux
Helm 要求 kubelet 可以访问 socat 程序的副本，以代理与 Tiller API 的连接。在 Container Linux 上，Kubelet 在具有 socat 的 [hyperkube](https://github.com/kubernetes/kubernetes/tree/master/cluster/images/hyperkube) 容器映像中运行。因此，尽管 Container Linux 没有 socat, 运行 kubelet 的容器​​文件系统具有 socat。要了解更多信息，请阅读 [Kubelet Wrapper](https://coreos.com/kubernetes/docs/latest/kubelet-wrapper.html) 文档。

## Openshift
Helm 可在 OpenShift Online，OpenShift Dedicated，OpenShift Container Platform（版本 > = 3.6）或 OpenShift Origin（版本 > = 3.6）中直接使用。要了解更多，请阅读此 [博客文章](https://blog.openshift.com/getting-started-helm-openshift/)。

## Platform9
Helm Client 和 Helm Server（Tiller）预装在 [Platform9 Managed Kubernetes](https://platform9.com/managed-kubernetes/?utm_source=helm_distro_notes)。Platform9 通过 App 目录 UI 和本地 Kubernetes CLI 提供对所有官方 Helm charts 的访问。其他 repo 存储库可以手动添加。有关更多详细信息，请参阅 [Platform9 App Catalog 文章](https://blog.openshift.com/getting-started-helm-openshift/)。

## DC / OS
Helm（客户端和服务器）已经过测试，在Mesospheres DC / OS 1.11 Kubernetes平台工作正常，无需其他配置。
