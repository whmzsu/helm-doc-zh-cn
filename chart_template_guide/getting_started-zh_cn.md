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

```console
$ helm create mychart
Creating mychart
```

从这里开始，我们将在`mychart`目录中工作。

### 快速看一下目录`mychart/templates/`

看一下`mychart/templates/``目录，发现如下几个文件已经存在。

- NOTES.txt：chart的“帮助文本”。这会在用户运行`helm install`时显示给用户。
- deployment.yaml：创建Kubernetes [deployment](http://kubernetes.io/docs/user-guide/deployments/)的基本manifest
- service.yaml：为deployment创建service端点的基本manifest
- _helpers.tpl：放置模板助手的地方，可以在整个chart中重复使用

而我们要做的就是...... 全部删除它们！这样我们就可以从头开始学习我们的教程。实际上，我们将创建自己的NOTES.txt和_helpers.tpl。


```console
$ rm -rf mychart/templates/*.*
```

在编写生产级chart时，使用这些chart的基本版本可能非常有用。所以在你的日常chart制作中，你可能不想删除它们。

## 第一个模板






我们要创建的第一个模板将是a ConfigMap。在Kubernetes中，ConfigMap只是存储配置数据的容器。其他的东西，比如豆荚，可以访问ConfigMap中的数据。

由于ConfigMaps是基础资源，它们为我们提供了一个很好的起点。

我们首先创建一个名为mychart/templates/configmap.yaml：

apiVersion：v1
类：ConfigMap
元数据：
   名称：mychart-configmap
数据：
   myvalue：“ Hello World ”
提示：模板名称不遵循严格的命名模式。但是，我们建议.yaml为YAML文件和.tpl助手使用后缀。

上面的YAML文件是一个简单的ConfigMap，具有最少的必要字段。由于该文件位于templates/目录中，因此将通过模板引擎发送。

在templates/目录中放置一个像这样的纯YAML文件就好了。当Tiller读取这个模板时，它会直接发送给Kubernetes。

有了这个简单的模板，我们现在有一个可安装的图表。我们可以像这样安装它：

$ helm install ./mychart
名称：full-coral
上次部署时间：Tue Nov 1 17:36:01 2016
NAMESPACE：default
状态：已部署

资源：
==> v1 / ConfigMap
名称数据年龄
mychart-configmap 1 1m
在上面的输出中，我们可以看到我们的ConfigMap已经创建。使用Helm，我们可以检索版本并查看加载的实际模板。

$ helm获得清单全珊瑚

---
＃source：mychart / templates / configmap.yaml
apiVersion：v1
kind：ConfigMap
metadata：
  name：mychart-configmap
data：
  myvalue：“Hello World”
该helm get manifest命令获取版本名称（full-coral）并打印出上传到服务器的所有Kubernetes资源。每个文件都以---YAML文档的开始为开始，然后是一个自动生成的注释行，告诉我们该模板文件生成的这个YAML文档。

从那里开始，我们可以看到YAML数据正是我们在我们的configmap.yaml文件中所做的 。

现在我们可以删除我们的版本：helm delete full-coral。

添加一个简单的模板调用
硬编码name:成资源通常被认为是不好的做法。名称应该是唯一的一个版本。所以我们可能希望通过插入发行版名称来生成一个名称字段。

提示：name:由于DNS系统的限制，该字段限制为63个字符。因此，版本名称限制为53个字符。Kubernetes 1.3及更早版本仅限于24个字符（即14个字符名称）。

让我们相应地改变configmap.yaml。

apiVersion：v1
kind：ConfigMap
metadata：
   name：{{.Release.Name}}  -  configmap
data：
   myvalue：“ Hello World ”
name:现在这个领域 的价值发生了巨大的变化{{ .Release.Name }}-configmap。

模板指令包含在{{和}}块中。

模板指令{{ .Release.Name }}将发布名称注入模板。传递给模板的值可以认为是命名空间对象，其中dot（.）分隔每个命名空间元素。

前面的前一个小圆点Release表示我们从这个范围的最上面的命名空间开始（我们将稍微谈一下范围）。所以我们可以这样理解.Release.Name：“从顶层命名空间开始，找到Release对象，然后在里面查找名为Name” 的对象。

该Release对象是Helm的内置对象之一，稍后我们将更深入地介绍它。但就目前而言，这足以说明这将显示Tiller分配给我们发布的版本名称。

现在，当我们安装我们的资源时，我们会立即看到使用这个模板指令的结果：

$ helm install ./mychart
名称：clunky-serval
最后部署：Tue Nov 1 17:45:37 2016
NAMESPACE：default
状态：已部署

资源：
==> v1 / ConfigMap
名称数据年龄
clunky-serval-configmap 1 1m
请注意，在该RESOURCES部分中，我们看到的名称clunky-serval-configmap 不是mychart-configmap。

您可以运行helm get manifest clunky-serval以查看整个生成的YAML。

有嵌入在模板指令YAML文件：在这一点上，我们已经在他们最基本看到模板{{和}}。在下一部分中，我们将深入研究模板。但在继续之前，有一个快速技巧可以使建筑模板更快：当您想测试模板渲染，但实际上没有安装任何东西时，可以使用helm install --debug --dry-run ./mychart。这会将图表发送到Tiller服务器，它将呈现模板。但不是安装图表，它会将渲染模板返回给您，以便您可以看到输出：

$ helm install --debug --dry-run ./mychart
SERVER：“localhost：44134”
CHART PATH：/Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME：
goodly - guppy TARGET NAMESPACE：default
CHART：mychart 0.1.0
MANIFEST：
---
＃source：mychart / templates / configmap.yaml
apiVersion：v1
kind：ConfigMap
metadata：
  name：goodly-guppy-configmap
data：
  myvalue：“Hello World”
使用--dry-run可以更容易地测试代码，但不能确保Kubernetes本身会接受您生成的模板。最好不要假定你的图表只会因为--dry-run作品而被安装。

在接下来的几节中，我们将采用我们在这里定义的基本图表，并详细探索Helm模板语言。我们将开始使用内置对象。
