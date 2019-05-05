# 标签和注释
最佳实践指南的这一部分讨论了在 chart 中使用标签和注释的最佳做法。

## 它是一个标签还是一个注释？
在下列条件下，元数据项应该是标签：

- Kubernetes 使用它来识别此资源
- 为了查询系统目的，向操作员暴露是非常有用的。

例如，我们建议使用 `helm.sh/chart: NAME-VERSION` 标签作为标签，以便操作员可以方便地查找要使用的特定 chart 的所有实例。

如果元数据项不用于查询，则应将其设置为注释。

Helm hook 总是注释。

## 标准标签
下表定义了 Helm chart 使用的通用标签。Helm 本身从不要求特定的标签。标记为 REC 的标签是表示推荐的，应放置在 chart 上以保持全局一致性。那些标记 OPT 是表示可选的。这些都是惯用的或通常使用的，但不是经常用于运维目的。

名称 | 状态 | 描述
-----|------|----------
`app.kubernetes.io/name` | REC | This should be the app name, reflecting the entire app. Usually {{ template "name" . }} is used for this. This is used by many Kubernetes manifests, and is not Helm-specific.
`helm.sh/chart` | REC | This should be the chart name and version: `{{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}`
`app.kubernetes.io/managed-by` | REC | This should always be set to `{{ .Release.Service }}`. It is for finding all things managed by Tiller.
`app.kubernetes.io/instance` | REC | This should be the `{{ .Release.Name }}`. It aid in differentiating between different instances of the same application.
`app.kubernetes.io/version` | OPT | The version of the app and can be set to `{{ .Chart.AppVersion }}`.
`app.kubernetes.io/component` | OPT | This is a common label for marking the different roles that pieces may play in an application. For example, `app.kubernetes.io/component: frontend`.
`app.kubernetes.io/part-of` | OPT | When multiple charts or pieces of software are used together to make one application. For example, application software and a database to produce a website. This can be set to the top level application being supported.

获取更多关于 `app.kubernetes.io`前缀的 Kubernetes labels 的信息 [Kubernetes documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
