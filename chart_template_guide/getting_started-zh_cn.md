# 开始使用chart模板

在本指南的这部分，我们将创建一个chart，然后添加第一个模板。我们在这里创建的chart将在指南的其他部分使用。

开始，我们来看一下Helm chart。

## Charts

如chart指南中所述，Helm chart的结构如下所示：

```
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```

`templates/`目录用于放置模板文件。当Tiller评估chart时，它将`templates/`通过模板渲染引擎发送目录中的所有文件。然后，Tiller收集这些模板的结果并将它们发送给Kubernetes。

`values.yaml`文件对模板也很重要。该文件包含chart默认值。这些值可能在用户在`helm install`或 `helm upgrade`期间被覆盖。

`Chart.yaml`文件包含chart的说明。可以从模板中查看访问它。该`charts/`目录可能包含其他chart（我们称之为子chart）。在本指南的后面，我们将看到它们在模板渲染方面如何起作用。

## 初始 chart

对于本指南，我们将创建一个名为mychart的简单chart，然后我们将在chart内部创建一些模板。

```bash
$ helm create mychart
Creating mychart
```

从这里开始，我们将在`mychart`目录中工作。

### 快速看一下目录`mychart/templates/`

看一下`mychart/templates/`目录，发现如下几个文件已经存在。

- NOTES.txt：chart的“帮助文本”。这会在用户运行`helm install`时显示给用户。
- deployment.yaml：创建Kubernetes [deployment](http://kubernetes.io/docs/user-guide/deployments/)的基本manifest
- service.yaml：为deployment创建service端点的基本manifest
- `_helpers.tpl`：放置模板助手的地方，可以在整个chart中重复使用

而我们要做的就是...... 全部删除它们！这样我们就可以从头开始学习我们的教程。实际上，我们将创建自己的NOTES.txt和_helpers.tpl。


```bash
$ rm -rf mychart/templates/*.*
```

在编写生产级chart时，使用这些chart的基本版本可能非常有用。所以在你的日常chart制作中，可以不删除它们。

## 第一个模板


我们要创建的第一个模板将是一个ConfigMap。在Kubernetes中，ConfigMap只是存储配置数据的地方。其他的东西，比如Pod，可以访问ConfigMap中的数据。

由于ConfigMaps是基础资源，它们为我们提供了一个很好的起点。

我们首先创建一个名为mychart/templates/configmap.yaml：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

**提示：** 模板名称不遵循严格的命名模式。但是，我们建议`.yaml`为YAML文件后缀，`.tpl`为模板助手后缀。

上面的YAML文件是一个简单的ConfigMap，具有最少的必要字段。由于该文件位于`templates/`目录中，因此将通过模板引擎发送。

在`templates/`目录中放置一个像这样的纯YAML文件。当Tiller读取这个模板时，它会直接发送给Kubernetes。

有了这个简单的模板，我们现在有一个可安装的chart。我们可以像这样安装它：

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

在上面的输出中，我们可以看到我们的ConfigMap已经创建。使用Helm，我们可以检索版本并查看加载的实际模板。

$ helm获得清单全珊瑚

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

该`helm get manifest`命令获取release名称（full-coral）并打印出上传到服务器的所有Kubernetes资源。每个文件都以`---`开始作为YAML文档的开始，然后是一个自动生成的注释行，告诉我们该模板文件生成的这个YAML文档。

从那里开始，我们可以看到YAML数据正是我们在我们的`configmap.yaml`文件中所设计的 。

现在我们可以删除我们的release：`helm delete full-coral`。

### 添加一个简单的模板调用

硬编码`name:`成资源通常被认为是不好的做法。名称应该是唯一的一个版本。所以我们可能希望通过插入release名称来生成一个名称字段。

**提示：** name:由于DNS系统的限制，该字段限制为63个字符。因此，release名称限制为53个字符。Kubernetes 1.3及更早版本仅限于24个字符（即14个字符名称）。

让我们改一下`configmap.yaml`。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

`name:`现在这个值发生了变化成了`{{ .Release.Name }}-configmap`。

> 模板指令包含在`{{`和`}}`块中。

模板指令`{{ .Release.Name }}`将release名称注入模板。传递给模板的值可以认为是namespace对象，其中dot（.）分隔每个namespace元素。

Release前面的前一个小圆点表示我们从这个范围的最上面的namespace开始（我们将稍微谈一下scope）。所以我们可以这样理解`.Release.Name：`"从顶层命名空间开始，找到Release对象，然后在里面查找名为`Name` 的对象"。

该Release对象是Helm的内置对象之一，稍后我们将更深入地介绍它。但就目前而言，这足以说明这会显示Tiller分配给我们发布的release名称。

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

注意，在该RESOURCES部分中，我们看到的名称clunky-serval-configmap 不是mychart-configmap。

可以运行helm get manifest clunky-serval以查看整个生成的YAML。

现在，我们看过了基础的模板：YAML文件嵌入了模板指令，通过{{和}}。在下一部分中，我们将深入研究模板。但在继续之前，有一个快速技巧可以使构建模板更快：当您想测试模板渲染，但实际上没有安装任何东西时，可以使用`helm install --debug --dry-run ./mychart`。这会将图表发送到Tiller服务器，它将渲染模板。但不是安装chart，它会将渲染模板返回，以便可以看到输出：

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

使用`--dry-run`可以更容易地测试代码，但不能确保Kubernetes本身会接受生成的模板。最好不要假定你的chart只要`--dry-run`成功而被安装。

在接下来的几节中，我们将采用我们在这里定义的基本chart，并详细探索Helm模板语言。我们将开始使用内置对象。
