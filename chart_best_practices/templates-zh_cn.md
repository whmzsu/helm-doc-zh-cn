# 模板
最佳实践指南的这一部分重点介绍模板。

## templates 目录结构

templates 目目录的结构应如下所示：

- 如果他们产生 YAML 输出，模板文件应该有扩展名 `.yaml`。扩展名. tpl 可用于产生不需要格式化内容的模板文件。
- 模板文件名应该使用横线符号（`my-example-configmap.yaml`），而不是 camelcase。
- 每个资源定义应该在它自己的模板文件中。
- 模板文件名应该反映名称中的资源种类。例如 `foo-pod.yaml`， `bar-svc.yaml`

## 定义模板的名称

定义的模板（在 `{{define}}` 指令内创建的模板）可以全局访问。这意味着 chart 及其所有子 chart 都可以访问所有使用 `{{ define }}` 创建的模板。

出于这个原因，所有定义的模板名称应该是带有某个 namespace。

正确：

```yaml
{{- define "nginx.fullname"}}
{{/* ... */}}
{{end -}}
```

不正确：

```yaml
{{- define "fullname" -}}
{{/* ... */}}
{{end -}}
```

强烈建议通过 `helm create` 命令创建新 chart，因为根据此最佳做法自动定义模板名称。

## 格式化模板

模板应该使用两个空格缩进（不是制表符）。

模板指令在大括号之后和大括号之前应该有空格：

正确：

```
{{.foo}}
{{print "foo"}}
{{- print "bar" -}}
```

不正确：

```
{{.foo}}
{{print "foo"}}
{{-print "bar"-}}
```

模板应尽可能地填充空格：

```
foo:
  {{- range .Values.items}}
  {{.}}
  {{end -}}
```

块（如控制结构）可以缩进以指示模板代码的流向。

```
{{if $foo -}}
  {{- with .Bar}}Hello{{ end -}}
{{- end -}}
```

但是，由于 YAML 是一种面向空格的语言，因此代码缩进有时经常不能遵循该约定。

## 生成模板中的空格

最好将生成的模板中的空格保持最小。特别是，许多空行不应该彼此相邻。但偶尔空行（特别是逻辑段之间）很好。

这是最好的：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: example
  labels:
    first: first
    second: second
```

这没关系：

```yaml
apiVersion: batch/v1
kind: Job

metadata:
  name: example

  labels:
    first: first
    second: second

```

但这应该避免：

```yaml
apiVersion: batch/v1
kind: Job

metadata:
  name: example





  labels:
    first: first

    second: second

```
## Resource Naming in Templates

将 `name:` 硬编码到资源中通常被认为是不好的做法。名称对于 release 应该是唯一的。因此，我们可能希望通过插入 release 名称来生成 name 字段，例如：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{.Release.Name}}-myservice
```

Or if there is only one resource of this kind then we could use .Release.Name or the template fullname function defined in \_helpers.tpl (which uses release name):

或者，如果只有一个此类资源，那么可以使用 .Release.Name 或 \_helpers.tpl（使用 release 名称）中定义的模板 fullname 函数：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{template "fullname" .}}
```
尽快如此，可能还存在不会来自固定名称的命名冲突的情况。在这些情况下，固定名称可能使应用程序更容易找到诸如服务之类的资源。如果需要固定名称，那么一种可能的管理方法是通过使用 values.yaml 中的 service.name 值来显式设置名称（如果提供的话）:

```yaml
apiVersion: v1
kind: Service
metadata:
  {{- if .Values.service.name}}
    name: {{.Values.service.name}}
  {{- else}}
    name: {{template "fullname" .}}
  {{- end}}
```

## 注释（YAML 注释与模板注释）
YAML 和头盔模板都有注释标记。

YAML 注释：

```yaml
# This is a comment
type: sprocket
```

模板注释：

```yaml
{{- /*
This is a comment.
*/ -}}
type: frobnitz
```

记录模板功能时应使用模板注释，如解释定义的模板：

```yaml
{{- /*
mychart.shortname provides a 6 char truncated version of the release name.
*/ -}}
{{define "mychart.shortname" -}}
{{.Release.Name | trunc 6}}
{{- end -}}

```

在模板内部，当 Helm 用户可能（有可能）在调试过程中看到注释时，可以使用 YAML 注释。

```
# This may cause problems if the value is more than 100Gi
memory: {{.Values.maxMem | quote}}
```

上面的注释在用户运行 `helm install --debug` 时可见，而在 `{{- /* */ -}}` 部分中指定的注释不是。

## 在模板和模板输出中使用 JSON

YAML 是 JSON 的超集。在某些情况下，使用 JSON 语法可以比其他 YAML 表示更具可读性。

例如，这个 YAML 更接近表达列表的正常 YAML 方法：

```yaml
arguments:
  - "--dirname"
  - "/foo"
```


但是，当折叠为 JSON 列表样式时，它更容易阅读：

```yaml
arguments: ["--dirname", "/foo"]
```

使用 JSON 增加易读性是很好的。但是，不应该使用 JSON 语法来表示更复杂的构造。

在处理嵌入到YAML中的纯JSON时（例如init容器配置），使用JSON格式当然是合适的。
