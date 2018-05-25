# 标签和注释
最佳实践指南的这一部分讨论了在chart中使用标签和注释的最佳做法。

## 它是一个标签还是一个注释？
在下列条件下，元数据项应该是标签：

- Kubernetes使用它来识别此资源
- 为了查询系统目的，向操作员暴露是非常有用的。

例如，我们建议使用`chart: NAME-VERSION` 标签作为标签，以便操作员可以方便地查找要使用的特定chart的所有实例。

如果元数据项不用于查询，则应将其设置为注释。

Helm hook总是注释。

## 标准标签
下表定义了Helm chart使用的通用标签。Helm本身从不要求特定的标签。标记为REC的标签是表示推荐的，应放置在chart上以保持全局一致性。那些标记OPT是表示可选的。这些都是惯用的或通常使用的，但不是经常用于运维目的。

名称|状态|描述
-----|------|----------
heritage|	REC	|这应该始终设置为`{{ .Release.Service }}`。它用于查找Tiller管理的所有东西。
release| REC |这应该是`{{ .Release.Name }}`。
chart| REC| 这应该是chart名称和版本：`{{ .Chart.Name }}`,`{{ .Chart.Version }}`。
app| REC| 这应该是应用程序名称，代表了整个应用程序。通常`{{ template "name" . }}` 用于此。这被许多Kubernetes manifests所使用，而不是Helm特有的。
component| OPT|	这是标记片段,在应用程序中可能发挥的不同角色的常用标签。例如，`component: frontend`。
