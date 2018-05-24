# 一般约定

最佳实践指南的这一部分介绍了一般约定。

## Chart名称

Chart名称应该是小写字母和数字组成，可以用破折号（-）分隔：

举例：

```
drupal
nginx-lego
aws-cluster-autoscaler
```

Chart名称中不能使用大写字母和下划线。Chart名称不应使用点。

包含chart的目录必须与chart具有相同的名称。因此，chart `nginx-lego`必须在名为`nginx-lego/`的目录中创建。这不仅仅是一种风格的细节，而是Helm Chart格式的要求。

## 版本号

只要有可能，Helm使用[SemVer 2](http://semver.org) 来表示版本号。（请注意，Docker镜像tag不一定遵循SemVer，因此被视为该规则的一个例外。）

当SemVer版本存储在Kubernetes标签中时，我们通常会将该`+`字符更改为一个`_`字符，因为标签不允许`+`标志作为值。

## 格式化YAML

YAML文件应该使用两个空格缩进（而不是制表符）。

## 单词Helm，Tiller和Chart的用法
使用Helm，helm，Tiller和tiller这两个词有一些小的惯例。

- Helm是指该项目，通常用作总括术语
- `helm `指的是客户端命令
- Tiller是后端的专有名称
- `tiller` 是后端二进制运行的名称
- 术语“chart”不需要大写，因为它不是专有名词。

如有疑问，请使用Helm（大写'H'）。

## 通过版本限制Tiller

一个`Chart.yaml`文件可以指定一个`tillerVersion` SemVer 约束：

```yaml
name: mychart
version: 0.2.0
tillerVersion: ">=2.4.0"
```

当模板使用Helm旧版本不支持的新功能时，应该设置此限制。虽然此参数将接受复杂的SemVer规则，但最佳做法是默认为格式>=2.4.0，其中2.4.0 引入了chart中使用的新功能的版本。

此功能是在Helm 2.4.0中引入的，因此任何2.4.0版本以下的Tiller都会忽略此字段。
