# values 文件
在上一节中，我们看了 Helm 模板提供的内置对象。四个内置对象之一是 Values。该对象提供对传入 chart 的值的访问。其内容来自四个来源：

- chart 中的 `values.yaml` 文件
- 如果这是一个子 chart，来自父 chart 的 `values.yaml` 文件
- value 文件通过 helm install 或 helm upgrade 的 - f 标志传入文件（`helm install -f myvals.yaml ./mychart`）
- 通过 `--set`（例如 `helm install --set foo=bar ./mychart`）

上面的列表按照特定的顺序排列：values.yaml 在默认情况下，父级 chart 的可以覆盖该默认级别，而该 chart values.yaml 又可以被用户提供的 values 文件覆盖，而该文件又可以被 --set 参数覆盖。

值文件是纯 YAML 文件。我们编辑 `mychart/values.yaml`，然后来编辑我们的 `ConfigMap` 模板。

删除默认带的 values.yaml，我们只设置一个参数：

```yaml
favoriteDrink: coffee
```

现在我们可以在模板中使用这个：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favoriteDrink }}
```

注意我们在最后一行 \{\{ .Values.favoriteDrink\}\} 获取 `favoriteDrink` 的值。

让我们看看这是如何渲染的。

```bash
$ helm install --dry-run --debug ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   geared-marsupi
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: geared-marsupi-configmap
data:
  myvalue: "Hello World"
  drink: coffee
```

由于 `favoriteDrink` 在默认 `values.yaml` 文件中设置为 `coffee`，这就是模板中显示的值。我们可以轻松地在我们的 helm install 命令中通过加一个 `--set` 添标志来覆盖：

```
helm install --dry-run --debug --set favoriteDrink=slurm ./mychart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/k8s.io/helm/_scratch/mychart
NAME:   solid-vulture
TARGET NAMESPACE:   default
CHART:  mychart 0.1.0
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: solid-vulture-configmap
data:
  myvalue: "Hello World"
  drink: slurm
```

由于 `--set` 比默认 `values.yaml` 文件具有更高的优先级，我们的模板生成 `drink: slurm`。

values 文件也可以包含更多结构化内容。例如，我们在 values.yaml 文件中可以创建 `favorite` 部分，然后在其中添加几个键：

```yaml
favorite:
  drink: coffee
  food: pizza
```

现在我们稍微修改模板：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food }}
```

虽然以这种方式构建数据是可以的，但建议保持 value 树浅一些，平一些。当我们看看为子 chart 分配值时，我们将看到如何使用树结构来命名值。

## 删除默认 key

如果您需要从默认值中删除一个键，可以覆盖该键的值为 null，在这种情况下，Helm 将从覆盖值合并中删除该键。

例如，stable 版本的 Drupal chart 允许配置 liveness 探测器，如果你配置自定义的 image。以下是默认值：

```yaml
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  initialDelaySeconds: 120
```

如果尝试覆盖 liveness Probe 处理程序 `exec` 而不是 `httpGet`，使用 `--set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt]`，Helm 会将默认和重写的键合并在一起，从而产生以下 YAML：

```yaml
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  exec:
    command:
    - cat
    - docroot/CHANGELOG.txt
  initialDelaySeconds: 120
```

但是，Kubernetes 会报错，因为无法声明多个 liveness Probe 处理程序。为了克服这个问题，你可以指示 Helm 过将 livenessProbe.httpGet 通设置为空来删除它：

```bash
helm install stable/drupal --set image=my-registry/drupal:0.1.0 --set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt] --set livenessProbe.httpGet=null
```

到这里，我们已经看到了几个内置对象，并用它们将信息注入到模板中。现在我们来看看模板引擎的另外内容：函数和管道。
