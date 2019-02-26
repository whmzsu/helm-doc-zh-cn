# Hooks

Helm 提供了一个 hook 机制，允许 chart 开发人员在 release 的生命周期中的某些点进行干预。例如，可以使用 hooks 来：

- 在加载任何其他 chart 之前，在安装过程中加载 ConfigMap 或 Secret。
- 在安装新 chart 之前执行作业以备份数据库，然后在升级后执行第二个作业以恢复数据。
- 在删除 release 之前运行作业，以便在删除 release 之前优雅地停止服务。

Hooks 像常规模板一样工作，但它们具有特殊的注释，可以使 Helm 以不同的方式使用它们。在本节中，我们介绍 Hooks 的基本使用模式。

## 可用的 Hooks

定义了以下 hooks：

- 预安装 pre-install:：在模板渲染后执行，但在 Kubernetes 中创建任何资源之前执行。
- 安装后 post-install：在所有资源加载到 Kubernetes 后执行
- 预删除 pre-delete：在从 Kubernetes 删除任何资源之前执行删除请求。
- 删除后 post-delete：删除所有 release 的资源后执行删除请求。
- 升级前 pre-upgrade：在模板渲染后，但在任何资源加载到 Kubernetes 之前执行升级请求（例如，在 Kubernetes 应用操作之前）。
- 升级后 post-upgrade：在所有资源升级后执行升级。
- 预回滚 pre-rollback：在渲染模板之后，但在任何资源已回滚之前，在回滚请求上执行。
- 回滚后 post-rollback：在修改所有资源后执行回滚请求。

## Hooks 和 release 的生命周期

Hooks 让 chart 开发人员有机会在 release 的生命周期中的关键点执行操作。例如，考虑 a 的生命周期。默认情况下，生命周期如下所示：

- 用户运行 `helm install foo`
- chart 被加载到 Tiller 中
- 经过一些验证后，Tiller 渲染 foo 模板
- Tiller 将产生的资源加载到 Kubernetes 中
- Tiller 将 release 名称（和其他数据）返回给客户端
- 客户端退出

Helm 为 install 生命周期定义了两个 hook：`pre-install` 和 `post-install`。如果 `foo` chart 的开发者实现了两个 hook，那么生命周期就像这样改变：

- 用户运行 `helm install foo`
- chart 被加载到 Tiller 中
- 经过一些验证后，Tiller 渲染 `foo` 模板
- Tiller 准备执行 `pre-install` hook（将 hook 资源加载到 Kubernetes 中）
- Tiller 会根据权重对 hook 进行排序（默认分配权重 0），并按相同权重的 hook 按升序排序。
- Tiller 然后装载最低权重的 hook（从负到正）
- Tiller 等待，直到 hook“准备就绪”
- Tiller 将产生的资源加载到 Kubernetes 中。请注意，如果设置 `--wait` 标志，Tiller 将等待，直到所有资源都处于就绪状态，并且在准备就绪之前不会运行 `post-install`hook。
- Tiller 执行 `post-install` hook（加载 hook 资源）
- Tiller 等待，直到 hook“准备就绪”
- Tiller 将 release 名称（和其他数据）返回给客户端
- 客户端退出

等到 hook 准备就绪是什么意思？这取决于在 hook 中声明的资源。如果资源是 Job 者一种资源，Tiller 将等到作业成功完成。如果作业失败，则发布失败。这是一个阻塞操作，所以 Helm 客户端会在 Job 运行时暂停。

对于所有其他类型，只要 Kubernetes 将资源标记为加载（添加或更新），资源就被视为 “就绪”。当一个 hook 声明了很多资源时，这些资源将被串行执行。如果他们有 hook 权重（见下文），他们按照加权顺序执行。否则，订购过程不能保证。（在 Helm 2.3.0 及之后的版本中，它们按字母顺序排列，但这种行为并未被视为具有约束力，将来可能会发生变化）。添加挂钩权重被认为是很好的做法，并将其设置为 0 如果权重不是重要。

### Hook 资源不与相应的 release 一起进行管理
Hook 创建的资源不作为 release 的一部分进行跟踪或管理。一旦 Tiller 验证 hook 已经达到其就绪状态，它将 hook 资源放在一边。

实际上，这意味着如果在 hook 中创建资源，则不能依赖于 helm delete 删除资源。要销毁这些资源，需要编写代码在 `pre-delete` 或 `post-delete`hook 中执行此操作，或者将 `"helm.sh/hook-delete-policy"` 注释添加到 hook 模板文件。

## 写一个 hook
Hook 只是 Kubernetes manifest 文件，在 metadata 部分有特殊的注释 。因为他们是模板文件，可以使用所有的 Normal 模板的功能，包括读取 `.Values`，`.Release` 和 `.Template`。

例如，在此模板中, 存储在 `templates/post-install-job.yaml` 的声明要在 `post-install` 阶段运行作业：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{.Release.Name}}"
  labels:
    app.kubernetes.io/managed-by: {{.Release.Service | quote}}
    app.kubernetes.io/instance: {{.Release.Name | quote}}
    helm.sh/chart: "{{.Chart.Name}}-{{.Chart.Version}}"
  annotations:
    # This is what defines this resource as a hook. Without this line, the
    # job is considered part of the release.
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    metadata:
      name: "{{.Release.Name}}"
      labels:
      app.kubernetes.io/managed-by: {{.Release.Service | quote}}
      app.kubernetes.io/instance: {{.Release.Name | quote}}
      helm.sh/chart: "{{.Chart.Name}}-{{.Chart.Version}}"
    spec:
      restartPolicy: Never
      containers:
      - name: post-install-job
        image: "alpine:3.3"
        command: ["/bin/sleep","{{default"10".Values.sleepyTime}}"]

```

注释使这个模板成为 hook：


```
  annotations:
    "helm.sh/hook": post-install
```

一个资源可以部署多个 hook：

```
  annotations:
    "helm.sh/hook": post-install,post-upgrade
```

同样，实现一个给定的 hook 的不同种类资源数量没有限制。例如，我们可以将 secret 和 config map 声明为预安装 hook。

子 chart 声明 hook 时，也会评估这些 hook。顶级 chart 无法禁用子 chart 所声明的 hook。

可以为一个 hook 定义一个权重，这将有助于建立一个确定性的执行顺序。权重使用以下注释来定义：

```
  annotations:
    "helm.sh/hook-weight": "5"
```

hook 权重可以是正数或负数，但必须表示为字符串。当 Tiller 开始执行一个特定类型的 hook (例： `pre-install` hooks  `post-install` hooks, 等等) 执行周期时，它会按升序对这些 hook 进行排序。

还可以定义确定何时删除相应的 hook 资源的策略。hook 删除策略使用以下注释来定义：

```
  annotations:
    "helm.sh/hook-delete-policy": hook-succeeded
```

可以选择一个或多个定义的注释值：

* "hook-succeeded" 指定 Tiller 应该在 hook 成功执行后删除 hook。
* "hook-failed" 指定如果 hook 在执行期间失败，Tiller 应该删除 hook。
* "before-hook-creation" 指定 Tiller 应在删除新 hook 之前删除以前的 hook。

### 自动删除以前版本的 hook
当 helm 的 release 更新, 如果这个 release 使用了 hook，有可能 hook 资源已经存在于群集中。默认情况下，helm 会尝试创建资源，并抛出错误 "... already exists"。

Hook 资源可能已经存在的一个常见原因是，在以前的安装 / 升级中使用它之后它没有被删除。事实上，有充分理由说明为什么人们可能想要保留 hook：例如，在出现问题时帮助手动调试。在这种情况下，确保后续尝试创建 hook 的推​​荐方法不会失败是定义一个 `"hook-delete-policy"`，它可以处理这个：`"helm.sh/hook-delete-policy": "before-hook-creation"`。在安装新 hook 之前，此 hook 注释会导致删除任何现有挂钩。

如果偏好在每次使用后删除钩子（而不是在后续使用时处理它，如上所示）
我们可以选择使用 `"helm.sh/hook-delete-policy": "hook-succeeded,hook-failed"` ：
