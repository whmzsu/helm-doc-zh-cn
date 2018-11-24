# 一般约定

最佳实践指南的这一部分介绍了一般约定。

## Chart 名称

Chart 名称应该是小写字母和数字组成，字母开头：

举例：
可以使用破折号 `-`, 但在 Helm templates 中使用需要一些小技巧 (查看 [issue #2192](https://github.com/helm/helm/issues/2192) 获取更多信息).
+
这里有一些好例子 [Helm Community Charts](https://github.com/helm/charts):

```
drupal
cert-manager
oauth2-proxy
```

Chart 名称中不能使用大写字母和下划线。Chart 名称不应使用点。

包含 chart 的目录必须与 chart 具有相同的名称。因此，chart `cert-manager` 必须在名为 `cert-manager/` 的目录中创建。这不仅仅是一种风格的细节，而是 Helm Chart 格式的要求。

## 版本号

只要有可能，Helm 使用 [SemVer 2](http://semver.org) 来表示版本号。（请注意，Docker 镜像 tag 不一定遵循 SemVer，因此被视为该规则的一个例外。）

当 SemVer 版本存储在 Kubernetes 标签中时，我们通常会将该 `+` 字符更改为一个 `_` 字符，因为标签不允许 `+` 标志作为值。

## 格式化 YAML

YAML 文件应该使用两个空格缩进（而不是制表符）。

## 单词 Helm，Tiller 和 Chart 的用法
使用 Helm，helm，Tiller 和 tiller 这两个词有一些小的惯例。

- Helm 是指该项目，通常用作总括术语
- `helm ` 指的是客户端命令
- Tiller 是后端的专有名称
- `tiller` 是后端二进制运行的名称
- 术语 “chart” 不需要大写，因为它不是专有名词。

如有疑问，请使用 Helm（大写'H'）。

## 通过版本限制 Tiller

一个 `Chart.yaml` 文件可以指定一个 `tillerVersion` SemVer 约束：

```yaml
name: mychart
version: 0.2.0
tillerVersion: ">=2.4.0"
```

当模板使用 Helm 旧版本不支持的新功能时，应该设置此限制。虽然此参数将接受复杂的 SemVer 规则，但最佳做法是默认为格式 >=2.4.0，其中 2.4.0 引入了 chart 中使用的新功能的版本。

此功能是在Helm 2.4.0中引入的，因此任何2.4.0版本以下的Tiller都会忽略此字段。
