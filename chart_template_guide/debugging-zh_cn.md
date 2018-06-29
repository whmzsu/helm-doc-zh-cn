# 调试模板

调试模板可能会很棘手，因为模板在 Tiller 服务器而不是 Helm 客户端上渲染。然后渲染的模板被发送到 Kubernetes API 服务器，可能由于格式以外的原因，服务器可能会拒绝接收这些 YAML 文件。

有几个命令可以帮助您进行调试。

- `helm lint` 是验证 chart 是否遵循最佳实践的首选工具
- `helm install --dry-run --debug`：我们已经知道了这个窍门。这是让服务器渲染你的模板，然后返回结果清单文件的好方法。
- `helm get manifest`：这是查看服务器上安装的模板的好方法。

当你的 YAML 没有解析，但想看看生成了什么时，检索 YAML 的一个简单方法是注释模板中的问题部分，然后重新运行 `helm install --dry-run --debug`：


```yaml
apiVersion: v1
# some: problem section
# {{ .Values.foo | quote }}
```

以上内容将被完整渲染并返回。

```yaml
apiVersion: v1
# some: problem section
#  "bar"
```

这提供了一种快速查看生成的容的方式，而不会由于YAML分析错误而被阻止。
