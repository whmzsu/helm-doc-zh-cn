# Pod 和 Pod 模板

最佳实践指南的这一部分讨论了如何格式化 chart 清单中的 Pod 和 PodTemplate 部分。

以下（非详尽）资源列表使用 PodTemplates：

- Deployment
- ReplicationController
- ReplicaSet
- DaemonSet
- StatefulSet

## 镜像

容器镜像应该使用固定标签或镜像的 SHA。它不应该使用的标签 `latest`，`head`，`canary`，或其他设计为 “浮动” 的标签。

镜像可以在 `values.yaml` 文件中定义，可以很容易地换为镜像地址。

```
image: {{.Values.redisImage | quote}}
```

镜像和标签可以在 `values.yaml` 中定义为两个单独的字段：

```
image: "{{.Values.redisImage}}:{{ .Values.redisTag }}"
```

## ImagePullPolicy

`helm create` 设置 `imagePullPolicy` 为 `IfNotPresent`, 在 `deployment.yaml` 中：

```yaml
imagePullPolicy: {{.Values.image.pullPolicy}}
```

和 values.yaml 中：

```yaml
pullPolicy: IfNotPresent
```

同样，Kubernetes 默认 `imagePullPolicy` 为 `IfNotPresent`，如果它根本没有被定义。如果想要的值不是 IfNotPresent，只需将 `values.yaml` 中的值更新为所需的值即可。

## PodTemplates 应声明选择器

所有的 PodTemplate 部分都应该指定一个选择器。例如：

```yaml
selector:
  matchLabels:
      app.kubernetes.io/name: MyName
template:
  metadata:
    labels:
      app.kubernetes.io/name: MyName
```

这是一个很好的做法，因为它可以使 set 和 pod 之间保持关系。

但对于像Deployment这样的集合来说，这更为重要。如果没有这一点，整套标签将用于选择匹配的pod，如果使用的标签（如版本或发布日期）变化了，则将会导致app中断。
