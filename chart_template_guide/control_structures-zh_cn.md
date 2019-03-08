# 流程控制
控制结构（模板说法中称为 “动作”）为模板作者提供了控制模板生成流程的能力。Helm 的模板语言提供了以下控制结构：

- `if/else` 用于创建条件块
- `with` 指定范围
- `range`，它提供了一个 “for each” 风格的循环

除此之外，它还提供了一些声明和使用命名模板段的操作：

- `define` 在模板中声明一个新的命名模板
- `template` 导入一个命名模板
- `block` 声明了一种特殊的可填写模板区域

在本节中，我们将谈论 `if`，`with` 和 `range`。其他内容在本指南后面的 “命名模板” 一节中介绍。

## if/else

我们要看的第一个控制结构是用于在模板中有条件地包含文本块。这就是 if/else 块。

条件的基本结构如下所示：

```
{{if PIPELINE}}
  # Do something
{{else if OTHER PIPELINE}}
  # Do something else
{{else}}
  # Default case
{{end}}
```

注意，我们现在讨论的是管道而不是值。其原因是要明确控制结构可以执行整个管道，而不仅仅是评估一个值。

如果值为如下情况，则管道评估为 false。

- 一个布尔型的假
- 一个数字零
- 一个空的字符串
- 一个 `nil`（空或 null）
- 一个空的集合（`map`，`slice`，`tuple`，`dict`，`array`）

在其他情况下, 条件值为 _true_ 此管道被执行。

我们为 ConfigMap 添加一个简单的条件。如果饮料被设置为咖啡，我们将添加另一个设置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello World"
  drink: {{.Values.favorite.drink | default "tea" | quote}}
  food: {{.Values.favorite.food | upper | quote}}
  {{if and .Values.favorite.drink (eq .Values.favorite.drink "coffee") }}mug: true{{ end }}

```
注意 `.Values.favorite.drink` 必须已定义，否则在将它与 “coffee” 进行比较时会抛出错误。由于我们在上一个例子中注释掉了 `drink：coffee`，因此输出不应该包含 `mug：true` 标志。但是如果我们将该行添加回 `values.yaml` 文件中，输出应该如下所示:

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

## 控制空格

在查看条件时，我们应该快速查看模板中的空格控制方式。让我们看一下前面的例子，并将其格式化为更容易阅读的格式：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello World"
  drink: {{.Values.favorite.drink | default "tea" | quote}}
  food: {{.Values.favorite.food | upper | quote}}
  {{if eq .Values.favorite.drink "coffee"}}
    mug: true
  {{end}}
```

最初，这看起来不错。但是如果我们通过模板引擎运行它，我们会得到一个错误的结果：

```bash
$ helm install --dry-run --debug ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
Error: YAML parse error on mychart/templates/configmap.yaml: error converting YAML to JSON: yaml: line 9: did not find expected key
```

发生了什么？由于上面的空格，我们生成了不正确的 YAML。

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eyewitness-elk-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
    mug: true
```

mug 不正确地缩进。让我们简单地缩进那行，然后重新运行：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello World"
  drink: {{.Values.favorite.drink | default "tea" | quote}}
  food: {{.Values.favorite.food | upper | quote}}
  {{if eq .Values.favorite.drink "coffee"}}
  mug: true
  {{end}}
```

当我们发送该信息时，我们会得到有效的 YAML，但仍然看起来有点意思：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: telling-chimp-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"

  mug: true

```

请注意，我们在 YAML 中收到了一些空行。为什么？当模板引擎运行时，它将删除 `{{` 和 `}}` 中的空白内容，但是按原样保留剩余的空白。

YAML 中的缩进空格是严格的，因此管理空格变得非常重要。幸运的是，Helm 模板有几个工具可以帮助我们。

首先，可以使用特殊字符修改模板声明的大括号语法，以告诉模板引擎填充空白。`{{- `（添加了破折号和空格）表示应该将格左移，而 ` -}}` 意味着应该删除右空格。注意！换行符也是空格！

> 确保 `-` 和其他指令之间有空格。`{{- 3}}` 意思是 “删除左空格并打印 3”，而 `{{-3}}` 意思是 “打印 -3”。

使用这个语法，我们可以修改我们的模板来摆脱这些新行：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello World"
  drink: {{.Values.favorite.drink | default "tea" | quote}}
  food: {{.Values.favorite.food | upper | quote}}
  {{- if eq .Values.favorite.drink "coffee"}}
  mug: true
  {{- end}}
```

为了清楚说明这一点，让我们调整上面的内容，将空格替换为 `*`, 按照此规则将每个空格将被删除。一个在该行的末尾的 `*` 指示换行符将被移除


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello World"
  drink: {{.Values.favorite.drink | default "tea" | quote}}
  food: {{.Values.favorite.food | upper | quote}}*
**{{- if eq .Values.favorite.drink "coffee"}}
  mug: true*
**{{- end}}

```

牢记这一点，我们可以通过 Helm 运行我们的模板并查看结果：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: clunky-cat-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: true
```

小心使用 chomping 修饰符。这样很容易引起意外：


```yaml
  food: {{.Values.favorite.food | upper | quote}}
  {{- if eq .Values.favorite.drink "coffee" -}}
  mug: true
  {{- end -}}

```

这将会产生 food: "PIZZA"mug:true，因为删除了双方的换行符。

> 有关模板中空格控制的详细信息，请参阅官方 Go 模板文档 [Official Go template documentation](https://godoc.org/text/template)

最后，有时候告诉模板系统如何缩进更容易，而不是试图掌握模板指令的间距。因此，有时可能会发现使用 `indent` 函数（`{{indent 2 "mug:true"}}`）会很有用。

## 使用 with 修改范围

下一个要看的控制结构是 `with`。它控制着变量作用域。回想一下，`.` 是对当前范围的引用。因此，`.Values` 告诉模板在当前范围中查找 `Values` 对象。

其语法 with 类似于一个简单的 if 语句：

```
{{with PIPELINE}}
  # restricted scope
{{end}}
```
范围可以改变。with 可以允许将当前范围（`.`）设置为特定的对象。例如，我们一直在使用的 `.Values.favorites`。让我们重写我们的 ConfigMap 来改变 `.` 范围来指向 `.Values.favorites`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite}}
  drink: {{.drink | default "tea" | quote}}
  food: {{.food | upper | quote}}
  {{- end}}
```


注意，现在我们可以引用 `.drink` 和 `.food` 无需对其进行限定。这是因为该 `with` 声明设置 `.` 为指向 `.Values.favorite`。在 `{{end}}` 后 `.` 复位其先前的范围。

但是请注意！在受限范围内，此时将无法从父范围访问其他对象。例如，下面会报错：

```yaml
  {{- with .Values.favorite}}
  drink: {{.drink | default "tea" | quote}}
  food: {{.food | upper | quote}}
  release: {{.Release.Name}}
  {{- end}}
```

它会产生一个错误，因为 Release.Name 它不在 `.` 限制范围内。但是，如果我们交换最后两行，所有将按预期工作，因为范围在 {{end}} 之后被重置。

```yaml
  {{- with .Values.favorite}}
  drink: {{.drink | default "tea" | quote}}
  food: {{.food | upper | quote}}
  {{- end}}
  release: {{.Release.Name}}
```

看下 `range`，我们看看模板变量，它提供了一个解决上述范围问题的方法。

## 循环 `range` 动作

许多编程语言都支持使用 `for` 循环，`foreach` 循环或类似的功能机制进行循环。在 Helm 的模板语言中，遍历集合的方式是使用 `range` 操作子。

首先，让我们在我们的 `values.yaml` 文件中添加一份披萨配料列表：

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

现在我们有一个列表（模板中称为 slice）pizzaToppings。我们可以修改我们的模板，将这个列表打印到我们的 ConfigMap 中：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite}}
  drink: {{.drink | default "tea" | quote}}
  food: {{.food | upper | quote}}
  {{- end}}
  toppings: |-
    {{- range .Values.pizzaToppings}}
    - {{. | title | quote}}
    {{- end}}

```
让我们仔细看看 `toppings`:list。该 range 函数将遍历 pizzaToppings 列表。但现在发生了一些有趣的事. 就像 `with`sets 的范围 `.`，`range` 操作子也是一样。每次通过循环时，`.` 都设置为当前比萨饼顶部。也就是第一次 `.` 设定 mushrooms。第二个迭代它设置为 `cheese`，依此类推。

我们可以直接向管道发送 `.` 的值，所以当我们这样做时 `{{. | title | quote}}`，它会发送 `.` 到 title（标题 case 函数），然后发送到 `quote`。如果我们运行这个模板，输出将是：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-dragonfly-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  toppings: |-
    - "Mushrooms"
    - "Cheese"
    - "Peppers"
    - "Onions"
```

现在，在这个例子中，我们碰到了一些棘手的事情。该 `toppings: |-` 行声明了一个多行字符串。所以我们的 toppings list 实际上不是 YAML 清单。这是一个很大的字符串。我们为什么要这样做？因为 ConfigMaps 中的数据 `data` 由键 / 值对组成，其中键和值都是简单的字符串。要理解这种情况，请查看 [Kubernetes ConfigMap 文档](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/).。但对我们来说，这个细节并不重要。

> YAML 中的 `|-` 标记表示一个多行字符串。这可以是一种有用的技术，用于在清单中嵌入大块数据，如此处所示。

有时能快速在模板中创建一个列表，然后遍历该列表是很有用的。Helm 模板有一个功能可以使这个变得简单：`tuple`。在计算机科学中，元组是类固定大小的列表类集合，但是具有任意数据类型。这粗略地表达了 tuple 的使用方式。

```yaml
  sizes: |-
    {{- range tuple "small" "medium" "large"}}
    - {{.}}
    {{- end}}
```

```yaml
  sizes: |-
    - small
    - medium
    - large
```
除了list和tuple之外，`range`还可以用于遍历具有键和值的集合（如`map` 或 `dict`）。当在下一节我们介绍模板变量时，将看到如何做到这一点。
