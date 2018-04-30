# Charts
Helm使用称为chart的包装格式。chart是描述相关的一组Kubernetes资源的文件集合。单个chart可能用于部署简单的东西，比如memcached pod，或者一些复杂的东西，比如完整的具有HTTP服务，数据库，缓存等的Web应用程序堆栈。

chart通过创建为特定目录树的文件，将它们打包到版本化的压缩包，然后进行部署。

本文档解释了chart格式，提供使用Helm构建chart的基本指导。

## Chart文件结构

chart被组织为一个目录内的文件集合。目录名称是chart的名称（没有版本信息）。例如，描述WordPress的chart将被存储在wordpress/目录中。

在这个目录里面，Helm期望如下这样一个的结构的目录树：

```
wordpress/
  Chart.yaml          # A YAML file containing information about the chart
  LICENSE             # OPTIONAL: A plain text file containing the license for the chart
  README.md           # OPTIONAL: A human-readable README file
  requirements.yaml   # OPTIONAL: A YAML file listing dependencies for the chart
  values.yaml         # The default configuration values for this chart
  charts/             # A directory containing any charts upon which this chart depends.
  templates/          # A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
  templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
```

Helm保留使用charts/和templates/目录以及上面列出的文件名称。其他文件将被忽略。

## Chart.yaml文件

`Chart.yaml`文件是chart所必需的。它包含以下字段：

```yaml
apiVersion: The chart API version, always "v1" (required)
name: The name of the chart (required)
version: A SemVer 2 version (required)
kubeVersion: A SemVer range of compatible Kubernetes versions (optional)
description: A single-sentence description of this project (optional)
keywords:
  - A list of keywords about this project (optional)
home: The URL of this project's home page (optional)
sources:
  - A list of URLs to source code for this project (optional)
maintainers: # (optional)
  - name: The maintainer's name (required for each maintainer)
    email: The maintainer's email (optional for each maintainer)
    url: A URL for the maintainer (optional for each maintainer)
engine: gotpl # The name of the template engine (optional, defaults to gotpl)
icon: A URL to an SVG or PNG image to be used as an icon (optional).
appVersion: The version of the app that this contains (optional). This needn't be SemVer.
deprecated: Whether this chart is deprecated (optional, boolean)
tillerVersion: The version of Tiller that this chart requires. This should be expressed as a SemVer range: ">2.0.0" (optional)
```

如果熟悉`Chart.yaml` Helm Classic 的文件格式，注意到指定依赖性的字段已被删除。这是因为新的chart使用charts/目录表示依赖关系。

其他字段将被忽略。

### Charts和版本控制
每个chart都必须有一个版本号。版本必须遵循[SemVer 2](http://semver.org/)标准。与Helm Class 格式不同，Kubernetes Helm使用版本号作为发布标记。存储库中的软件包由名称加版本识别。

例如，`nginx` version字段设置为1.2.3将被命名为：

```
nginx-1.2.3.tgz
```

更复杂的SemVer 2命名也是支持的，例如 version: 1.2.3-alpha.1+ef365。但非SemVer命名是明确禁止的。

**注意**：虽然Helm Classic和Deployment Manager在chart方面都非常适合GitHub，但Kubernetes Helm并不依赖或需要GitHub甚至Git。因此，它不使用Git SHA进行版本控制。

许多Helm工具都使用`Chart.yaml`的v`ersion`字段，其中包括CLI和Tiller服务。在生成包时，helm package命令将使用它在Chart.yaml中的版本名作为包名。系统假定chart包名称中的版本号与`Chart.yaml`中的版本号相匹配。不符合这个情况会导致错误。

### appVersion字段

请注意，`appVersion`字段与`version`字段无关。这是一种指定应用程序版本的方法。例如，`drupal` chart可能有一个`appVersion: 8.2.1`，表示chart中包含的Drupal版本（默认情况下）是`8.2.1`。该字段是信息标识，对chart版本没有影响。

### 弃用charts

在管理chart tepo库中的chart时，有时需要弃用chart。`Chart.yaml`的d`eprecated`字段可用于将chart标记为已弃用。如果存储库中最新版本的chart标记为已弃用，则整个chart被视为已弃用。chart名称稍后可以通过发布未标记为已弃用的较新版本来重新使用。废弃chart的工作流程根据[kubernetes/charts](https://github.com/kubernetes/charts) 项目的工作流程如下：

-   更新chart的`Chart.yaml`以将chart标记为启用，并且更新版本
-   在chart Repository中发布新的chart版本
-   从源代码库中删除chart（例如git）

## Chart许可证文件，自述文件和说明文件
chart还可以包含描述chart的安装，配置，使用和许可证的文件。chart的自述文件应由Markdown（README.md）语法格式化，并且通常应包含：

-   chart提供的应用程序或服务的描述
-   运行chart的任何前提条件或要求
-   选项`values.yaml`和默认值的说明
-   任何其他可能与安装或配置chart相关的信息

chart还可以包含一个简短的纯文本`templates/NOTES.txt`文件，在安装后以及查看版本状态时将打印出来。此文件将作为模板[template](#templates-and-values)进行评估 ，并可用于显示使用说明，后续步骤或任何其他与发布chart相关的信息。例如，可以提供用于连接到数据库或访问Web UI的指令。由于运行时，该文件被打印到标准输出 `helm install`或`helm status`，建议保持内容简短并把更多细节指向自述文件。

## Chart依赖关系

在Helm中，一个chart可能依赖于任何数量的其他chart。这些依赖关系可以通过requirements.yaml 文件动态链接或引入`charts/`目录并手动管理。

虽然有一些团队需要手动管理依赖关系的优势，但声明依赖关系的首选方法是使用 chart 内部的`requirements.yaml`文件。

**注意：** 传统Helm 的Chart.yaml `dependencies:`部分字段已被完全删除弃用。

### 用`requirements.yaml`来管理依赖关系

`requirements.yaml`文件是列出chart的依赖关系的简单文件。

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: http://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: http://another.example.com/charts
```

-   该name字段是chart的名称。
-   version字段是chart的版本。
-   repository字段是chart repo的完整URL。请注意，还必须使用helm repo add添加该repo到本地才能使用。

有了依赖关系文件，你可以通过运行`helm dependency update` ，它会使用你的依赖关系文件将所有指定的chart下载到你的`charts/`目录中。

```console
$ helm dep up foochart
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "example" chart repository
...Successfully got an update from the "another" chart repository
Update Complete. Happy Helming!
Saving 2 charts
Downloading apache from repo http://example.com/charts
Downloading mysql from repo http://another.example.com/charts
```

当`helm dependency update`检索chart时，它会将它们作为chart存档存储在`charts/`目录中。因此，对于上面的示例，可以在chart目录中看到以下文件：


```
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

通过`requirements.yaml`管理chart是一种轻松更新chart的好方法，还可以在整个团队中共享requirements信息。

#### requirements.yaml中的`alias`字段
除上述其他字段外，每个requirement条目可能包含可选字段`alias`。

为依赖的chart添加别名会将chart放入依赖关系中，并使用别名作为新依赖关系的名称。

如果需要使用其他名称访问chart，可以使用`alias`。

```yaml
# parentchart/requirements.yaml
dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-1
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-2
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
```

在上面的例子中，我们将得到parentchart的3个依赖关系

```
subchart
new-subchart-1
new-subchart-2
```

实现这一目的的手动方法是`charts/`中用不同名称多次复制/粘贴目录中的同一chart 。

#### requirements.yaml中的`tags`和`condition`字段

除上述其他字段外，每个需求条目可能包含可选字段`tags`和`condition`。

所有charts都会默认加载。如果存在`tags`或`condition`字段，将对它们进行评估并用于控制应用的chart的加载。

Condition - condition 字段包含一个或多个YAML路径（用逗号分隔）。如果此路径存在于顶级父级的值中并且解析为布尔值，则将根据该布尔值启用或禁用chart。只有在列表中找到的第一个有效路径才被评估，如果没有路径存在，那么该条件不起作用。

Tags - 标签字段是与此chart关联的YAML标签列表。在顶级父级的值中，可以通过指定标签和布尔值来启用或禁用所有带有标签的chart。

````
# parentchart/requirements.yaml
dependencies:
      - name: subchart1
        repository: http://localhost:10191
        version: 0.1.0
        condition: subchart1.enabled, global.subchart1.enabled
        tags:
          - front-end
          - subchart1

      - name: subchart2
        repository: http://localhost:10191
        version: 0.1.0
        condition: subchart2.enabled,global.subchart2.enabled
        tags:
          - back-end
          - subchart2
````
````
# parentchart/values.yaml

subchart1:
  enabled: true
tags:
  front-end: false
  back-end: true
````

在上面的示例中，所有带有标签的`front-end的`charts都将被禁用，但由于 `subchart1.enabled`的值在父项值中为“真”，因此条件将覆盖该 `front-end`标签，`subchart1`会启用。

由于`subchart2`被标记`back-end`和标签的计算结果为`true`，`subchart2`将被启用。还要注意的是，虽然`subchart2`有一个在`requirements.yaml`中指定的条件，但父项的值中没有对应的路径和值，因此条件无效。

#### 使用命令行时带有tag和conditions

`--set`参数可使用来更改tag和conditions值。

````
helm install --set tags.front-end=true --set subchart2.enabled=false

````

##### `tags`和`conditions`解析

*   **Conditions (设置 values) 会覆盖tags配置.**。第一个存在的condition路径生效，后续该chart的condition路径将被忽略。
*   如果chart的某tag的任一tag的值为true，那么该tag的值为true，并启用这个chart。
*   Tags 和 conditions值必须在顶级父级的值中进行设置。
*   `tags:`值中的关键字必须是顶级关键字。目前不支持全局和嵌套`tags:`表格。

#### 通过requirements.yaml导入子值

在某些情况下，希望允许子chart的值传到父chart并作为通用默认值共享。使用这种exports格式的另一个好处是它可以使未来的工具能够考虑用户可设置的值。

要导入的值的键可以在父chart文件中`requirements.yaml`使用YAML list指定。list中的每个项目都是从子chart `exports`字段导入的key。

要导入不包含在exports key中的值，请使用子父级[child-parent](#using-the-child-parent-format)格式。下面描述了两种格式的例子。

##### 使用exports格式
如果子chart的values.yaml文件exports在根目录中包含一个字段，则可以通过指定要导入的关键字将其内容直接导入到父项的值中，如下例所示：


```yaml
# parent's requirements.yaml file
    ...
    import-values:
      - data
```
```yaml
# child's values.yaml file
...
exports:
  data:
    myint: 99
```

由于我们在导入列表中指定了`data`键，因此Helm会在exports子图的字段中查找data键并导入其内容。

最终的父值将包含我们的导出字段：

```yaml
# parent's values file
...
myint: 99

```
请注意，父键data不包含在父母的最终值中。如果您需要指定父键，请使用'child-parent'格式。

#### 使用child-parent格式

要访问未包含在子chart键值`exports`的中的值，需要指定要导入的值的源键（`child`）和父chart值（`parent`）中的目标路径。

下面的例子中的`import-values`告诉Helm去拿在`child:`路径发现的任何值，并将其复制到父值`parent:`指定的路径

```yaml
# parent's requirements.yaml file
dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    ...
    import-values:
      - child: default.data
        parent: myimports
```

在上面的例子中，在subchart1`default.data`的值中找到的值将被导入到父chart值中`myimports`的键值，详细如下：

```yaml
# parent's values.yaml file

myimports:
  myint: 0
  mybool: false
  mystring: "helm rocks!"

```
```yaml
# subchart1's values.yaml file

default:
  data:
    myint: 999
    mybool: true

```

父chart的结果值为：

```yaml
# parent's final values

myimports:
  myint: 999
  mybool: true
  mystring: "helm rocks!"

```

父chart的最终值现在包含从subchart1导入的`myint`和`mybool`字段。

### 通过`charts/`目录手动管理依赖性

如果需要更多的控制依赖关系，可以通过将依赖的charts复制到`charts/`目录中来明确表达这些依赖关系 。

依赖关系可以是chart归档（`foo-1.2.3.tgz`）或解压缩的chart目录。但它的名字不能从`_`或`.`开始。这些文件被chart加载器忽略。

例如，如果WordPress chart依赖于Apache chart，则在WordPress chart的`charts/`目录中提供（正确版本的）Apache chart：

```
wordpress:
  Chart.yaml
  requirements.yaml
  # ...
  charts/
    apache/
      Chart.yaml
      # ...
    mysql/
      Chart.yaml
      # ...
```

上面的示例显示了WordPress chart如何通过在其`charts/``目录中包含这些charts来表示它对Apache和MySQL的依赖关系。

**提示：** 将依赖项放入您的charts/目录，请使用` helm fetch`命令

### 使用依赖关系的操作方面影响

上面的部分解释了如何指定chart依赖关系，但是这会如何影响使用`helm install`和`helm upgrade`的chart安装？

假设名为“A”的chart创建以下Kubernetes对象

-   namespace "A-Namespace"
-   statefulset "A-StatefulSet"
-   service "A-Service"

此外，A依赖于创建对象的chart B.

-   namespace "B-Namespace"
-   replicaset "B-ReplicaSet"
-   service "B-Service"

安装/升级chart A后，会创建/修改单个Helm版本。该版本将按以下顺序创建/更新所有上述Kubernetes对象：

-   A-Namespace
-   B-Namespace
-   A-StatefulSet
-   B-ReplicaSet
-   A-Service
-   B-Service

这是因为当Helm安装/升级charts时，charts中的Kubernetes对象及其所有依赖项都是如下

-   聚合成一个单一的集合; 然后
-   按类型排序，然后按名称排序; 接着
-   按该顺序创建/更新。

因此，单个release是使用charts及其依赖关系创建的所有对象。

Kubernetes类型的安装顺序由kind_sorter.go中的枚举InstallOrder给出（[the Helm source file](https://github.com/kubernetes/helm/blob/master/pkg/tiller/kind_sorter.go#L26))）。

## 模板Templates和值Values
Helm chart模板是用Go模板语言[Go template language](https://golang.org/pkg/text/template/)编写的 ，其中添加了来自Sprig库[from the Sprig library](https://github.com/Masterminds/sprig)的50个左右的附加模板函数以及一些其他专用函数[specialized functions](charts_tips_and_tricks.md)。

所有模板文件都存储在chart的templates/文件夹中。当Helm渲染charts时，它将通过模板引擎传递该目录中的每个文件。

模板的值有两种提供方法：

-   chart开发人员可能会在chart内部提供一个values.yaml文件。该文件可以包含默认值。
-   chart用户可能会提供一个包含值的YAML文件。这可以通过命令行提供`helm install -f`。

当用户提供自定义值时，这些值将覆盖chart中v`alues.yaml`文件中的值。

### 模板文件
模板文件遵循用于编写Go模板的标准约定（请参阅文本/模板Go包文档 以了解详细信息）。示例模板文件可能如下所示：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    heritage: deis
spec:
  replicas: 1
  selector:
    app: deis-database
  template:
    metadata:
      labels:
        app: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```

上面的示例基于[此网址](https://github.com/deis/charts)，是Kubernetes replication controller的模板。它可以使用以下四个模板值（通常在values.yaml文件中定义 ）：

-   imageRegistry：Docker镜像的源。
-   dockerTag：docker镜像的标签。
-   pullPolicy：Kubernetes镜像拉取策略。
-   storage：存储后端，其默认设置为 `"minio"`

所有这些值都由模板作者定义。Helm不需要或指定参数。

要查更多charts，请查看Kubernetes charts项目

### 预定义值

通过`values.yaml`文件（或通过`--set` 标志）提供的值可以从`.Values`模板中的对象访问。可以在模板中访问其他预定义的数据片段。

以下值是预定义的，可用于每个模板，并且不能被覆盖。与所有值一样，名称区分大小写。

-   `Release.Name`：release的名称（不是chart）
-   `Release.Time`：chart版本上次更新的时间。这将匹配`Last Released`发布对象上的时间。
-   `Release.Namespace`：chart release发布的namespace。
-   `Release.Service`：进行发布的服务。通常是Tiller。
-   `Release.IsUpgrade`：如果当前操作是升级或回滚，则设置为true。
-   `Release.IsInstall`：如果当前操作是安装，则设置为true。
-   `Release.Revision`：版本号。它从1开始，并随着每个helm upgrade增加。
-   `Chart`：`Chart.yaml`的内容。chart版本可以从`Chart.Version`和维护人员 `Chart.Maintainers`一起获得。
-   `Files`：包含chart中所有非特殊文件的map-like对象。不会允许你访问模板，但会让你访问存在的其他文件（除非它们被排除使用`.helmignore`）。可以使用`{{index .Files "file.name"}}`或使用`{{.Files.Get name}}`或 `{{.Files.GetString name}}`功能来访问文件。也可以使用`{{.Files.GetBytes}}`访问该文件的内容`[byte]`
-   Capabilities：包含有关Kubernetes版本信息的map-like对象（`{{.Capabilities.KubeVersion}}`，Tiller（`{{.Capabilities.TillerVersion}}`和支持的Kubernetes API版本（`{{.Capabilities.APIVersions.Has "batch/v1"}}`）

**注意:** 任何未知的Chart.yaml字段将被删除。它们不会在chart对象内部被访问。因此，Chart.yaml不能用于将任意结构化的数据传递到模板中。values文件可以用于传递。

### 值values文件

考虑到上一节中的模板`values.yaml`，提供了如下必要值的信息：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

values文件是YAML格式的。chart可能包含一个默认 values.yaml文件。Helm install命令允许用户通过提供额外的YAML值来覆盖值：

```console
$ helm install --values=myvals.yaml wordpress
```

当以这种方式传递值时，它们将被合并到默认values文件中。例如，考虑一个如下所示的myvals.yaml文件：

```yaml
storage: "gcs"
```

当它与chart中values.yaml的内容合并时，生成的内容将为：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "gcs"
```

注意只有最后一个字段被覆盖了，其他的不变。

**注：**包含在chart内的默认values文件必须命名 `values.yaml`。但是在命令行上指定的文件可以被命名为任何名称。

**注：**如果在helm install或helm upgrade使用`--set`，则这些值仅在客户端转换为YAML。

**注意：**如果values文件中存在任何必需的条目，则可以使用'required'功能['required' function](charts_tips_and_tricks-zh_cn.md)在chart模板中声明它们

然后可以在模板内部访问任何这些`.Values`对象值 ：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    heritage: deis
spec:
  replicas: 1
  selector:
    app: deis-database
  template:
    metadata:
      labels:
        app: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}

```

### 范围Scope，依赖Dependencies和值Values

values文件可以声明顶级chart的值，也可以为chart的charts/目录中包含的任何chart声明值。或者，用不同的方式来描述它，values文件可以为chart及其任何依赖项提供值。例如，上面的演示WordPresschart具有mysql和apache依赖性。values文件可以为所有这些组件提供值：

```yaml
title: "My WordPress Site" # Sent to the WordPress template

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

更高级别的chart可以访问下面定义的所有变量。所以WordPresschart可以访问MySQL密码 `.Values.mysql.password`。但较低级别的chart无法访问父chart中的内容，因此MySQL将无法访问该`title`属性。同样的，也不能访问`apache.port`。

值是命名空间限制的，但命名空间已被修剪。因此对于WordPresschart来说，它可以访问MySQL密码字段`.Values.mysql.password`。但是对于MySQL chart来说，这些值的范围已经减小了，并且删除了名namespace前缀，所以它会将密码字段简单地视为 `.Values.password`。

#### 全局值

从2.0.0-Alpha.2开始，Helm支持特殊的“全局”值。考虑前面例子的这个修改版本：

标题：“我的WordPress网站”  ＃发送到WordPress模板

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

上面添加了一个global区块，值`app: MyWordPress`。此值可供所有chart使用`.Values.global.app`。

比如，该mysql模板可以访问`app`如`{{.Values.global.app}}`，apache chart也同样的。上面的values文件是这样高效重新生成的：


```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  global:
    app: MyWordPress
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  global:
    app: MyWordPress
  port: 8080 # Passed to Apache
```

这提供了一种与所有子chart共享一个顶级变量的方法，这对设置metadata中像标签这样的属性很有用。

如果子chart声明了一个全局变量，则该全局将向下传递 （到子chart的子chart），但不向上传递到父chart。子chart无法影响父chart的值。

此外，父chart的全局变量优先于子chart中的全局变量。

### 参考
当涉及到编写模板和values文件时，有几个标准参考可以帮助你。

-   [Go templates](https://godoc.org/text/template)
-   [Extra template functions](https://godoc.org/github.com/Masterminds/sprig)
-   [The YAML format](http://yaml.org/spec/)

## 使用Helm管理chart

该helm工具有几个用于处理chart的命令。

它可以为你创建一个新的chart：

```console
$ helm create mychart
Created mychart/
```

编辑完chart后，`helm`可以将其打包到chart压缩包中：

```console
$ helm package mychart
Archived mychart-0.1.-.tgz
```

您可以用`helm`来帮助查找chart格式或信息的问题：

```console
$ helm lint mychart
No issues found
```

## Chart repo库
chart repo库是容纳一个或多个封装的chart的HTTP服务器。虽然helm可用于管理本地chart目录，但在共享chart时，首选机制是chart repo库。

任何可以提供YAML文件和tar文件并可以回答GET请求的HTTP服务器都可以用作repo库服务器。

Helm附带用于开发人员测试的内置服务器（`helm serve`）。Helm团队测试了其他服务器，包括启用了网站模式的Google Cloud Storage以及启用了网站模式的S3。

repo库的主要特征是存在一个名为的特殊文件`index.yaml`，它具有repo库提供的所有软件包的列表以及允许检索和验证这些软件包的元数据。

在客户端，repo库使用`helm repo`命令进行管理。但是，Helm不提供将chart上传到远程存储服务器的工具。这是因为这样做会增加部署服务器的需求，从而增加配置repo库的难度。

## Chart 起始包

`helm create`命令采用可选`--starter`选项，可以指定“起始chart”。

起始chart只是普通的chart，位于$HELM_HOME/starters。作为chart开发人员，可以创作专门设计用作起始的chart。记住这些chart时应考虑以下因素：

-   `Chart.yaml`将被生成器覆盖。
-   用户将期望修改这样的chart内容，因此文档应该指出用户如何做到这一点。
-   所有匹配项<CHARTNAME>将被替换为指定的chart名称，以便起始chart可用作模板。
-   目前添加chart的唯一方法是手动将其复制到`$HELM_HOME/starters`。在chart的文档中，你需要解释该过程。
