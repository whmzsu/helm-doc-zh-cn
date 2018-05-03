# 模板函数和管道

目前为止，我们已经知道如何将信息放入模板中。但是这些信息未经修改就被放入模板中。有时我们想要转换这些数据，使得他们对我们来说更有用。

让我们从一个最佳实践开始：当从.Values对象注入字符串到模板中时，我们引用这些字符串。我们可以通过调用quote模板指令中的函数来实现：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

模板函数遵循语法`functionName arg1 arg2...`。在上面的代码片段中，`quote .Values.favorite.drink`调用quote函数并将一个参数传递给它。

Helm拥有超过60种可用函数。其中一些是由Go模板语言[Go template language](https://godoc.org/text/template) 本身定义的。其他大多数都是Sprig模板库[Sprig template library](https://godoc.org/github.com/Masterminds/sprig)的一部分。在我们讲解例子进行的过程中，我们会看到很多。

> 虽然我们将Helm模板语言视为Helm特有的，但它实际上是Go模板语言，一些额外函数和各种包装器的组合，以将某些对象暴露给模板。Go模板上的许多资源在了解模板时可能会有所帮助。

## 管道

模板语言的强大功能之一是其管道概念。利用UNIX的一个概念，管道是一个链接在一起的一系列模板命令的工具，以紧凑地表达一系列转换。换句话说，管道是按顺序完成几件事情的有效方式。我们用管道重写上面的例子。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | quote }}
```

在这个例子中，没有调用`quote ARGUMENT`，我们调换了顺序。我们使用管道（|）将“参数”发送给函数：`.Values.favorite.drink | quote`。使用管道，我们可以将几个功能链接在一起：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```
> 反转顺序是模板中的常见做法。你会看到.`val | quote`比`quote .val`更常见。练习也是。

当评估时，该模板将产生如下结果：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: trendsetting-p-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

请注意，我们的原来`pizza`现在已经转换为`"PIZZA"`。

当有像这样管道参数时，第一个评估（`.Values.favorite.drink`）的结果将作为函数的最后一个参数发送。我们可以修改上面的饮料示例来说明一个带有两个参数的函数`repeat COUNT STRING`：


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | repeat 5 | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

该repeat函数将回送给定的字符串和给定的次数，所以我们将得到这个输出：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: melting-porcup-configmap
data:
  myvalue: "Hello World"
  drink: "coffeecoffeecoffeecoffeecoffee"
  food: "PIZZA"
```

## 使用default函数

经常使用的一个函数是`default`：`default DEFAULT_VALUE GIVEN_VALUE`。该功能允许在模板内部指定默认值，以防该值被省略。让我们用它来修改上面的饮料示例：

```yaml
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

如果我们像往常一样运行，我们会得到我们的coffee：


```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: virtuous-mink-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

现在，我们将从以下位置删除喜欢的饮料设置values.yaml：

```yaml
favorite:
  #drink: coffee
  food: pizza
```

现在重新运行`helm install --dry-run --debug ./mychart`会产生这个YAML：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fair-worm-configmap
data:
  myvalue: "Hello World"
  drink: "tea"
  food: "PIZZA"
```

在实际的chart中，所有静态默认值应该存在于values.yaml中，不应该使用该default命令重复（否则它们将是重复多余的）。但是，default命令对于计算的值是合适的，因为计算值不能在values.yaml中声明。例如：

```yaml
drink: {{ .Values.favorite.drink | default (printf "%s-tea" (include "fullname" .)) }}
```

在一些地方，一个`if`条件可能比这`default`更适合。我们将在下一节中看到这些。

模板函数和管道是转换信息并将其插入到YAML中的强大方法。但有时候需要添加一些比插入字符串更复杂一些的模板逻辑。在下一节中，我们将看看模板语言提供的控制结构。

## 操作子函数

对于模板，操作子（eq，ne，lt，gt，and，or等等）都是已实现的功能。在管道中，操作子可以用圆括号（`(`和`)`）分组。

现在我们可以从函数和管道转向流控制,条件，循环和范围修饰符。
