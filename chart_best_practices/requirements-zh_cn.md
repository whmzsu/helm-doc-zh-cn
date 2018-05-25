# Requirements文件

本指南的这一部分介绍了`requirements.yaml`文件的最佳实践。

## 版本
在可能的情况下，使用版本范围，而不是固定到确切版本。建议的默认值是使用补丁级别的版本匹配：

```yaml
version: ~1.2.3
```

这将匹配版本`1.2.3`和该版本的任何补丁。换句话说，`~1.2.3`相当于`>= 1.2.3, < 1.3.0`

有关完整的版本匹配语法，请参阅[semver documentation](https://github.com/Masterminds/semver#checking-version-constraints)

### 存储库URL

如有可能，请使用`https://存`储库URL，然后使用`http://URL`。

如果存储库已添加到存储库索引文件，则存储库名称可用作URL的别名。使用`alias:`或`@`跟随存储库名称。

文件URL（`file://...`）被视为对于由固定部署管道组装的chart“特殊情况”。正式Helm库中是不允许在一个r`equirements.yaml`使用`file://`的。

## 条件和标签

条件或标签应添加到任何可选的依赖项中。

条件的优选形式是：


```yaml
condition: somechart.enabled
```

`somechart`是依赖的chart名称

当多个子chart（依赖关系）一起提供可选或可交换功能时，这些图应共享相同的标签。

例如，如果nginx和memcached在一起，共同提供性能优化，给chart中的主应用程序，并要求已启用该功能时两者都存在，那么他们可能有这样的标记：

```
tags:
  - webaccelerator
```

这允许用户使用一个标签打开和关闭该功能。
