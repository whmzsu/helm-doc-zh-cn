# Chart 开发 Tips 和 Tricks
本指南涵盖了 Helm chart 开发人员在构建生产级质量的 chart 时学到的一些提示和技巧。

## 了解模板函数

Helm 使用 Go 模板 [Go templates](https://godoc.org/text/template) 来模板化你的资源文件。虽然 Go 提供了几个内置函数，但我们添加了许多其他函数。

首先，我们在 Sprig 库 [Sprig library](https://godoc.org/github.com/Masterminds/sprig) 中添加了几乎所有的函数 。出于安全原因，我们删除了两个：`env` 和 `expandenv`（这会让 chart 作者访问 Tiller 的环境）。

我们还添加了两个特殊的模板函数：`include` 和 `required`。`include` 函数允许引入另一个模板，然后将结果传递给其他模板函数。

例如，该模板片段包含一个调用的模板 `mytpl`，然后将结果小写，然后用双引号将其包起来。

```yaml
value: {{include "mytpl" . | lower | quote}}
```

`required` 函数允许根据模板渲染的要求声明特定的值条目。如果该值为空，则模板渲染将失败并显示用户提交的错误消息。

下面的 `required` 函数示例声明了. Values.who 的条目是必需的，并且在缺少该条目时将显示错误消息：

```yaml
value: {{required "A valid .Values.who entry required!" .Values.who}}
```

## 引用字符串，不要引用整数

当使用字符串数据时，引用字符串比把它们留为空白字符更安全：

```yaml
name: {{.Values.MyName | quote}}
```

但是，使用整数时 _不要引用值_。在很多情况下，这可能会导致 Kubernetes 内部的解析错误。

```yaml
port: {{.Values.Port}}
```

这种做法不适用于预期为字符串的 env 变量值，即使它们表示为整数：

```yaml
env:
  -name: HOST
    value: "http://host"
  -name: PORT
    value: "1234"
```

## 使用'include'函数

Go 提供了一种使用内置 `template` 指令将一个模板包含在另一个模板中的方法 。但是，Go 模板管道中不能使用内置函数。

为了能够包含模板，然后对该模板的输出执行操作，Helm 有一个特殊的 `include` 函数：

```
{{- include "toYaml" $value | nindent 2}}
```

上面包含一个名为的模板 toYaml，传递它 $value 的值，然后将该模板的输出传递给该 indent 函数。

因为 YAML 的缩进级别和空白的很重要，所以这是包含代码片段的好方法，并在相关的上下文中处理缩进。

## 使用'required'函数

Go 提供了一种设置模板选项以控制 map 使用 map 中不存在的键编制索引时的行为的方法。这通常使用 template.Options（“missingkey = option”）设置，其中 option 可以是默认值，零或错误。将此选项设置为错误将停止执行并出现错误，这应用于 map 中每个缺失的键。这适用于 chart 开发人员想要强制为 values.yml 文件选择值来实施此行为的情况。

该 `required` 函数使开发人员能够根据模板渲染的要求声明值条目。如果 values.yml 中的条目为空，模板将不会渲染，并会返回开发人员提供的错误消息。

例如：

```
{{required "A valid foo is required!" .Values.foo}}
```

上面将在定义. Values.foo 时渲染模板，但在未定义. Values.foo 时无法渲染并报错退出。
## 使用'tpl' 函数

`tpl` 函数允许开发人员将字符串计算为模板内的模板。
这对于将模板字符串作为值传递给 chart 或渲染外部配置文件很有用。

语法: `{{tpl TEMPLATE_STRING VALUES}}`

样例:

```yaml
# values
template: "{{.Values.name}}"
name: "Tom"

# template
{{tpl .Values.template .}}

# output
Tom
```

渲染一个外部配置文件:

```yaml
# external configuration file conf/app.conf
firstName={{.Values.firstName}}
lastName={{.Values.lastName}}

# values
firstName: Peter
lastName: Parker

# template
{{tpl (.Files.Get "conf/app.conf") . }}

# output
firstName=Peter
lastName=Parker
```
## 创建镜像拉取的 Secrets

镜像拉的 secrets 实质上是注册，用户名和密码的组合。在正在部署的应用程序中可能需要它们，但要创建它们需要多次运行 base64。我们可以编写一个帮助程序模板来组合 Docker 配置文件，以用作 Secret 的有效载体。这里是一个例子：

首先，假设凭证在 `values.yaml` 文件中定义如下：

```yaml
imageCredentials:
  registry: quay.io
  username: someone
  password: sillyness
```

然后我们定义我们的帮助模板如下：
```
{{- define "imagePullSecret"}}
{{- printf "{\"auths\": {\"%s\": {\"auth\": \"%s\"}}}" .Values.imageCredentials.registry (printf "%s:%s" .Values.imageCredentials.username .Values.imageCredentials.password | b64enc) | b64enc }}
{{- end}}
```
最后，我们在更大的模板中使用助手模板来创建 Secret manifest：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{template "imagePullSecret" .}}
```

## ConfigMaps 或 Secrets 更改时自动 Roll Deployments

通常情况下，configmaps 或 secrets 被作为配置文件注入容器中。根据应用程序的不同，可能需要重新启动才能使用后续更新 `helm upgrade`，但如果 deployment spece 本身未更改，则应用程序会一直以旧配置运行，导致 deployment 不一致。

该 `sha256sum` 函数可用于确保在一个文件发生更改时更新 deployment 的注释部分：

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
[...]
```

另请参阅 h`elm upgrade --recreate-pods` 标志以解决此问题的稍微不同的方式。

## 告诉 Tiller 不要删除资源

有时候有一些资源在 Helm 运行 `helm delete` 时不应该被删除 。chart 开发人员可以将注释添加到资源以防止被删除。

```yaml
kind: Secret
metadata:
  annotations:
    "helm.sh/resource-policy": keep
[...]
```

（需要双引号）

注释 `"helm.sh/resource-policy": keep` 指示 Tiller 在 helm delete 操作过程中跳过此资源。但是，此资源变成孤儿资源。Helm 将不再以任何方式管理它。如果 `helm install --replace` 在已被删除的 release 上使用，但保留了资源，则这可能会引发问题。

## 使用 “Partials” 和 includes 模板

有时候你想在 chart 中创建一些可重用的部分，无论它们是块还是模板部分。通常，将这些文件保存在自己的文件中会更整洁。

在 `templates/` 目录中，任何以下划线 "-" 开头的文件都不会输出 Kubernetesmanifest 文件。另按照惯例，辅助模板和 partials 被放置在一个 `_helpers.tpl` 文件中。

## 具有许多依赖关系的复杂 chart

官方 chart repo 存储库 [official charts repository](https://github.com/helm/charts) 中的许多 chart 是用于创建更高级应用程序的 “构建块”。chart 可能用于创建大规模应用程序的实例。在这种情况下，一张伞形 chart 可能有多个子 chart，每个子 chart 都是整体的一部分。

当前最佳做法是：从各个子 chart 组成复杂应用程序，创建公开全局配置的顶层伞形 chart，然后使用 charts / 子目录嵌入每个组件 chart。

下面的项目说明了两种强大的设计模式：

**SAP's [OpenStack chart](https://github.com/sapcc/openstack-helm):**：该 chart 在 Kubernetes 上安装完整的 OpenStack IaaS。所有 chart 都收集在一个 GitHub 存储库中。

**Deis's [Workflow](https://github.com/deis/workflow/tree/master/charts/workflow):**： 该 chart 显示了整个 Deis PaaS 系统的一个 chart。但与 SAP chart 不同的是，该伞形 chart 是从每个组件构建而来的，每个组件都在不同的 Git 存储库中进行跟踪。查看 `requirements.yaml` 文件以查看此 chart 是如何由其 CI/CD 流水线组成的。

这两个 chart 都说明了使用 Helm 建立复杂环境的成熟技术。

## YAML 是 JSON 的超级 Superset

根据 YAML 规范，YAML 是 JSON 的超集。这意味着任何有效的 JSON 结构都应该在 YAML 中有效。

这有一个优点：有时模板开发人员可能会发现使用类似 JSON 的语法来表示数据结构更容易，而不是处理 YAML 的空白敏感度。

作为最佳实践，模板应遵循类似 YAML 的语法，除非 JSON 语法大幅降低了格式问题的风险。

## 小心随机值生成
Helm 中有一些函数允许生成随机数据，加密密钥等。这些都很好用。但请注意，在升级过程中，模板会被重新执行。当模板运行产生与上次运行不同的数据时，将触发该资源的更新。

## 升级一个 release 的 idempotently
为了在安装和升级发行版时使用相同的命令，请使用以下命令：

```bash
helm upgrade --install <release name> --values <values file> <chart directory>
```
