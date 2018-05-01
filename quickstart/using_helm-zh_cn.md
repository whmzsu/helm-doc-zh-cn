# 使用
本指南讲述使用Helm（和Tiller）来管理Kubernetes群集上的软件包的基础知识。前提是假定你已经安装了Helm客户端和Tiller服务端（通常通过helm init）。

如果只是想运行一些简单命令，可以从[快速入门指南](quickstart-zh_cn.md)开始。本章将介绍Helm命令的具体内容，并解释如何使用Helm。

## 三大概念

A *Chart* 是一个Helm包。它包含在Kubernetes集群内部运行应用程序，工具或服务所需的所有资源定义。把它想像为一个自制软件，一个Apt dpkg或一个Yum RPM文件的Kubernetes环境里面的等价物。

A *Repository* 是Charts收集和共享的地方。它就像Perl的[CPAN archive](http://www.cpan.org)或Fedora软件包repo[Fedora Package Database](https://admin.fedoraproject.org/pkgdb/)。

A *Release* 是处于Kubernetes集群中运行的Chart的一个实例。一个chart通常可以多次安装到同一个群集中。每次安装时，都会创建一个新 _release_ 。比如像一个MySQL chart。如果希望在群集中运行两个数据库，则可以安装该chart两次。每个都有自己的 _release_，每个 _release_ 都有自己的 _release name_。

有了这些概念，我们现在可以这样解释Helm：

Helm将 _charts_ 安装到Kubernetes中，每个安装创建一个新 _release_ 。要找到新的chart，可以搜索Helm charts 存储库 _repositories_。

## 'helm search':查找Charts

首次安装Helm时，它已预配置为与官方Kubernetes chart 存储库repo。该repo包含许多精心策划和维护的charts。此charts repo默认以stable命名。

可以通过运行`helm search`查看哪些charts可用：

```
$ helm search
NAME                 	VERSION 	DESCRIPTION
stable/drupal   	0.3.2   	One of the most versatile open source content m...
stable/jenkins  	0.1.0   	A Jenkins Helm chart for Kubernetes.
stable/mariadb  	0.5.1   	Chart for MariaDB
stable/mysql    	0.1.0   	Chart for MySQL
...
```

如果没有使用过滤，helm search显示所有可用的charts。可以通过使用过滤器进行搜索来缩小搜索结果范围：

```
$ helm search mysql
NAME               	VERSION	DESCRIPTION
stable/mysql  	0.1.0  	Chart for MySQL
stable/mariadb	0.5.1  	Chart for MariaDB
```
现在只会看到过滤器匹配的结果。

为什么`mariadb`在列表中？因为它的包描述与MySQL相关。我们可以使用`helm inspect chart`到这个：

```
$ helm inspect stable/mariadb
Fetched stable/mariadb to mariadb-0.5.1.tgz
description: Chart for MariaDB
engine: gotpl
home: https://mariadb.org
keywords:
- mariadb
- mysql
- database
- sql
...
```

搜索是找到可用软件包的好方法。一旦找到想要安装的软件包，可以使用`helm install`它来安装它。

## 'helm install'：安装一个软件包
要安装新的软件包，请使用该`helm install`命令。最简单的方法，它只需要一个参数：chart的名称。

```
$ helm install stable/mariadb
Fetched stable/mariadb-0.3.0 to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
happy-panda
Last Deployed: Wed Sep 28 12:32:28 2016
Namespace: default
Status: DEPLOYED

Resources:
==> extensions/Deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
happy-panda-mariadb   1         0         0            0           1s

==> v1/Secret
NAME                     TYPE      DATA      AGE
happy-panda-mariadb   Opaque    2         1s

==> v1/Service
NAME                     CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
happy-panda-mariadb   10.0.0.70    <none>        3306/TCP   1s


Notes:
MariaDB can be accessed via port 3306 on the following DNS name from within your cluster:
happy-panda-mariadb.default.svc.cluster.local

To connect to your database run the following command:

   kubectl run happy-panda-mariadb-client --rm --tty -i --image bitnami/mariadb --command -- mysql -h happy-panda-mariadb
```

当mariadb chart已安装，请注意，安装chart会创建一个新 _release_ 对象。上面的release被命名 为`happy-panda`。（如果你想使用你自己的release名称，只需使用 --name 参数 配合helm install。）

在安装过程中，`helm`客户端将打印有关创建哪些资源的有用信息，release的状态以及是否可以或应该采取其他的配置步骤。

Helm不会一直等到所有资源都运行才退出。许多charts需要大小超过600M的Docker 镜像，并且可能需要很长时间才能安装到群集中。

要跟踪release状态或重新读取配置信息，可以使用`helm status`：

```
$ helm status happy-panda
Last Deployed: Wed Sep 28 12:32:28 2016
Namespace: default
Status: DEPLOYED

Resources:
==> v1/Service
NAME                     CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
happy-panda-mariadb   10.0.0.70    <none>        3306/TCP   4m

==> extensions/Deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
happy-panda-mariadb   1         1         1            1           4m

==> v1/Secret
NAME                     TYPE      DATA      AGE
happy-panda-mariadb   Opaque    2         4m


Notes:
MariaDB can be accessed via port 3306 on the following DNS name from within your cluster:
happy-panda-mariadb.default.svc.cluster.local

To connect to your database run the following command:

   kubectl run happy-panda-mariadb-client --rm --tty -i --image bitnami/mariadb --command -- mysql -h happy-panda-mariadb
```

以上显示了release的当前状态。

### 在安装前自定义chart

上面的安装方式使用chart的默认配置选项。很多时候，我们需要自定义chart以使用自定义配置。

要查看chart上可配置的选项，请使用`helm inspect values`：

```bash
helm inspect values stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
## Bitnami MariaDB image version
## ref: https://hub.docker.com/r/bitnami/mariadb/tags/
##
## Default: none
imageTag: 10.1.14-r3

## Specify a imagePullPolicy
## Default to 'Always' if imageTag is 'latest', else set to 'IfNotPresent'
## ref: http://kubernetes.io/docs/user-guide/images/#pre-pulling-images
##
# imagePullPolicy:

## Specify password for root user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#setting-the-root-password-on-first-run
##
# mariadbRootPassword:

## Create a database user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-user-on-first-run
##
# mariadbUser:
# mariadbPassword:

## Create a database
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-on-first-run
##
# mariadbDatabase:
```

然后，可以在YAML格式的文件中覆盖任何这些设置，然后在安装过程中传递该文件。

```bash
$ echo '{mariadbUser: user0, mariadbDatabase: user0db}' > config.yaml
$ helm install -f config.yaml stable/mariadb
```

以上将创建一个名称为MariaDB的默认用户`user0`，并授予此用户对新创建`user0db`数据库的访问权限，其他使用该chart的默认值。

在安装过程中有两种方式传递自定义配置数据：

- --values（或-f）：指定一个overrides的YAML文件。可以指定多次，最右边的文件将优先使用
- --set：在命令行上指定overrides。

如果两者都使用，则将`--set`值合并到`--values`更高的优先级中。指定的override `--set`将保存在configmap中。`--set`可以通过使用特定的版本查看已经存在的值 `helm get values <release-name>`,`--set`设置的值可以通过运行helm upgrade与--reset-values 重置。

#### `--set`格式和限制

`--set`选项使用零个或多个name/value对。最简单的用法：--set name=value。YAML体现是：

```yaml
name: value
```

多个值由,字符分隔。因此--set a=b,c=d变成：

```yaml
a: b
c: d
```

支持更复杂的表达式。例如，--set outer.inner=value变成这样：

```yaml
outer:
  inner: value
```

列表可以通过在{和}中包含值来表示。例如， --set name={a, b, c}转化为：

```yaml
name:
  - a
  - b
  - c
```

从Helm 2.5.0开始，可以使用数组索引语法访问列表项。例如，--set servers[0].port=80变成：

```yaml
servers:
  - port: 80
```

可以通过这种方式设置多个值。该行--set servers[0].port=80,servers[0].host=example变成：

```yaml
servers:
  - port: 80
    host: example
```

有时候你需要在`--set`行中使用特殊字符。可以使用反斜杠来转义字符; --set name=value1\,value2会变成：

```yaml
name: "value1,value2"
```

同样，也可以转义点序列，这可能在chart中使用`toYaml`函数解析注释，标签和节点选择器时派上用场 。--set nodeSelector."kubernetes\.io/role"=master变为的语法 ：

```yaml
nodeSelector:
  kubernetes.io/role: master
```

使用深层嵌套的数据结构可能很难用`--set`表达。鼓励chart设计师在设计values.yaml文件格式时考虑`--set`使用情况。

### 更多的安装方法
helm install命令可以从多个来源安装：

- 一个chart repository (像上面看到的)
- 一个本地 chart 压缩包 (`helm install foo-0.1.1.tgz`)
- 一个解压后的chart目录 (`helm install path/to/foo`)
- 一个完整URL (`helm install https://example.com/charts/foo-1.2.3.tgz`)


## 'helm upgrade' and 'helm rollback'：升级版本和失败时恢复
当新版本的chart发布时，或者当你想要更改release配置时，可以使用`helm upgrade` 命令。

升级需要已有的release并根据提供的信息进行升级。由于Kubernetes chart可能很大而且很复杂，因此Helm会尝试执行最小侵入式升级。它只会更新自上次发布以来发生更改的内容。

```bash
$ helm upgrade -f panda.yaml happy-panda stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/k8s.io/helm/mariadb-0.3.0.tgz
happy-panda has been upgraded. Happy Helming!
Last Deployed: Wed Sep 28 12:47:54 2016
Namespace: default
Status: DEPLOYED
...
```

在上面的例子中，happy-panda release使用相同的chart进行升级，但使用新的YAML文件：

```yaml
mariadbUser: user1
```

我们可以使用`helm get values`看看这个新设置是否生效。

```bash
$ helm get values happy-panda
mariadbUser: user1
```

该helm get命令是查看集群中的release的有用工具。正如我们上面所看到的，它表明我们的新值 panda.yaml已被部署到群集中。

现在，如果在发布过程中某些事情没有按计划进行，那么使用回滚到以前的版本很容易`helm rollback [RELEASE] [REVISION]`。

```bash
$ helm rollback happy-panda 1
```

上述回滚我们的“happy-panda”到它的第一个release版本。release版本是增量修订。每次安装，升级或回滚时，修订版本号都会增加1.第一个修订版本号始终为1.我们可以使用`helm history [RELEASE]`查看特定版本的修订版号。

## 安装/升级/回滚的有用选项
在安装/升级/回滚期间，可以指定几个其他有用的选项来定制Helm的行为。请注意，这不是cli参数的完整列表。要查看所有参数的说明，请运行 helm <command> --help。

- `--timeout`：等待Kubernetes命令完成的超时时间值（秒），默认值为300（5分钟）
- `--wait`：等待所有Pod都处于就绪状态，PVC绑定完，将release标记为成功之前，Deployments有最小（Desired-maxUnavailable）Pod处于就绪状态，并且服务具有IP地址（如果是`LoadBalancer`，则为Ingress ）。它会等待 `--timeout`的值。如果达到超时，release将被标记为 FAILED。注意：在部署replicas设置为1 maxUnavailable且未设置为0，作为滚动更新策略的一部分的情况下， `--wait`它将返回就绪状态，因为它已满足就绪状态下的最小Pod。
- `--no-hooks`：这会跳过命令的运行钩子
- `--recreate-pods`（仅适用于upgrade和rollback）：此参数将导致重新创建所有pod（属于deployment的pod除外）

## 'helm delete'：删除Release

在需要从群集中卸载或删除release时，请使用以下`helm delete`命令：

```
$ helm delete happy-panda
```

这将从集群中删除该release。可以使用以下helm list命令查看当前部署的所有release：

```
$ helm list
NAME           	VERSION	UPDATED                        	STATUS         	CHART
inky-cat       	1      	Wed Sep 28 12:59:46 2016       	DEPLOYED       	alpine-0.1.0
```

从上面的输出中，我们可以看到该happy-panda release已被删除。

尽快如此，Helm总是保留记录发生了什么。需要查看已删除的版本？`helm list --deleted` 可显示这些内容，并`helm list --all`显示了所有release（已删除和当前部署的，以及失败的版本）：


```bash
⇒  helm list --all
NAME           	VERSION	UPDATED                        	STATUS         	CHART
happy-panda   	2      	Wed Sep 28 12:47:54 2016       	DELETED        	mariadb-0.3.0
inky-cat       	1      	Wed Sep 28 12:59:46 2016       	DEPLOYED       	alpine-0.1.0
kindred-angelf 	2      	Tue Sep 27 16:16:10 2016       	DELETED        	alpine-0.1.0
```

由于Helm保留已删除release的记录，因此不能重新使用release名称。（如果 _确实_ 需要重新使用此release名称，则可以使用此 `--replace`参数，但它只会重用现有release并替换其资源。）

请注意，因为release以这种方式保存，所以可以回滚已删除的资源并重新激活它。

## 'helm repo'：使用存储库

到目前为止，我们一直只从stable存储库repo安装chart。但是可以配置helm使用其他repo。Helm在该helm repo命令下提供了多个repo工具。

可以使用helm repo list以下命令查看配置了哪些repo：

```bash
$ helm repo list
NAME           	URL
stable         	https://kubernetes-charts.storage.googleapis.com
local          	http://localhost:8879/charts
mumoshu        	https://mumoshu.github.io/charts
```

新的repo可以通过`helm repo add`添加：

```bash
$ helm repo add dev https://example.com/dev-charts
```

由于chart repo经常更改，因此可以随时通过运行`helm repo updat`确保Helm客户端处于最新状态。

## 创建你自己的charts
该chart开发指南[Chart Development Guide](charts.md) 介绍了如何开发自己的charts。也可以通过使用以下helm create 命令快速入门：

```bash
$ helm create deis-workflow
Creating deis-workflow
```

现在有一个chart`./deis-workflow`。可以编辑它并创建自己的模板。

在编辑chart时，可以通过`helm lint`验证它是否格式正确。

当将chart打包分发时，可以运行以下 helm package命令：

```bash
$ helm package deis-workflow
deis-workflow-0.1.0.tgz
```

现在可以通过`helm install`以下方式轻松安装该chart：

```bash
$ helm install ./deis-workflow-0.1.0.tgz
...
```

可以将已归档的chart加载到chart repo中。请参阅chart repo服务器的文档以了解如何上传。

注意：stable repo在Kubernetes Charts GitHub存储库上进行管理。该项目接受chart源代码，并且（在审计后）自动打包。

## Tiller，Namespaces和RBAC
在某些情况下，可能希望将Tiller的范围或将多个Tillers部署到单个群集。以下是在这些情况下操作的一些最佳做法。

1. Tiller可以安装到任何namespace。默认情况下，它安装在kube-system中。可以运行多个Tillers，只要它们各自在自己的namespace中运行。
2. 限制Tiller只能安装到特定的namespace和/或资源类型由Kubernetes RBAC角色和角色绑定控制。可以通过在配置Helm时通过`helm init --service-account <NAME>`向Tiller添加服务帐户。你可以在这里[here](rbac.md).找到更多的信息。
3. Release名称在每个Tiller实例中是唯一的。
4. chart应该只包含存在于单个命名空间中的资源。
5. 不建议将多个Tillers配置为在相同的命名空间中管理资源。
## 总结

本章介绍了helm客户端的基本使用模式，包括搜索，安装，升级和删除。它也涵盖了有用的工具命令类似如`helm status`，`helm get`和 `helm repo`。

有关这些命令的更多信息，请查看Helm的内置帮助：`helm help`。

在下一章中，我们将看看开发chart的过程。
