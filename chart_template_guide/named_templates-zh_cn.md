# 命名模板

现在是开始创建超过一个模板的时候了。在本节中，我们将看到如何在一个文件中定义命名模板，然后在别处使用它们。命名模板（有时称为部分或子模板）是限定在一个文件内部的模板，并起一个名称。我们有两种创建方法，以及几种不同的使用方法。

在 “流量控制” 部分中，我们介绍了声明和管理模板三个动作：`define`，`template`，和 `block`。在本节中，我们将介绍这三个动作，并介绍一个 `include` 函数，与 `template` 类似功能。

在命名模板时要注意一个重要的细节：模板名称是全局的。如果声明两个具有相同名称的模板，则最后加载一个模板是起作用的模板。由于子 chart 中的模板与顶级模板一起编译，因此注意小心地使用特定 chart 的名称来命名模板。

通用的命名约定是为每个定义的模板添加 chart 名称：`{{ define "mychart.labels" }}`。通过使用特定 chart 名称作为前缀，我们可以避免由于同名模板的两个不同 chart 而可能出现的任何冲突。

## partials 和 `_` 文件

到目前为止，我们已经使用了一个文件，一个文件包含一个模板。但 Helm 的模板语言允许创建指定的嵌入模板，可以通过名称访问。

在我们开始编写这些模板之前，有一些文件命名约定值得一提：

* 大多数文件 `templates/` 被视为包含 Kubernetes manifests
* `NOTES.txt` 是一个例外
* 名称以下划线（`_`）开头的文件被假定为没有内部 manifest。这些文件不会渲染 Kubernetes 对象定义，而是在其他 chart 模板中随处可用以供调用。

这些文件用于存储 partials 和辅助程序。事实上，当我们第一次创建时 mychart，我们看到一个叫做文件 `_helpers.tpl`。该文件是模板 partials 的默认位置。

## 用 `define` 和 `template` 声明和使用模板

该 define 操作允许我们在模板文件内创建一个命名模板。它的语法如下所示：

```yaml
{{ define "MY.NAME" }}
  # body of template here
{{ end }}
```

例如，我们可以定义一个模板来封装一个 Kubernetes 标签块：

```yaml
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```


现在我们可以将此模板嵌入到现有的 ConfigMap 中，然后将其包含在 template 操作中：

```yaml
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

当模板引擎读取该文件时，它将存储引用 mychart.labels 直到 template "mychart.labels" 被调用。然后它将在文件内渲染该模板。所以结果如下所示：


```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: running-panda-configmap
  labels:
    generator: helm
    date: 2016-11-02
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

通常，Helm chart 通常将这些模板放入 partials 文件中，通常是 `_helpers.tpl`。让我们在这里移动这个功能：

```yaml
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

按照惯例，define 函数应该有一个简单的文档块（`{{/* ... */}}`）来描述他们所做的事情。

即使这个定义在 `_helpers.tpl`，它仍然可以在 configmap.yaml 以下位置访问：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

如上所述，** 模板名称是全局的 ** 。因此，如果两个模板被命名为相同的名称，则最后一次使用的模板将被使用。由于子 chart 中的模板与顶级模板一起编译，因此最好使用 chart 专用名称命名模板。一个流行的命名约定是为每个定义的模板添加 chart 名称：`{{ define "mychart.labels" }}`。

## 设置模板的范围

在我们上面定义的模板中，我们没有使用任何对象。我们只是使用函数。让我们修改我们定义的模板以包含 chart 名称和 chart 版本：

```yaml
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
{{- end }}
```

如果我们这样做，将不会得到我们所期望的结果：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: moldy-jaguar-configmap
  labels:
    generator: helm
    date: 2016-11-02
    chart:
    version:
```

名称和版本发生了什么变化？他们不在我们定义的模板的范围内。当一个已命名的模板（用于创建 define）被渲染时，它将接收由该 template 调用传入的作用域。在我们的例子中，我们包含了这样的模板：


```yaml
{{- template "mychart.labels" }}
```

没有范围被传入，因此在模板中我们无法访问任何内容.。虽然这很容易解决。我们只需将范围传递给模板：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" . }}
```

请注意，我们在调用 template 时末尾传递了 `.`。我们可以很容易地通过 `.Values` 或者 `.Values.favorite` 或者我们想要的任何范围。但是我们想要的是顶级范围。

现在，当我们用 `helm install --dry-run --debug ./mychart` 执行这个模板，我们得到这个：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: plinking-anaco-configmap
  labels:
    generator: helm
    date: 2016-11-02
    chart: mychart
    version: 0.1.0
```

现在 `{{ .Chart.Name }}` 解析为 `mychart`,`{{ .Chart.Version }}` 解析为 `0.1.0`。

## include 函数

假设我们已经定义了一个如下所示的简单模板：

```yaml
{{- define "mychart.app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}+{{ .Release.Time.Seconds }}"
{{- end -}}
```

现在我想插入到我的模板的 `labels:` 部分和 `data:` 部分：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{ template "mychart.app" .}}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ template "mychart.app" . }}
```

输出不是我们所期望的：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: measly-whippet-configmap
  labels:
    app_name: mychart
app_version: "0.1.0+1478129847"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
app_name: mychart
app_version: "0.1.0+1478129847"
```

注意，app_version 缩进在两个地方都是错误的。为什么？因为被替换的模板具有与右侧对齐的文本。因为 `template` 是一个动作，而不是一个函数，所以没有办法将 template 调用的输出传递给其他函数; 数据只是内嵌插入。

为了解决这个问题，Helm 提供了一个替代 `template` 方案，将模板的内容导入到当前管道中，并将其传递到管道中的其函数。

这里是上面的例子，用 `indent` 纠正正确缩进 `mychart_app` 模板：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{ include "mychart.app" . | indent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ include "mychart.app" . | indent 2 }}
```

现在生成的 YAML 每个部分都正确缩进：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-mole-configmap
  labels:
    app_name: mychart
    app_version: "0.1.0+1478129987"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
  app_version: "0.1.0+1478129987"
```

> 在 Helm 模板中使用 `include` 比 `template` 会更好，可以更好地为 YAML 处理输出格式。

有时我们想要导入内容，但不是作为模板。也就是说，我们要逐字输入文件。我们下一节中描述可以通过访问`.Files`的对象来读取文件。
