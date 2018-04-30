# Hooks

Helm提供了一个hook机制，允许chart开发人员在release的生命周期中的某些点进行干预。例如，可以使用hooks来：

- 在加载任何其他chart之前，在安装过程中加载ConfigMap或Secret。
- 在安装新chart之前执行作业以备份数据库，然后在升级后执行第二个作业以恢复数据。
- 在删除release之前运行作业，以便在删除release之前优雅地停止服务。

Hooks像常规模板一样工作，但它们具有特殊的注释，可以使Helm以不同的方式使用它们。在本节中，我们介绍Hooks的基本使用模式。

## 可用的Hooks

定义了以下hooks：

- 预安装pre-install:：在模板渲染后执行，但在Kubernetes中创建任何资源之前执行。
- 安装后post-install：在所有资源加载到Kubernetes后执行
- 预删除pre-delete：在从Kubernetes删除任何资源之前执行删除请求。
- 删除后post-delete：删除所有release的资源后执行删除请求。
- 升级前pre-upgrade：在模板渲染后，但在任何资源加载到Kubernetes之前执行升级请求（例如，在Kubernetes应用操作之前）。
- 升级后post-upgrade：在所有资源升级后执行升级。
- 预回滚pre-rollback：在渲染模板之后，但在任何资源已回滚之前，在回滚请求上执行。
- 回滚后post-rollback：在修改所有资源后执行回滚请求。

## Hooks和release的生命周期

Hooks让chart开发人员有机会在release的生命周期中的关键点执行操作。例如，考虑a的生命周期。默认情况下，生命周期如下所示：

- 用户运行 `helm install foo`
- chart被加载到Tiller中
- 经过一些验证后，Tiller渲染foo模板
- Tiller将产生的资源加载到Kubernetes中
- Tiller将release名称（和其他数据）返回给客户端
- 客户端退出

Helm为install生命周期定义了两个hook：`pre-install`和 `post-install`。如果`foo` chart的开发者实现了两个hook，那么生命周期就像这样改变：

- 用户运行 `helm install foo`
- chart被加载到Tiller中
- 经过一些验证后，Tiller渲染`foo`模板
- Tiller准备执行`pre-install` hook（将hook资源加载到Kubernetes中）
- Tiller会根据权重对hook进行排序（默认分配权重0），并按相同权重的hook按升序排序。
- Tiller然后装载最低权重的hook（从负到正）
- Tiller等待，直到hook“准备就绪”
- Tiller将产生的资源加载到Kubernetes中。请注意，如果设置`--wait`标志，Tiller将等待，直到所有资源都处于就绪状态，并且在准备就绪之前不会运行`post-install`hook。
- Tiller执行`post-install` hook（加载hook资源）
- Tiller等待，直到hook“准备就绪”
- Tiller将release名称（和其他数据）返回给客户端
- 客户端退出

等到hook准备就绪是什么意思？这取决于在hook中声明的资源。如果资源是Job者一种资源，Tiller将等到作业成功完成。如果作业失败，则发布失败。这是一个阻塞操作，所以Helm客户端会在Job运行时暂停。

对于所有其他类型，只要Kubernetes将资源标记为加载（添加或更新），资源就被视为“就绪”。当一个hook声明了很多资源时，这些资源将被串行执行。如果他们有hook权重（见下文），他们按照加权顺序执行。否则，订购过程不能保证。（在Helm 2.3.0及之后的版本中，它们按字母顺序排列，但这种行为并未被视为具有约束力，将来可能会发生变化）。添加挂钩权重被认为是很好的做法，并将其设置为0如果权重不是重要。

### Hook资源不与相应的release一起进行管理
Hook创建的资源不作为release的一部分进行跟踪或管理。一旦Tiller验证hook已经达到其就绪状态，它将hook资源放在一边。

实际上，这意味着如果在hook中创建资源，则不能依赖于helm delete删除资源。要销毁这些资源，需要编写代码在`pre-delete` 或`post-delete`hook中执行此操作，或者将`"helm.sh/hook-delete-policy"`注释添加到hook模板文件。

## 写一个hook
Hook只是Kubernetes manifest文件，在metadata部分有特殊的注释 。因为他们是模板文件，可以使用所有的Normal模板的功能，包括读取`.Values`，`.Release`和`.Template`。

例如，在此模板中,存储在`templates/post-install-job.yaml`的声明要在`post-install`阶段运行作业：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{.Release.Name}}"
  labels:
    heritage: {{.Release.Service | quote }}
    release: {{.Release.Name | quote }}
    chart: "{{.Chart.Name}}-{{.Chart.Version}}"
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
        heritage: {{.Release.Service | quote }}
        release: {{.Release.Name | quote }}
        chart: "{{.Chart.Name}}-{{.Chart.Version}}"
    spec:
      restartPolicy: Never
      containers:
      - name: post-install-job
        image: "alpine:3.3"
        command: ["/bin/sleep","{{default "10" .Values.sleepyTime}}"]

```

注释使这个模板成为hook：


```
  annotations:
    "helm.sh/hook": post-install
```

一个资源可以部署多个hook：

```
  annotations:
    "helm.sh/hook": post-install,post-upgrade
```

同样，实现一个给定的hook的不同种类资源数量没有限制。例如，我们可以将secret和config map声明为预安装hook。

子chart声明hook时，也会评估这些hook。顶级chart无法禁用子chart所声明的hook。

可以为一个hook定义一个权重，这将有助于建立一个确定性的执行顺序。权重使用以下注释来定义：

```
  annotations:
    "helm.sh/hook-weight": "5"
```

hook权重可以是正数或负数，但必须表示为字符串。当Tiller开始执行一个特定类型的hook的执行周期时，它会按升序对这些hook进行排序。

还可以定义确定何时删除相应的hook资源的策略。hook删除策略使用以下注释来定义：

```
  annotations:
    "helm.sh/hook-delete-policy": hook-succeeded
```

可以选择一个或多个定义的注释值：

* "hook-succeeded" 指定Tiller应该在hook成功执行后删除hook。
* "hook-failed" 指定如果hook在执行期间失败，Tiller应该删除hook。
* "before-hook-creation" 指定Tiller应在删除新hook之前删除以前的hook。

### 自动删除以前版本的hook
当helm 的release更新时，有可能hook资源已经存在于群集中。默认情况下，helm会尝试创建资源，并抛出错误"... already exists"。

我们可以选择`"helm.sh/hook-delete-policy": "before-hook-creation"`，取代 `"helm.sh/hook-delete-policy": "hook-succeeded,hook-failed"`因为：

* 例如为了手动调试，将错误的hook作业资源保存在kubernetes中是很方便的。
* 出于某种原因，可能有必要将成功的hook资源保留在kubernetes中。
* 同时，在helm release升级之前进行手动资源删除是不可取的。

`"helm.sh/hook-delete-policy": "before-hook-creation"` 在hook中的注释，如果在新的hook启动前有一个hook的话，会使Tiller将以前的release中的hook删除，，而这个hook同时它可能正在被其他一个策略使用。
