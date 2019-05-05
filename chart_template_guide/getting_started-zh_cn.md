# 开始使用 chart 模板

在本指南的这部分，我们将创建一个 chart，然后添加第一个模板。我们在这里创建的 chart 将在指南的其他部分使用。

开始，我们来看一下 Helm chart。

## Charts

如 chart 指南中所述，Helm chart 的结构如下所示：

```
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```

`templates/` 目录用于放置模板文件。当 Tiller 评估 chart 时，它将 `templates/` 通过模板渲染引擎发送目录中的所有文件。然后，Tiller 收集这些模板的结果并将它们发送给 Kubernetes。

`values.yaml` 文件对模板也很重要。该文件包含 chart 默认值。这些值可能在用户在 `helm install` 或 `helm upgrade` 期间被覆盖。

`Chart.yaml` 文件包含 chart 的说明。可以从模板中查看访问它。该 `charts/` 目录可能包含其他 chart（我们称之为子 chart）。在本指南的后面，我们将看到它们在模板渲染方面如何起作用。

## 初始 chart

对于本指南，我们将创建一个名为 mychart 的简单 chart，然后我们将在 chart 内部创建一些模板。

```bash
$ helm create mychart
Creating mychart
```

从这里开始，我们将在 `mychart` 目录中工作。

### 快速看一下目录 `mychart/templates/`

看一下 `mychart/templates/` 目录，发现如下几个文件已经存在。

- NOTES.txt：chart 的 “帮助文本”。这会在用户运行 `helm install` 时显示给用户。
- deployment.yaml：创建 Kubernetes [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 的基本 manifest
- service.yaml：为 deployment 创建 service 端点 [service endpoint](https://kubernetes.io/docs/concepts/services-networking/service/) 的基本 manifest
- `_helpers.tpl`：放置模板助手的地方，可以在整个 chart 中重复使用

而我们要做的就是...... 全部删除它们！这样我们就可以从头开始学习我们的教程。实际上，我们将创建自己的 NOTES.txt 和_helpers.tpl。


```bash
$ rm -rf mychart/templates/*.*
```

在编写生产级 chart 时，使用这些 chart 的基本版本可能非常有用。所以在你的日常 chart 制作中，可以不删除它们。

## 第一个模板


我们要创建的第一个模板将是一个 ConfigMap。在 Kubernetes 中，ConfigMap 只是存储配置数据的地方。其他的东西，比如 Pod，可以访问 ConfigMap 中的数据。

由于 ConfigMaps 是基础资源，它们为我们提供了一个很好的起点。

我们首先创建一个名为 mychart/templates/configmap.yaml：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

** 提示：** 模板名称不遵循严格的命名模式。但是，我们建议 `.yaml` 为 YAML 文件后缀，`.tpl` 为模板助手后缀。

上面的 YAML 文件是一个简单的 ConfigMap，具有最少的必要字段。由于该文件位于 `templates/` 目录中，因此将通过模板引擎发送。

在 `templates/` 目录中放置一个像这样的纯 YAML 文件。当 Tiller 读取这个模板时，它会直接发送给 Kubernetes。

有了这个简单的模板，我们现在有一个可安装的 chart。我们可以像这样安装它：

```bash
$ helm install ./mychart
NAME: full-coral
LAST DEPLOYED: Tue Nov  1 17:36:01 2016
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                DATA      AGE
mychart-configmap   1         1m
```

在上面的输出中，我们可以看到我们的 ConfigMap 已经创建。使用 Helm，我们可以检索版本并查看加载的实际模板。

```bash
$ helm get manifest full-coral

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

该 `helm get manifest` 命令获取 release 名称（full-coral）并打印出上传到服务器的所有 Kubernetes 资源。每个文件都以 `---` 开始作为 YAML 文档的开始，然后是一个自动生成的注释行，告诉我们该模板文件生成的这个 YAML 文档。

从那里开始，我们可以看到 YAML 数据正是我们在我们的 `configmap.yaml` 文件中所设计的 。

现在我们可以删除我们的 release：`helm delete full-coral`。

### 添加一个简单的模板调用

硬编码 `name:` 成资源通常被认为是不好的做法。名称应该是唯一的一个版本。所以我们可能希望通过插入 release 名称来生成一个名称字段。

** 提示：** name: 由于 DNS 系统的限制，该字段限制为 63 个字符。因此，release 名称限制为 53 个字符。Kubernetes 1.3 及更早版本仅限于 24 个字符（即 14 个字符名称）。

让我们改一下 `configmap.yaml`。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

`name:` 现在这个值发生了变化成了 `{{ .Release.Name }}-configmap`。

模板指令包含在 `{{` 和 `}}` 块中。

模板指令 `{{ .Release.Name }}` 将 release 名称注入模板。传递给模板的值可以认为是 namespace 对象，其中 dot（.）分隔每个 namespace 元素。

Release 前面的前一个小圆点表示我们从这个范围的最上面的 namespace 开始（我们将稍微谈一下 scope）。所以我们可以这样理解 `.Release.Name：`"从顶层命名空间开始，找到 Release 对象，然后在里面查找名为 `Name` 的对象"。

该 Release 对象是 Helm 的内置对象之一，稍后我们将更深入地介绍它。但就目前而言，这足以说明这会显示 Tiller 分配给我们发布的 release 名称。

现在，当我们安装我们的资源时，我们会立即看到使用这个模板指令的结果：

```bash
$ helm install ./mychart
NAME: clunky-serval
LAST DEPLOYED: Tue Nov  1 17:45:37 2016
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                      DATA      AGE
clunky-serval-configmap   1         1m
```

注意，在该 RESOURCES 部分中，我们看到的名称 clunky-serval-configmap 不是 mychart-configmap。

可以运行 helm get manifest clunky-serval 以查看整个生成的 YAML。

现在，我们看过了基础的模板：YAML 文件嵌入了模板指令，通过 {{和}}。在下一部分中，我们将深入研究模板。但在继续之前，有一个快速技巧可以使构建模板更快：当您想测试模板渲染，但实际上没有安装任何东西时，可以使用 `helm install --debug --dry-run ./mychart`。这会将 chart 发送到 Tiller 服务器，它将渲染模板。但不是安装 chart，它会将渲染模板返回，以便可以看到输出：

```bash
$ helm install --debug --dry-run ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   goodly-guppy
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: goodly-guppy-configmap
data:
  myvalue: "Hello World"

```

使用 `--dry-run` 可以更容易地测试代码，但不能确保 Kubernetes 本身会接受生成的模板。最好不要假定你的 chart 只要 `--dry-run` 成功而被安装。

在接下来的几节中，我们将采用我们在这里定义的基本chart，并详细探索Helm模板语言。我们将开始使用内置对象。
