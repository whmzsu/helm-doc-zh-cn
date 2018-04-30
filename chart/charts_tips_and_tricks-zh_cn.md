# hart开发Tips和Tricks
本指南涵盖了Helm chart开发人员在构建生产级质量的chart时学到的一些提示和技巧。

## 了解模板函数

Helm使用Go模板[Go templates](https://godoc.org/text/template) 来模板化你的资源文件。虽然Go提供了几个内置函数，但我们添加了许多其他函数。

首先，我们在Sprig库[Sprig library](https://godoc.org/github.com/Masterminds/sprig)中添加了几乎所有的功能 。出于安全原因，我们删除了两个：`env`和`expandenv`（这会让chart作者访问Tiller的环境）。

我们还添加了两个特殊的模板函数：`include`和`required`。`include` 函数允许引入另一个模板，然后将结果传递给其他模板函数。

例如，该模板片段包含一个调用的模板`mytpl`，然后将结果小写，然后用双引号将其包起来。

```yaml
value: {{include "mytpl" . | lower | quote}}
```

`required`函数允许根据模板渲染的要求声明特定的值条目。如果该值为空，则模板渲染将失败并显示用户提交的错误消息。

下面的`required`函数示例声明了.Values.who的条目是必需的，并且在缺少该条目时将显示错误消息：

```yaml
value: {{required "A valid .Values.who entry required!" .Values.who }}
```

## 引用字符串，不要引用整数

当使用字符串数据时，引用字符串比把它们留为空白字符更安全：

```
name: {{.Values.MyName | quote }}
```

但是，使用整数时 _不要引用值_。在很多情况下，这可能会导致Kubernetes内部的解析错误。

```
port: {{ .Values.Port }}
```

这种做法不适用于预期为字符串的env变量值，即使它们表示为整数：

```
env:
  -name: HOST
    value: "http://host"
  -name: PORT
    value: "1234"
```

## 使用'include'函数

Go提供了一种使用内置`template`指令将一个模板包含在另一个模板中的方法 。但是，Go模板管道中不能使用内置函数。

为了能够包含模板，然后对该模板的输出执行操作，Helm有一个特殊的`include`函数：

```
{{ include "toYaml" $value | indent 2 }}
```

上面包含一个名为的模板toYaml，传递它$value的值，然后将该模板的输出传递给该indent函数。

因为YAML的缩进级别和空白的很重要，所以这是包含代码片段的好方法，并在相关的上下文中处理缩进。

## 使用'required'函数

Go提供了一种设置模板选项以控制map使用map中不存在的键编制索引时的行为的方法。这通常使用template.Options（“missingkey = option”）设置，其中option可以是默认值，零或错误。将此选项设置为错误将停止执行并出现错误，这应用于map中每个缺失的键。这适用于chart开发人员想要强制为values.yml文件选择值来实施此行为的情况。

该`required`函数使开发人员能够根据模板渲染的要求声明值条目。如果values.yml中的条目为空，模板将不会渲染，并会返回开发人员提供的错误消息。

例如：

```
{{ required "A valid foo is required!" .Values.foo }}
```

上面将在定义.Values.foo时渲染模板，但在未定义.Values.foo时无法渲染并报错退出。

## 创建镜像拉取的Secrets

镜像拉的secrets实质上是注册，用户名和密码的组合。在正在部署的应用程序中可能需要它们，但要创建它们需要多次运行base64。我们可以编写一个帮助程序模板来组合Docker配置文件，以用作Secret的有效载体。这里是一个例子：

首先，假设凭证在`values.yaml`文件中定义如下：

```
imageCredentials:
  registry: quay.io
  username: someone
  password: sillyness
```

然后我们定义我们的帮助模板如下：
```
{{- define "imagePullSecret" }}
{{- printf "{\"auths\": {\"%s\": {\"auth\": \"%s\"}}}" .Values.imageCredentials.registry (printf "%s:%s" .Values.imageCredentials.username .Values.imageCredentials.password | b64enc) | b64enc }}
{{- end }}
```
最后，我们在更大的模板中使用助手模板来创建Secret manifest：

```
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{ template "imagePullSecret" . }}
```

## ConfigMaps或Secrets更改时自动Roll Deployments

通常情况下，configmaps或secrets被作为配置文件注入容器中。根据应用程序的不同，可能需要重新启动才能使用后续更新`helm upgrade`，但如果deployment spece本身未更改，则应用程序会一直以旧配置运行，导致deployment不一致。

该`sha256sum`函数可用于确保在一个文件发生更改时更新deployment的注释部分：

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
[...]
```

另请参阅h`elm upgrade --recreate-pods`标志以解决此问题的稍微不同的方式。

## 告诉Tiller不要删除资源

有时候有一些资源在Helm运行`helm delete`时不应该被删除 。chart开发人员可以将注释添加到资源以防止被删除。

```yaml
kind: Secret
metadata:
  annotations:
    "helm.sh/resource-policy": keep
[...]
```

（需要双引号）

注释`"helm.sh/resource-policy": keep`指示Tiller在helm delete操作过程中跳过此资源。但是，此资源变成孤儿资源。Helm将不再以任何方式管理它。如果`helm install --replace`在已被删除的release上使用，但保留了资源，则这可能会引发问题。

## 使用“Partials”和includes模板

有时候你想在chart中创建一些可重用的部分，无论它们是块还是模板部分。通常，将这些文件保存在自己的文件中会更整洁。

在`templates/`目录中，任何以下划线“_”开头的文件都不会输出Kubernetesmanifest文件。另按照惯例，辅助模板和partials被放置在一个 _helpers.tpl文件中。

## 具有许多依赖关系的复杂chart

官方chart repo存储库[official charts repository](https://github.com/kubernetes/charts)中的许多chart是用于创建更高级应用程序的“构建块”。chart可能用于创建大规模应用程序的实例。在这种情况下，一张伞形chart可能有多个子chart，每个子chart都是整体的一部分。

当前最佳做法是：从各个子chart组成复杂应用程序，创建公开全局配置的顶层伞形chart，然后使用charts/子目录嵌入每个组件chart。

下面的项目说明了两种强大的设计模式：

**SAP's [OpenStack chart](https://github.com/sapcc/openstack-helm):**：该chart在Kubernetes上安装完整的OpenStack IaaS。所有chart都收集在一个GitHub存储库中。

**Deis's [Workflow](https://github.com/deis/workflow/tree/master/charts/workflow):**： 该chart显示了整个Deis PaaS系统的一个chart。但与SAP chart不同的是，该伞形chart是从每个组件构建而来的，每个组件都在不同的Git存储库中进行跟踪。查看`requirements.yaml`文件以查看此chart是如何由其CI/CD流水线组成的。

这两个chart都说明了使用Helm建立复杂环境的成熟技术。

## YAML是JSON的超级Superset

根据YAML规范，YAML是JSON的超集。这意味着任何有效的JSON结构都应该在YAML中有效。

这有一个优点：有时模板开发人员可能会发现使用类似JSON的语法来表示数据结构更容易，而不是处理YAML的空白敏感度。

作为最佳实践，模板应遵循类似YAML的语法，除非 JSON语法大幅降低了格式问题的风险。

## 小心随机值生成
Helm中有一些函数允许生成随机数据，加密密钥等。这些都很好用。但请注意，在升级过程中，模板会被重新执行。当模板运行产生与上次运行不同的数据时，将触发该资源的更新。

## 升级一个release的idempotently
为了在安装和升级发行版时使用相同的命令，请使用以下命令：

```bash
helm upgrade --install <release name> --values <values file> <chart directory>
```
