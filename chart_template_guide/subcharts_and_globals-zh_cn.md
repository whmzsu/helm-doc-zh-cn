# 子chart和全局值

到目前为止，我们只用一张chart工作。但是chart可以有称为子chart的依赖关系，它们也有自己的值和模板。在本节中，我们将创建一个子chart，并查看我们可以从模板中访问值的不同方式。

在我们深入了解代码之前，需要了解一些有关子chart的重要细节。

子chart被认为是“独立的”，这意味着子chart不能明确依赖于其父chart。
因此，子chart无法访问其父项的值。
父chart可以覆盖子chart的值。
Helm有全局值的概念，可以被所有chart访问。
当我们在本节中通过示例时，其中许多概念将变得更加清晰。

## 创建一个子chart

对于这些练习，我们将从本指南开始时创建的chart`mychart/`开始，并在其中添加一个新chart。

```console
$ cd mychart/charts
$ helm create mysubchart
Creating mysubchart
$ rm -rf mysubchart/templates/*.*
```

注意，和以前一样，我们删除了所有的基本模板，以便我们可以从头开始。在本指南中，我们专注于模板如何工作，而不是管理依赖关系。但chart指南有更多关于子chart工作的信息。




## 将值和模板添加到子chart
接下来，我们为`mysubchart`chart创建一个简单的模板和值文件。应该已经有一个values.yamlin了mychart/charts/mysubchart。我们将这样设置：

甜点：蛋糕
接下来，我们将在下面创建一个新的ConfigMap模板mychart/charts/mysubchart/templates/configmap.yaml：

apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
由于每个子图都是独立的chart，因此我们可以mysubchart自行测试：

$ helm install --dry-run --debug mychart / charts / mysubchart
SERVER：“localhost：44134”
CHART PATH：/Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart/charts/mysubchart
NAME：newbie-elk
TARGET NAMESPACE：default
CHART：mysubchart 0.1.0
MANIFEST：
---
＃source：mysubchart / templates / configmap.yaml
apiVersion：v1
kind：ConfigMap
metadata：
  name：newbie-elk-cfgmap2
data：
  dessert：cake
覆盖父chart中的值
我们原来的chart，mychart现在是其父chartmysubchart。这种关系完全基于mysubchart内在的事实mychart/charts。

由于mychart是父级，我们可以指定配置mychart并将配置推入mysubchart。例如，我们可以mychart/values.yaml像这样修改：

最爱：
   饮料：咖啡
  食物：披
萨比萨餐具：
  - 蘑菇
  - 奶酪
  - 辣椒
  - 洋葱

mysubchart：
   甜点：冰淇淋
请注意最后两行。该mysubchart部分内的任何指令都将发送到mysubchartchart。所以如果我们运行helm install --dry-run --debug mychart，我们将看到的一个是mysubchartConfigMap：

＃来源：mychart /chart/ mysubchart /模板/ configmap.yaml
apiVersion： V1
样： ConfigMap
元数据：
  名称：精神错乱，蜂cfgmap2
数据：
  甜点：冰淇淋
顶层的值现在已经覆盖了子图的价值。

这里有一个重要的细节需要注意。我们没有改变mychart/charts/mysubchart/templates/configmap.yaml指向的模板.Values.mysubchart.dessert。从该模板的角度来看，该值仍位于.Values.dessert。随着模板引擎一起传递值，它会设置范围。所以对于mysubchart模板，只有特定的值mysubchart才可用.Values。

但有时候，您确实希望某些值可用于所有模板。这是使用全局chart值完成的。

全局chart值
全局值是可以从任何chart或子图完全相同的名称访问的值。全局需要明确声明。您不能像现在的全局一样使用现有的非全局。

值数据类型有一个保留部分，称为Values.global可以设置全局值的位置。让我们在我们的mychart/values.yaml文件中设置一个。

最爱：
   饮料：咖啡
  食物：披
萨比萨餐具：
  - 蘑菇
  - 奶酪
  - 辣椒
  - 洋葱

mysubchart：
   甜点：冰淇淋

全球：
   沙拉：凯撒
因为这样全局的工作，既mychart/templates/configmap.yaml和mysubchart/templates/configmap.yaml应该能够访问该值{{ .Values.global.salad}}。

mychart/templates/configmap.yaml：

apiVersion：v1
kind：ConfigMap
metadata：
   name：{{.Release.Name}}  -  configmap
data：
   salad：{{.Values.global.salad}}
mysubchart/templates/configmap.yaml：

apiVersion：v1
kind：ConfigMap
metadata：
   name：{{.Release.Name}}  -  cfgmap2
data：
   dessert：{{.Values.dessert}}
  salad：{{.Values.global.salad}}
现在，如果我们运行干运行安装，我们会在两个输出中看到相同的值：

＃源：mychart / templates / configmap.yaml
apiVersion： v1
类： ConfigMap
元数据：
  名称： silly-snake-configmap
数据：
  沙拉：凯撒

---
＃来源：mychart /chart/ mysubchart /模板/ configmap.yaml
apiVersion： V1
样： ConfigMap
元数据：
  名称：傻蛇-cfgmap2
数据：
  甜点：冰淇淋
  沙拉：凯撒
全局变量对于传递这样的信息非常有用，但它确实需要一些计划来确保将正确的模板配置为使用全局变量。

与子图共享模板
父chart和子chart可以共享模板。任何chart中的任何定义块都可用于其他chart。

例如，我们可以像这样定义一个简单的模板：

{{ -  define“labels”}} from：mychart {{end}}
回想一下模板上的标签是如何全局共享的。因此，labelschart可以包含在其他chart中。

尽管chart开发人员之间的选择include和template使用的一个优点include是，include可以动态地参考模板：

{{include $ mytemplate}}
以上将取消$mytemplate。该template功能，相反，将只接受一个字符串。

避免使用块
Go模板语言提供了一个block关键字，允许开发人员提供一个默认的实现，稍后将被覆盖。在Helmchart中，块不是重写的最佳工具，因为如果提供了同一个块的多个实现，那么所选的那个是不可预知的。

建议是改为使用include。
