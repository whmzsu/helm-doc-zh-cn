# 内置对象
对象从模板引擎传递到模板中。你的代码可以传递对象（我们将在说明 `with` 和 `range` 语句时看到示例）。甚至有几种方法在模板中创建新对象，就像我们稍后会看的 `tuple` 函数一样。

对象可以很简单，只有一个值。或者他们可以包含其他对象或函数。例如，`Release` 对象包含多个对象（如 `Release.Name`）并且 `Files` 对象具有一些函数。

在上一节中，我们使用 `{{ .Release.Name }}` 将 release 的名称插入到模板中。`Release` 是可以在模板中访问的顶级对象之一。

- `Release`：这个对象描述了 release 本身。它里面有几个对象：
- `Release.Name`：release 名称
- `Release.Time`：release 的时间
- `Release.Namespace`：release 的 namespace（如果清单未覆盖）
- `Release.Service`：release 服务的名称（始终是 `Tiller`）。
- `Release.Revision`：此 release 的修订版本号。它从 1 开始，每 `helm upgrade` 一次增加一个。
- `Release.IsUpgrade`：如果当前操作是升级或回滚，则将其设置为 `true`。
- `Release.IsInstall`：如果当前操作是安装，则设置为 `true`。
- `Values`：从 `values.yaml` 文件和用户提供的文件传入模板的值。默认情况下，Values 是空的。
- `Chart`：`Chart.yaml` 文件的内容。任何数据 Chart.yaml 将在这里访问。例如 \{\{.Chart.Name\}\}-\{\{.Chart.Version\}\} 将打印出来 mychart-0.1.0。chart 指南中 [Charts Guide](https://github.com/kubernetes/helm/blob/master/docs/charts.md#the-chartyaml-file) 列出了可用字段
- `Files`：这提供对 chart 中所有非特殊文件的访问。虽然无法使用它来访问模板，但可以使用它来访问 chart 中的其他文件。请参阅 "访问文件" 部分。
- `Files.Get` 是一个按名称获取文件的函数（`.Files.Get config.ini`）
- `Files.GetBytes` 是将文件内容作为字节数组而不是字符串获取的函数。这对于像图片这样的东西很有用。
- `Capabilities`：这提供了关于 Kubernetes 集群支持的功能的信息。
- `Capabilities.APIVersions` 是一组版本信息。
- `Capabilities.APIVersions.Has $version` 指示是否在群集上启用版本（`batch/v1`）。
- `Capabilities.KubeVersion` 提供了查找 Kubernetes 版本的方法。它具有以下值：Major，Minor，GitVersion，GitCommit，GitTreeState，BuildDate，GoVersion，Compiler，和 Platform。
- `Capabilities.TillerVersion` 提供了查找 Tiller 版本的方法。它具有以下值：SemVer，GitCommit，和 GitTreeState。
- `Template`：包含有关正在执行的当前模板的信息
- `Name`：到当前模板的 namespace 文件路径（例如 `mychart/templates/mytemplate.yaml`）
- `BasePath`：当前 chart 模板目录的 namespace 路径（例如 mychart/templates）。

这些值可用于任何顶级模板。我们稍后会看到，这并不意味着它们将在任何地方都要有。

内置值始终以大写字母开头。这符合Go的命名约定。当你创建自己的名字时，你可以自由地使用适合你的团队的惯例。一些团队，如Kubernetes chart团队，选择仅使用首字母小写字母来区分本地名称与内置名称。在本指南中，我们遵循该约定。
