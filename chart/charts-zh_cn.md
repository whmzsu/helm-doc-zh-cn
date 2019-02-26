# Charts
Helm 使用称为 chart 的包装格式。chart 是描述相关的一组 Kubernetes 资源的文件集合。单个 chart 可能用于部署简单的东西，比如 memcached pod，或者一些复杂的东西，比如完整的具有 HTTP 服务，数据库，缓存等的 Web 应用程序堆栈。

chart 通过创建为特定目录树的文件，将它们打包到版本化的压缩包，然后进行部署。

本文档解释了 chart 格式，提供使用 Helm 构建 chart 的基本指导。

## Chart 文件结构

chart 被组织为一个目录内的文件集合。目录名称是 chart 的名称（没有版本信息）。例如，描述 WordPress 的 chart 将被存储在 wordpress / 目录中。

在这个目录里面，Helm 期望如下这样一个的结构的目录树：

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

Helm 保留使用 charts / 和 templates / 目录以及上面列出的文件名称。其他文件将被忽略。

## Chart.yaml 文件

`Chart.yaml` 文件是 chart 所必需的。它包含以下字段：

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

如果熟悉 `Chart.yaml` Helm Classic 的文件格式，注意到指定依赖性的字段已被删除。这是因为新的 chart 使用 charts / 目录表示依赖关系。

其他字段将被忽略。

### Charts 和版本控制
每个 chart 都必须有一个版本号。版本必须遵循 [SemVer 2](http://semver.org/) 标准。与 Helm Class 格式不同，Kubernetes Helm 使用版本号作为发布标记。存储库中的软件包由名称加版本识别。

例如，`nginx` version 字段设置为 1.2.3 将被命名为：

```
nginx-1.2.3.tgz
```

更复杂的 SemVer 2 命名也是支持的，例如 version: 1.2.3-alpha.1+ef365。但非 SemVer 命名是明确禁止的。

** 注意 ** ：虽然 Helm Classic 和 Deployment Manager 在 chart 方面都非常适合 GitHub，但 Kubernetes Helm 并不依赖或需要 GitHub 甚至 Git。因此，它不使用 Git SHA 进行版本控制。

许多 Helm 工具都使用 `Chart.yaml` 的 v`ersion` 字段，其中包括 CLI 和 Tiller 服务。在生成包时，helm package 命令将使用它在 Chart.yaml 中的版本名作为包名。系统假定 chart 包名称中的版本号与 `Chart.yaml` 中的版本号相匹配。不符合这个情况会导致错误。

### appVersion 字段

请注意，`appVersion` 字段与 `version` 字段无关。这是一种指定应用程序版本的方法。例如，`drupal` chart 可能有一个 `appVersion: 8.2.1`，表示 chart 中包含的 Drupal 版本（默认情况下）是 `8.2.1`。该字段是信息标识，对 chart 版本没有影响。

### 弃用 charts

在管理 chart tepo 库中的 chart 时，有时需要弃用 chart。`Chart.yaml` 的 d`eprecated` 字段可用于将 chart 标记为已弃用。如果存储库中最新版本的 chart 标记为已弃用，则整个 chart 被视为已弃用。chart 名称稍后可以通过发布未标记为已弃用的较新版本来重新使用。废弃 chart 的工作流程根据 [helm/charts](https://github.com/helm/charts) 项目的工作流程如下：

-   更新 chart 的 `Chart.yaml` 以将 chart 标记为启用，并且更新版本
-   在 chart Repository 中发布新的 chart 版本
-   从源代码库中删除 chart（例如 git）

## Chart 许可证文件，自述文件和说明文件
Chart 还可以包含描述 chart 的安装，配置，使用和许可证的文件。

LICENSE 文件是一个纯文本文件，包含 chart 的 [许可证]（https://en.wikipedia.org/wiki/Software_license）。 Chart 可以包含许可证，它可能在模板中具有编程逻辑，因此不仅仅是配置。 如果需要，还可以为 chart 安装的应用程序提供单独的许可证。

Chart 的自述文件应由 Markdown（README.md）语法格式化，并且通常应包含：

-   chart 提供的应用程序或服务的描述
-   运行 chart 的任何前提条件或要求
-   选项 `values.yaml` 和默认值的说明
-   任何其他可能与安装或配置 chart 相关的信息

chart 还可以包含一个简短的纯文本 `templates/NOTES.txt` 文件，在安装后以及查看版本状态时将打印出来。此文件将作为模板 [template](#templates-and-values) 进行评估 ，并可用于显示使用说明，后续步骤或任何其他与发布 chart 相关的信息。例如，可以提供用于连接到数据库或访问 Web UI 的指令。由于运行时，该文件被打印到标准输出 `helm install` 或 `helm status`，建议保持内容简短并把更多细节指向自述文件。

## Chart 依赖关系

在 Helm 中，一个 chart 可能依赖于任何数量的其他 chart。这些依赖关系可以通过 requirements.yaml 文件动态链接或引入 `charts/` 目录并手动管理。

虽然有一些团队需要手动管理依赖关系的优势，但声明依赖关系的首选方法是使用 chart 内部的 `requirements.yaml` 文件。

** 注意：** 传统 Helm 的 Chart.yaml `dependencies:` 部分字段已被完全删除弃用。

### 用 `requirements.yaml` 来管理依赖关系

`requirements.yaml` 文件是列出 chart 的依赖关系的简单文件。

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: http://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: http://another.example.com/charts
```

-   该 name 字段是 chart 的名称。
-   version 字段是 chart 的版本。
-   repository 字段是 chart repo 的完整 URL。请注意，还必须使用 helm repo add 添加该 repo 到本地才能使用。

有了依赖关系文件，你可以通过运行 `helm dependency update` ，它会使用你的依赖关系文件将所有指定的 chart 下载到你的 `charts/` 目录中。

```
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

当 `helm dependency update` 检索 chart 时，它会将它们作为 chart 存档存储在 `charts/` 目录中。因此，对于上面的示例，可以在 chart 目录中看到以下文件：


```bash
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

通过 `requirements.yaml` 管理 chart 是一种轻松更新 chart 的好方法，还可以在整个团队中共享 requirements 信息。

#### requirements.yaml 中的 `alias` 字段
除上述其他字段外，每个 requirement 条目可能包含可选字段 `alias`。

为依赖的 chart 添加别名会将 chart 放入依赖关系中，并使用别名作为新依赖关系的名称。

如果需要使用其他名称访问 chart，可以使用 `alias`。

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

在上面的例子中，我们将得到 parentchart 的 3 个依赖关系

```bash
subchart
new-subchart-1
new-subchart-2
```

实现这一目的的手动方法是 `charts/` 中用不同名称多次复制 / 粘贴目录中的同一 chart 。

#### requirements.yaml 中的 `tags` 和 `condition` 字段

除上述其他字段外，每个需求条目可能包含可选字段 `tags` 和 `condition`。

所有 charts 都会默认加载。如果存在 `tags` 或 `condition` 字段，将对它们进行评估并用于控制应用的 chart 的加载。

Condition - condition 字段包含一个或多个 YAML 路径（用逗号分隔）。如果此路径存在于顶级父级的值中并且解析为布尔值，则将根据该布尔值启用或禁用 chart。只有在列表中找到的第一个有效路径才被评估，如果没有路径存在，那么该条件不起作用。

Tags - 标签字段是与此 chart 关联的 YAML 标签列表。在顶级父级的值中，可以通过指定标签和布尔值来启用或禁用所有带有标签的 chart。

```yaml
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
```
```yaml
# parentchart/values.yaml

subchart1:
  enabled: true
tags:
  front-end: false
  back-end: true
```

在上面的示例中，所有带有标签的 `front-end 的 `charts 都将被禁用，但由于 `subchart1.enabled` 的值在父项值中为 “真”，因此条件将覆盖该 `front-end` 标签，`subchart1` 会启用。

由于 `subchart2` 被标记 `back-end` 和标签的计算结果为 `true`，`subchart2` 将被启用。还要注意的是，虽然 `subchart2` 有一个在 `requirements.yaml` 中指定的条件，但父项的值中没有对应的路径和值，因此条件无效。

#### 使用命令行时带有 tag 和 conditions

`--set` 参数可使用来更改 tag 和 conditions 值。

```bash
helm install --set tags.front-end=true --set subchart2.enabled=false

```

##### `tags` 和 `conditions` 解析

*   **Conditions (设置 values) 会覆盖 tags 配置.**。第一个存在的 condition 路径生效，后续该 chart 的 condition 路径将被忽略。
*   如果 chart 的某 tag 的任一 tag 的值为 true，那么该 tag 的值为 true，并启用这个 chart。
*   Tags 和 conditions 值必须在顶级父级的值中进行设置。
*   `tags:` 值中的关键字必须是顶级关键字。目前不支持全局和嵌套 `tags:` 表格。

#### 通过 requirements.yaml 导入子值

在某些情况下，希望允许子 chart 的值传到父 chart 并作为通用默认值共享。使用这种 exports 格式的另一个好处是它可以使未来的工具能够考虑用户可设置的值。

要导入的值的键可以在父 chart 文件中 `requirements.yaml` 使用 YAML list 指定。list 中的每个项目都是从子 chart `exports` 字段导入的 key。

要导入不包含在 exports key 中的值，请使用子父级 [child-parent](#using-the-child-parent-format) 格式。下面描述了两种格式的例子。

##### 使用 exports 格式
如果子 chart 的 values.yaml 文件 exports 在根目录中包含一个字段，则可以通过指定要导入的关键字将其内容直接导入到父项的值中，如下例所示：


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

由于我们在导入列表中指定了 `data` 键，因此 Helm 会在 exports 子图的字段中查找 data 键并导入其内容。

最终的父值将包含我们的导出字段：

```yaml
# parent's values file
...
myint: 99

```
请注意，父键 data 不包含在父 chart 的最终值中。如果需要指定父键，请使用'child-parent'格式。

#### 使用 child-parent 格式

要访问未包含在子 chart 键值 `exports` 的中的值，需要指定要导入的值的源键（`child`）和父 chart 值（`parent`）中的目标路径。

下面的例子中的 `import-values` 告诉 Helm 去拿在 `child:` 路径发现的任何值，并将其复制到父值 `parent:` 指定的路径

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

在上面的例子中，在 subchart1`default.data` 的值中找到的值将被导入到父 chart 值中 `myimports` 的键值，详细如下：

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

父 chart 的结果值为：

```yaml
# parent's final values

myimports:
  myint: 999
  mybool: true
  mystring: "helm rocks!"

```

父 chart 的最终值现在包含从 subchart1 导入的 `myint` 和 `mybool` 字段。

### 通过 `charts/` 目录手动管理依赖性

如果需要更多的控制依赖关系，可以通过将依赖的 charts 复制到 `charts/` 目录中来明确表达这些依赖关系 。

依赖关系可以是 chart 归档（`foo-1.2.3.tgz`）或解压缩的 chart 目录。但它的名字不能从 `_` 或 `.` 开始。这些文件被 chart 加载器忽略。

例如，如果 WordPress chart 依赖于 Apache chart，则在 WordPress chart 的 `charts/` 目录中提供（正确版本的）Apache chart：

```bash
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

上面的示例显示了 WordPress chart 如何通过在其 `charts/` 目录中包含这些 charts 来表示它对 Apache 和 MySQL 的依赖关系。

** 提示：** 将依赖项放入 charts / 目录，请使用 ` helm fetch` 命令

### 使用依赖关系的操作方面影响

上面的部分解释了如何指定 chart 依赖关系，但是这会如何影响使用 `helm install` 和 `helm upgrade` 的 chart 安装？

假设名为 “A” 的 chart 创建以下 Kubernetes 对象

-   namespace "A-Namespace"
-   statefulset "A-StatefulSet"
-   service "A-Service"

此外，A 依赖于创建对象的 chart B.

-   namespace "B-Namespace"
-   replicaset "B-ReplicaSet"
-   service "B-Service"

安装 / 升级 chart A 后，会创建 / 修改单个 Helm 版本。该版本将按以下顺序创建 / 更新所有上述 Kubernetes 对象：

-   A-Namespace
-   B-Namespace
-   A-StatefulSet
-   B-ReplicaSet
-   A-Service
-   B-Service

这是因为当 Helm 安装 / 升级 charts 时，charts 中的 Kubernetes 对象及其所有依赖项都是如下

-   聚合成一个单一的集合; 然后
-   按类型排序，然后按名称排序; 接着
-   按该顺序创建 / 更新。

因此，单个 release 是使用 charts 及其依赖关系创建的所有对象。

Kubernetes 类型的安装顺序由 kind_sorter.go 中的枚举 InstallOrder 给出（[the Helm source file](https://github.com/helm/blob/master/pkg/tiller/kind_sorter.go#L26))）。

## 模板 Templates 和值 Values

Helm chart 模板是用 Go 模板语言 [Go template language](https://golang.org/pkg/text/template/) 编写的 ，其中添加了来自 Sprig 库 [from the Sprig library](https://github.com/Masterminds/sprig) 的 50 个左右的附加模板函数以及一些其他专用函数 [specialized functions](charts_tips_and_tricks.md)。

所有模板文件都存储在 chart 的 templates / 文件夹中。当 Helm 渲染 charts 时，它将通过模板引擎传递该目录中的每个文件。

模板的值有两种提供方法：

-   chart 开发人员可能会在 chart 内部提供一个 values.yaml 文件。该文件可以包含默认值。
-   chart 用户可能会提供一个包含值的 YAML 文件。这可以通过命令行提供 `helm install -f`。

当用户提供自定义值时，这些值将覆盖 chart 中 v`alues.yaml` 文件中的值。

### 模板文件

模板文件遵循用于编写 Go 模板的标准约定（请参阅文 [the text/template Go package documentation](https://golang.org/pkg/text/template/) 以了解详细信息）。示例模板文件可能如下所示：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
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

上面的示例基于 [此网址](https://github.com/deis/charts)，是 Kubernetes replication controller 的模板。它可以使用以下四个模板值（通常在 values.yaml 文件中定义 ）：

-   imageRegistry：Docker 镜像的源。
-   dockerTag：docker 镜像的标签。
-   pullPolicy：Kubernetes 镜像拉取策略。
-   storage：存储后端，其默认设置为 `"minio"`

所有这些值都由模板作者定义。Helm 不需要或指定参数。

要查更多 charts，请查看 Kubernetes charts 项目

### 预定义值

通过 `values.yaml` 文件（或通过 `--set` 标志）提供的值可以从 `.Values` 模板中的对象访问。可以在模板中访问其他预定义的数据片段。

以下值是预定义的，可用于每个模板，并且不能被覆盖。与所有值一样，名称区分大小写。

-   `Release.Name`：release 的名称（不是 chart 的）
-   `Release.Time`：chart 版本上次更新的时间。这将匹配 `Last Released` 发布对象上的时间。
-   `Release.Namespace`：chart release 发布的 namespace。
-   `Release.Service`：处理 release 的服务。通常是 Tiller。
-   `Release.IsUpgrade`：如果当前操作是升级或回滚，则设置为 true。
-   `Release.IsInstall`：如果当前操作是安装，则设置为 true。
-   `Release.Revision`：版本号。它从 1 开始，并随着每个 `helm upgrade` 增加。
-   `Chart`：`Chart.yaml` 的内容。chart 版本可以从 `Chart.Version` 和维护人员 `Chart.Maintainers` 一起获得。
-   `Files`：包含 chart 中所有非特殊文件的 map-like 对象。不会允许你访问模板，但会让你访问存在的其他文件（除非它们被排除使用 `.helmignore`）。可以使用 index .Files "file.name" 或使用. Files.Get name 或 .Files.GetString name 功能来访问文件。也可以使用. Files.GetBytes 访问该文件的内容 `[byte]`
-   Capabilities：包含有关 Kubernetes 版本信息的 map-like 对象（.Capabilities.KubeVersion)，Tiller（.Capabilities.TillerVersion) 和支持的 Kubernetes API 版本（.Capabilities.APIVersions.Has "batch/v1"）

** 注意:** 任何未知的 Chart.yaml 字段将被删除。它们不会在 chart 对象内部被访问。因此，Chart.yaml 不能用于将任意结构化的数据传递到模板中。values 文件可以用于传递。

### 值 values 文件

考虑到上一节中的模板 `values.yaml`，提供了如下必要值的信息：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

values 文件是 YAML 格式的。chart 可能包含一个默认 values.yaml 文件。Helm install 命令允许用户通过提供额外的 YAML 值来覆盖值：

```bash
$ helm install --values=myvals.yaml wordpress
```

当以这种方式传递值时，它们将被合并到默认 values 文件中。例如，考虑一个如下所示的 myvals.yaml 文件：

```yaml
storage: "gcs"
```

当它与 chart 中 values.yaml 的内容合并时，生成的内容将为：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "gcs"
```

注意只有最后一个字段被覆盖了，其他的不变。

** 注：** 包含在 chart 内的默认 values 文件必须命名 `values.yaml`。但是在命令行上指定的文件可以被命名为任何名称。

** 注：** 如果在 helm install 或 helm upgrade 使用 `--set`，则这些值仅在客户端转换为 YAML。

** 注意：** 如果 values 文件中存在任何必需的条目，则可以使用'required'功能 ['required' function](charts_tips_and_tricks-zh_cn.md) 在 chart 模板中声明它们

然后可以在模板内部访问任何这些 `.Values` 对象值 ：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
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

### 范围 Scope，依赖 Dependencies 和值 Values

values 文件可以声明顶级 chart 的值，也可以为 chart 的 charts / 目录中包含的任何 chart 声明值。或者，用不同的方式来描述它，values 文件可以为 chart 及其任何依赖项提供值。例如，上面的演示 WordPresschart 具有 mysql 和 apache 依赖性。values 文件可以为所有这些组件提供值：

```yaml
title: "My WordPress Site" # Sent to the WordPress template

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

更高级别的 chart 可以访问下面定义的所有变量。所以 WordPresschart 可以访问 MySQL 密码 `.Values.mysql.password`。但较低级别的 chart 无法访问父 chart 中的内容，因此 MySQL 将无法访问该 `title` 属性。同样的，也不能访问 `apache.port`。

值是命名空间限制的，但命名空间已被修剪。因此对于 WordPresschart 来说，它可以访问 MySQL 密码字段 `.Values.mysql.password`。但是对于 MySQL chart 来说，这些值的范围已经减小了，并且删除了名 namespace 前缀，所以它会将密码字段简单地视为 `.Values.password`。

#### 全局值

从 2.0.0-Alpha.2 开始，Helm 支持特殊的 “全局” 值。考虑前面例子的这个修改版本：


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

上面添加了一个 global 区块，值 `app: MyWordPress`。此值可供所有 chart 使用 `.Values.global.app`。

比如，该 mysql 模板可以访问 `app` 如. Values.global.app，apache chart 也同样的。上面的 values 文件是这样高效重新生成的：


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

这提供了一种与所有子 chart 共享一个顶级变量的方法，这对设置 metadata 中像标签这样的属性很有用。

如果子 chart 声明了一个全局变量，则该全局将向下传递 （到子 chart 的子 chart），但不向上传递到父 chart。子 chart 无法影响父 chart 的值。

此外，父 chart 的全局变量优先于子 chart 中的全局变量。

### 参考
当涉及到编写模板和 values 文件时，有几个标准参考可以帮助你。

-   [Go templates](https://godoc.org/text/template)
-   [Extra template functions](https://godoc.org/github.com/Masterminds/sprig)
-   [The YAML format](http://yaml.org/spec/)

## 使用 Helm 管理 chart

该 helm 工具有几个用于处理 chart 的命令。

它可以为你创建一个新的 chart：

```bash
$ helm create mychart
Created mychart/
```

编辑完 chart 后，`helm` 可以将其打包到 chart 压缩包中：

```bash
$ helm package mychart
Archived mychart-0.1.-.tgz
```

可以用 `helm` 来帮助查找 chart 格式或信息的问题：

```bash
$ helm lint mychart
No issues found
```

## Chart repo 库
chart repo 库是容纳一个或多个封装的 chart 的 HTTP 服务器。虽然 helm 可用于管理本地 chart 目录，但在共享 chart 时，首选机制是 chart repo 库。

任何可以提供 YAML 文件和 tar 文件并可以回答 GET 请求的 HTTP 服务器都可以用作 repo 库服务器。

Helm 附带用于开发人员测试的内置服务器（`helm serve`）。Helm 团队测试了其他服务器，包括启用了网站模式的 Google Cloud Storage 以及启用了网站模式的 S3。

repo 库的主要特征是存在一个名为的特殊文件 `index.yaml`，它具有 repo 库提供的所有软件包的列表以及允许检索和验证这些软件包的元数据。

在客户端，repo 库使用 `helm repo` 命令进行管理。但是，Helm 不提供将 chart 上传到远程存储服务器的工具。这是因为这样做会增加部署服务器的需求，从而增加配置 repo 库的难度。

## Chart 起始包

`helm create` 命令采用可选 `--starter` 选项，可以指定 “起始 chart”。

起始 chart 只是普通的 chart，位于 $HELM_HOME/starters。作为 chart 开发人员，可以创作专门设计用作起始的 chart。记住这些 chart 时应考虑以下因素：

-   `Chart.yaml` 将被生成器覆盖。
-   用户将期望修改这样的 chart 内容，因此文档应该指出用户如何做到这一点。
-   所有 templates 目录下的匹配项 `<CHARTNAME>` 将被替换为指定的 chart 名称，以便起始 chart 可用作模板。另外, `values.yaml` 的 `<CHARTNAME>` 也会被替换.
-   目前添加chart的唯一方法是手动将其复制到`$HELM_HOME/starters`。在chart的文档中，你需要解释该过程。
