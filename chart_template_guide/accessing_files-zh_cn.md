# 模板内访问文件

在上一节中，我们介绍了几种创建和访问命名模板的方法。这可以很容易地从另一个模板中导入一个模板。但有时需要导入不是模板的文件，并注入其内容而不通过模板渲染器发送内容。

Helm 通过 `.Files` 对象提供对文件的访问。在我们开始使用模板示例之前，需要注意一些关于它如何工作的内容：

- 向 Helm chart 添加额外的文件是可以的。这些文件将被捆绑并发送给 Tiller。不过要注意，由于 Kubernetes 对象的存储限制，chart 必须小于 1M。
- 通常出于安全原因，某些文件不能通过 `.Files` 对象访问。
    - templates / 无法访问文件。
    - 使用 `.helmignore` 排除的文件不能被访问。
- chart 不保留 UNIX 模式信息，因此文件级权限在涉及 `.Files` 对象时不会影响文件的可用性。

<!-- (see https://github.com/jonschlinkert/markdown-toc) -->

<!-- toc -->

- [基本示例](# 基本示例)
- [路径助手](# 路径助手)
- [Glob 模式](#Glob 模式)
- [ConfigMap 和 Secrets 工具函数](#ConfigMap 和 Secrets 工具函数)
- [编码](# 编码)
- [行](# 行)

# 基本示例

留意这些注意事项，我们编写一个模板，从三个文件读入我们的 ConfigMap。首先，我们将三个文件添加到 chart 中，将所有三个文件直接放在 mychart / 目录中。

`config1.toml`:

```
message = Hello from config 1
```

`config2.toml`:

```
message = This is config 2
```

`config3.toml`:

```
message = Goodbye from config 3
```

这些都是一个简单的 TOML 文件（想想老派的 Windows INI 文件）。我们知道这些文件的名称，所以我们可以使用一个 range 函数来遍历它们并将它们的内容注入到我们的 ConfigMap 中。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  {{- $files := .Files }}
  {{- range tuple "config1.toml" "config2.toml" "config3.toml" }}
  {{ . }}: |-
    {{ $files.Get . }}
  {{- end }}
```

这个配置映射使用了前几节讨论的几种技术。例如，我们创建一个 `$files` 变量来保存 `.Files` 对象的引用。我们还使用该 `tuple` 函数来创建我们循环访问的文件列表。然后我们打印每个文件名（`{{.}}: |-`），然后打印文件的内容 `{{ $files.Get . }}`。

运行这个模板将产生一个包含所有三个文件内容的 ConfigMap：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: quieting-giraf-configmap
data:
  config1.toml: |-
    message = Hello from config 1

  config2.toml: |-
    message = This is config 2

  config3.toml: |-
    message = Goodbye from config 3
```

## 路径助手

在处理文件时，对文件路径本身执行一些标准操作会非常有用。为了协助这个能力，Helm 从 Go 的 [path](https://golang.org/pkg/path/) 包中导入了许多函数供使用。它们都可以使用 Go 包中的相同名称访问，但使用时小写第一个字母，例如，`Base` 变成 `base`，等等

导入的功能是：

- Base
- Dir
- Ext
- IsAbs
- Clean

## Glob 模式

随着 chart 的增长，可能会发现需要组织更多地文件，因此我们提供了一种 `Files.Glob(pattern string)` 方法通过具有灵活性的模式 [glob patterns](https://godoc.org/github.com/gobwas/glob) 协助提取文件。

`.Glob` 返回一个 `Files` 类型，所以可以调用 `Files` 返回对象的任何方法。

例如，想象一下目录结构：


```
foo/:
  foo.txt foo.yaml

bar/:
  bar.go bar.conf baz.yaml
```

Globs 有多个方法可选择：

```yaml
{{ $root := . }}
{{ range $path, $bytes := .Files.Glob "**.yaml" }}
{{ $path }}: |-
{{ $root.Files.Get $path }}
{{ end }}
```

或

```yaml
{{ range $path, $bytes := .Files.Glob "foo/*" }}
{{ $path.base }}: '{{ $root.Files.Get $path | b64enc }}'
{{ end }}
```

## ConfigMap 和 Secrets 工具函数

（不存在于 2.0.2 或更早的版本中）

想要将文件内容放置到 configmap 和 secret 中非常常见，以便在运行时安装到 pod 中。为了解决这个问题，我们在这个 `Files` 类型上提供了一些实用的方法。

为了进一步组织文件，将这些方法与 `Glob` 方法结合使用尤其有用。

根据上面的 [Glob](#Glob 模式) 示例中的目录结构：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: conf
data:
{{ (.Files.Glob "foo/*").AsConfig | indent 2 }}
---
apiVersion: v1
kind: Secret
metadata:
  name: very-secret
type: Opaque
data:
{{ (.Files.Glob "bar/*").AsSecrets | indent 2 }}
```

## 编码

我们可以导入一个文件，并使用 base64 对模板进行编码以确保成功传输：


```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
data:
  token: |-
    {{ .Files.Get "config1.toml" | b64enc }}
```

以上例子将采用 `config1.toml` 文件，我们之前使用的相同文件并对其进行编码：

```yaml
# Source: mychart/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: lucky-turkey-secret
type: Opaque
data:
  token: |-
    bWVzc2FnZSA9IEhlbGxvIGZyb20gY29uZmlnIDEK
```

## 行

有时需要访问模板中文件的每一行。`Lines` 为此提供了一种方便的方法。

```yaml
data:
  some-file.txt: {{ range .Files.Lines "foo/bar.txt" }}
    {{ . }}{{ end }}
```

目前，无法将 `helm install` 期间将外部文件传递给 chart。因此，如果要求用户提供数据，则必须使用 `helm install -f` 或进行加载 `helm install --set`。

这个讨论将我们深入到写作Helm模板的工具和技术中。在下一节中，我们将看到如何使用一个特殊文件`templates/NOTES.txt`，向chart的用户发送安装后指导。
