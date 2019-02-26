# 子 chart 和全局值

到目前为止，我们只用一个 chart。但是 chart 可以有称为子 chart 的依赖关系，它们也有自己的值和模板。在本节中，我们将创建一个子 chart，并查看我们可以从模板中访问值的不同方式。

在我们深入了解代码之前，需要了解一些有关子 chart 的重要细节。

- 子 chart 被认为是 “独立的”，这意味着子 chart 不能明确依赖于其父 chart。
- 因此，子 chart 无法访问其父项的值。
- 父 chart 可以覆盖子 chart 的值。
- Helm 有全局值的概念，可以被所有 chart 访问。

当我们在本节中通过示例时，其中许多概念将变得更加清晰。

## 创建一个子 chart

对于这些练习，我们将从本指南开始时创建的 chart`mychart/` 开始，并在其中添加一个新 chart。

```bash
$ cd mychart/charts
$ helm create mysubchart
Creating mysubchart
$ rm -rf mysubchart/templates/*.*
```

注意，和以前一样，我们删除了所有的基本模板，以便我们可以从头开始。在本指南中，我们专注于模板如何工作，而不是管理依赖关系。但 chart 指南有更多关于子 chart 工作的信息。




## 将值和模板添加到子 chart

接下来，我们为 `mysubchart`chart 创建一个简单的模板和 values 文件。应该已经有一个 values.yaml 在文件夹 mychart/charts/mysubchart 中了。我们将这样设置：

```yaml
dessert: cake
```

接下来，我们将在下面创建一个新的 ConfigMap 模板 `mychart/charts/mysubchart/templates/configmap.yaml`：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-cfgmap2
data:
  dessert: {{.Values.dessert}}
```

由于每个子 chart 都是独立的 chart，因此我们可以给 mysubchart 自行测试：

```bash
$ helm install --dry-run --debug mychart/charts/mysubchart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart/charts/mysubchart
NAME:   newbie-elk
TARGET NAMESPACE:   default
CHART:  mysubchart 0.1.0
MANIFEST:
---
# Source: mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: newbie-elk-cfgmap2
data:
  dessert: cake
```

## 覆盖子 chart 中的值

我们原来的 chart，`mychart` 现在是其 `mysubchart` 的父 chart,。这种关系完全是因为 mysubchart 内在 mychart/charts 目录中。

由于 mychart 是父级，我们可以指定配置 mychart 并将配置推入 mysubchart。例如，我们可以 mychart/values.yaml 像这样修改：

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream
```

请注意最后两行。该 mysubchart 部分内的任何指令都将发送到 mysubchart chart。所以如果我们运行 helm install --dry-run --debug mychart，我们将看到的一个是 mysubchartConfigMap：

```yaml
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: unhinged-bee-cfgmap2
data:
  dessert: ice cream
```

顶层的值现在已经覆盖了子 chart 的值。

这里有一个重要的细节需要注意。我们没有改变 mychart/charts/mysubchart/templates/configmap.yaml 模板指向 `.Values.mysubchart.dessert`。从该模板的角度来看，该值仍位于 `.Values.dessert`。随着模板引擎一起传递值，它会设置范围。所以对于 mysubchart 模板，只有指定给 mysubchart 的值才会在 `.Values` 里。

但有时候，确实希望某些值可用于所有模板。这是使用全局 chart 值完成的。

## 全局 chart 值

全局值是可以从任何 chart 或子 chart 用完全相同的名称访问的值。全局值需要明确声明。不能像使用现有的非全局值一样来使用全局值。

values 数据类型有一个保留部分，称为 `Values.global`, 可以设置全局值。让我们在我们的 mychart/values.yaml 文件中设置一个。

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream

global:
  salad: caesar
```

因为这样全局值的使用方法，`mychart/templates/configmap.yaml` 和 `mychart/charts/mysubchart/templates/configmap.yaml` 都能够访问该值 `{{.Values.global.salad}}`。

`mychart/templates/configmap.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  salad: {{.Values.global.salad}}
```

`mysubchart/templates/configmap.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-cfgmap2
data:
  dessert: {{.Values.dessert}}
  salad: {{.Values.global.salad}}
```

现在，如果我们运行 dry run，我们会在两个输出中看到相同的值：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: silly-snake-configmap
data:
  salad: caesar

---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: silly-snake-cfgmap2
data:
  dessert: ice cream
  salad: caesar
```

全局变量对于传递这样的信息非常有用，但它确实需要一些计划来确保将正确的模板配置为使用全局变量。

## 与子 chart 共享模板

父 chart 和子 chart 可以共享模板。任何 chart 中的任何定义块都可用于其他 chart。

例如，我们可以像这样定义一个简单的模板：


```yaml
{{- define "labels"}}from: mychart{{ end }}
```

回想一下模板上的标签是如何全局共享的。因此，`labels` chart 可以包含在其他 chart 中。

尽管 chart 开发人员可以选择 `include` 和 `template`, 使用 `include` 的一个优点是，include 可以动态地引用模板：

```yaml
{{include $mytemplate}}
```

以上例子不会引用 $mytemplate。`template` 相反，将只接受一个字符串。

## 避免使用块
Go 模板语言提供了一个 `block` 关键字，允许开发人员提供一个默认的实现，后续将被覆盖。在 Helm chart 中，块不是重写的最佳工具，因为如果提供了同一个块的多个实现，那么所选哪个是不可预知的。

建议是改为使用`include`。
