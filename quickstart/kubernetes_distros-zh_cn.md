# Kubernetes各发行版本指南
本文档描述有关在各Kubernetes发行版本环境中使用Helm的信息。

我们尝试为此文档添加更多详细信息。如果可以，请通过Pull Requests提供。

## MiniKube
Helm已经过测试并且已知可以与minikube一起使用。它不需要额外的配置。

## scripts/local-cluster 和Hyperkube
通过配置Hyperkube scripts/local-cluster.sh已知可以工作。对于原始的Hyperkube，可能需要进行一些手动配置。

## GKE
已知Google的GKE托管Kubernetes平台与Helm一起工作，并且不需要额外的配置。

## Ubuntu与'kubeadm'
kubeadm构建的Kubernetes已知可用于以下Linux发行版：

- Ubuntu 16.04
- Fedora发布25

某些版本的Helm（v2.0.0-beta2）要求`export KUBECONFIG=/etc/kubernetes/admin.conf` 或创建一个`~/.kube/config`文件。

## CoreOS提供的Container Linux
Helm要求kubelet可以访问socat程序的副本，以代理与Tiller API的连接。在Container Linux上，Kubelet在具有socat 的[hyperkube](https://github.com/kubernetes/kubernetes/tree/master/cluster/images/hyperkube) 容器映像中运行。因此，尽管Container Linux没有socat,运行kubelet的容器​​文件系统具有socat。要了解更多信息，请阅读[Kubelet Wrapper](https://coreos.com/kubernetes/docs/latest/kubelet-wrapper.html) 文档。

## Openshift
Helm可在OpenShift Online，OpenShift Dedicated，OpenShift Container Platform（版本> = 3.6）或OpenShift Origin（版本> = 3.6）中直接使用。要了解更多，请阅读此[博客文章](https://blog.openshift.com/getting-started-helm-openshift/)。

## Platform9
Helm Client和Helm Server（Tiller）预装在[Platform9 Managed Kubernetes](https://platform9.com/managed-kubernetes/?utm_source=helm_distro_notes)。Platform9通过App目录UI和本地Kubernetes CLI提供对所有官方Helm charts的访问。其他repo存储库可以手动添加。有关更多详细信息，请参阅[Platform9 App Catalog文章](https://blog.openshift.com/getting-started-helm-openshift/)。

## DC / OS
Helm（客户端和服务器）已经过测试，在Mesospheres DC / OS 1.11 Kubernetes平台工作正常，无需其他配置。
