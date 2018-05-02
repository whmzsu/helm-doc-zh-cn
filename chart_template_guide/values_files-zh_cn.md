# values文件
在上一节中，我们看了Helm模板提供的内置对象。四个内置对象之一是Values。该对象提供对传入chart的值的访问。其内容来自四个来源：

- chart中的`values.yaml`文件
- 如果这是一个子chart，来自父chart的`values.yaml`文件
- value文件通过helm install或helm upgrade的-f标志传入文件（`helm install -f myvals.yaml ./mychart`）
- 通过`--set`（例如`helm install --set foo=bar ./mychart`）

上面的列表按照特定的顺序排列：values.yaml在默认情况下，父级chart的可以覆盖该默认级别，而该chart values.yaml又可以被用户提供的values文件覆盖，而该文件又可以被--set参数覆盖。

值文件是纯YAML文件。我们编辑`mychart/values.yaml`，然后来编辑我们的`ConfigMap`模板。

删除默认带的values.yaml，我们只设置一个参数：

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

注意我们在最后一行\{\{ .Values.favoriteDrink\}\}获取`favoriteDrink`的值。

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

由于`favoriteDrink`在默认`values.yaml`文件中设置为`coffee`，这就是模板中显示的值。我们可以轻松地在我们的helm install命令中通过加一个`--set`添标志来覆盖：

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

由于`--set`比默认`values.yaml`文件具有更高的优先级，我们的模板生成`drink: slurm`。

values文件也可以包含更多结构化内容。例如，我们在values.yaml文件中可以创建`favorite`部分，然后在其中添加几个键：

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

虽然以这种方式构建数据是可以的，但建议保持value树浅一些，平一些。当我们看看为子chart分配值时，我们将看到如何使用树结构来命名值。

## 删除默认key

如果您需要从默认值中删除一个键，可以覆盖该键的值为null，在这种情况下，Helm将从覆盖值合并中删除该键。

例如，stable版本的Drupal chart允许配置liveness探测器，如果你配置自定义的image。以下是默认值：

```yaml
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  initialDelaySeconds: 120
```

如果尝试覆盖liveness Probe处理程序`exec`而不是`httpGet`，使用`--set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt]`，Helm会将默认和重写的键合并在一起，从而产生以下YAML：

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

但是，Kubernetes会报错，因为无法声明多个liveness Probe处理程序。为了克服这个问题，你可以指示Helm 过将livenessProbe.httpGet通设置为空来删除它：

```bash
helm install stable/drupal --set image=my-registry/drupal:0.1.0 --set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt] --set livenessProbe.httpGet=null
```

到这里，我们已经看到了几个内置对象，并用它们将信息注入到模板中。现在我们来看看模板引擎的另外内容：函数和管道。
