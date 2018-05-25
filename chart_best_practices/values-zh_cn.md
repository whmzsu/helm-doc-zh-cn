# Values
这部分最佳实践指南涵盖了values的使用。在指南的这一部分，我们提供关于如何构建和使用values的建议，重点在于设计chart的`values.yaml`文件。

## 命名约定
变量名称应该以小写字母开头，单词应该用camelcase分隔：

正确写法：
```yaml
chicken: true
chickenNoodleSoup: true
```

不正确写法：


```yaml
Chicken: true  # initial caps may conflict with built-ins
chicken-noodle-soup: true # do not use hyphens in the name
```

请注意，Helm的所有内置变量都以大写字母开头，以便将它们与用户定义的value区分开来，如：`.Release.Name`， `.Capabilities.KubeVersion`。

## 展平或嵌套值

YAML是一种灵活的格式，并且值可以嵌套或扁平化。

嵌套：

```yaml
server:
  name: nginx
  port: 80
```

展平：

```yaml
serverName: nginx
serverPort: 80
```

在大多数情况下，展平应该比嵌套更受青睐。原因是对模板开发人员和用户来说更简单。

为了获得最佳安全性，必须在每个级别检查嵌套值：

```
{{ if .Values.server }}
  {{ default "none" .Values.server.name }}
{{ end }}
```

对于每一层嵌套，都必须进行存在检查。但对于展平配置，可以跳过这些检查，使模板更易于阅读和使用。

```
{{ default "none" .Values.serverName }}
```

当有大量相关变量时，且至少有一个是非可选的，可以使用嵌套值来提高可读性。

## 使类型清晰

YAML的类型强制规则有时是违反直觉的。例如， `foo: false`与`foo: "false"`不一样。f`oo: 12345678` 在某些情况下，大整数将被转换为科学记数法。

避免类型转换错误的最简单方法是明确地表示字符串，并隐含其他所有内容。或者，简而言之，引用所有字符串。

通常，为了避免整型转换问题，最好将整型存储为字符串，并在模板中使用`{{ int $value }}`将字符串转换为整数。

在大多数情况下，显式类型标签受到重视，所以`foo: !!string 1234`应该将`1234视`为一个字符串。但是，YAML解析器消费标签，因此类型数据在解析后会丢失。

## 考虑用户如何使用你的values

有三种潜在的values来源：

- chart的`values.yaml`文件
- 由`helm install -f` 或`helm upgrade -f`提供的value文件
- 传递给`--set`或的`--set-string`标志`helm install`或`helm upgrade`命令

在设计value的结构时，请记住chart的用户可能希望通过`-f`标志或`--set `选项覆盖它们。

由于`--set`在表现力方面比较有限，编写`values.yaml`文件的第一个指导原则可以轻松使用`--set`覆盖。

出于这个原因，使用map来构建value文件通常会更好。

难以配合`--set`使用：

```yaml
servers:
  - name: foo
    port: 80
  - name: bar
    port: 81
```

Helm `<=2.4`时，以上不能用`--set `来表示。在Helm 2.5中，访问foo上的端口是`--set servers[0].port=80`。用户不仅难以弄清楚，而且如果稍后servers改变顺序，则容易出错。

使用方便：

```yaml
servers:
  foo:
    port: 80
  bar:
    port: 81
```

访问foo的端口更为方便：`--set servers.foo.port=80`。

## 文档 'values.yaml'

应该记录'values.yaml'中的每个定义的属性。文档字符串应该以它描述的属性的名称开始，然后至少给出一个单句描述。

不正确：

```
# the host name for the webserver
serverHost = example
serverPort = 9191
```

正确：

```
# serverHost is the host name for the webserver
serverHost = example
# serverPort is the HTTP listener port for the webserver
serverPort = 9191
```

使用参数名称开始每个注释，它使文档易于grep，并使文档工具能够可靠地将文档字符串与其描述的参数关联起来。
