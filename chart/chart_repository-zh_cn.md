# Chart Repository 存储库指南

本节介绍如何创建和使用 Helm chart repo。在高层次上，chart 库是可以存储和共享打包 chart 的位置。

官方 chart 库由 [Kubernetes Charts](https://github.com/kubernetes/charts) 维护 ，我们欢迎参与贡献。Helm 还可以轻松创建和运行自己的 chart 库。本指南讲解了如何做到这一点。

## 前提条件

* 阅读快速入门指南 [Quickstart](quickstart.md)
* 通读 chart 文件 [Charts](charts.md)

## 创建 chart 库

chart 库是带有一个 index.yaml 文件和任意个打包 cahrt 的 HTTP 服务器。当准备好分享 chart 时，首选方法是将其上传到 chart 库。

** 注意：** 对于 Helm 2.0.0，chart 库没有任何内部认证。 在 GitHub 中有一个跟踪进度的问题 [issue tracking progress](https://github.com/helm/issues/1038)。

由于 chart 库可以是任何可以提供 YAML 和 tar 文件并可以回答 GET 请求的 HTTP 服务器，因此当托管自己的 chart 库时，很多选择。例如，可以使用 Google 云端存储（GCS）存储桶，Amazon S3 存储桶，Github Pages，甚至可以创建自己的 Web 服务器。

### chart 库结构

chart 库由打包的 chart 和一个名为的特殊文件组成， index.yaml 其中包含 chart 库中所有 chart 的索引。通常，index.yaml 描述的 chart 也是托管在同一台服务器上，源代码文件也是如此。

例如，chart 库的布局 https://example.com/charts 可能如下所示：

```
charts/
  |
  |- index.yaml
  |
  |- alpine-0.1.2.tgz
  |
  |- alpine-0.1.2.tgz.prov
```

这种情况下，索引文件包含有关一个 chart（Alpine chart）的信息，并提供该 chart 的下载 URL `https://example.com/charts/alpine-0.1.2.tgz`。

不要求 chart 包与 index.yaml 文件位于同一台服务器上 。但是，发在一起这样做通常是最简单的。

### 索引文件
索引文件是一个叫做 yaml 文件 index.yaml。它包含一些关于包的元数据，包括 chart 的 Chart.yaml 文件的内容。一个有效的 chart 库必须有一个索引文件。索引文件包含有关 chart 库中每个 chart 的信息。`helm repo index` 命令将根据包含打包的 chart 的给定本地目录生成索引文件。

下面一个索引文件的例子：

```
apiVersion: v1
entries:
  alpine:
    - created: 2016-10-06T16:23:20.499814565-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 99c76e403d752c84ead610644d4b1c2f2b453a74b921f422b9dcb8a7c8b559cd
      home: https://k8s.io/helm
      name: alpine
      sources:
      - https://github.com/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.2.0.tgz
      version: 0.2.0
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 515c58e5f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cd78727
      home: https://k8s.io/helm
      name: alpine
      sources:
      - https://github.com/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.1.0.tgz
      version: 0.1.0
  nginx:
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Create a basic nginx HTTP server
      digest: aaff4545f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cdffffff
      home: https://k8s.io/helm
      name: nginx
      sources:
      - https://github.com/kubernetes/charts
      urls:
      - https://technosophos.github.io/tscharts/nginx-1.1.0.tgz
      version: 1.1.0
generated: 2016-10-06T16:23:20.499029981-06:00
```

生成的索引和包可以从基本的网络服务器提供。可以使用 helm serve 启动本地服务器，在本地测试所有内容。

```bash
$ helm serve --repo-path ./charts
Regenerating index. This may take a moment.
Now serving you on 127.0.0.1:8879
```

上面启动了一个本地 web 服务器，为它在 `./charts` 目录找到的 chart 提供服务。serve 命令将在启动过程中自动生成一个 `index.yaml` 文件。

## 托管 chart 库

本部分介绍了提供 chart 库的几种方法。

### Google 云端存储

第一步是创建 GCS 存储桶。我们会给我们称之为 `fantastic-charts`。

创建一个 GCS 桶

![Create a GCS Bucket](../images/create-a-bucket.png)

接下来，通过 ** 编辑存储桶权限 ** 使存储桶公开。

编辑权限

![Edit Permissions](../images/edit-permissions.png)

插入此行 item 来 ** 公开存储 bucket**：

开放 Bucket

![Make Bucket Public](../images/make-bucket-public.png)

恭喜，现在你有一个空的 GCS bucket 准备好给 chart 提供服务！

可以使用 Google Cloud Storage 命令行工具或使用 GCS Web UI 上传 chart 库。这是官方 Kubernetes Charts 存储库托管其 chart 的技术，因此如果遇到困难，可能需要查看该项目 [peek at that project](https://github.com/kubernetes/charts) 。

** 注意：** 可以通过此处的 HTTPS 地址方便的访问公开的 GCS 存储桶 `https://bucket-name.storage.googleapis.com/`。

### JFrog Artifactory

还可以使用 JFrog Artifactory 来设置 chart 库。在 [此处](https://www.jfrog.com/confluence/display/RTF/Helm+Chart+Repositories) 阅读更多关于 JFrog Artifactory 和 chart 库的信息

### Github Pages 示例

以类似的方式，可以使用 GitHub Pages 创建 chart 库。

GitHub 允许两种不同的方式提供静态网页：

- 通过配置一个项目来提供其 `docs/`` 目录的内容
- 通过配置一个项目来为特定分支提供服务

我们会采取第二种方法，尽管第一种方法很简单。

第一步是创建你的 gh-pages 分支。可以在本地做到这一点。

```bash
$ git checkout -b gh-pages
```

或者通过使用网络浏览器在 Github 存储库上的分支按钮：

![Create Github Pages branch](../images/create-a-gh-page-button.png)

接下来，需要确保 gh-pages 分支设置为 Github Pages，点击 repo Settings 并向下滚动到 Github Pages 部分并按照以下设置：

![Create Github Pages branch](../images/set-a-gh-page.png)


默认情况下，源通常设置为 gh-pages 分支。如果这不是默认设置，那么请选择 gh-pages。

也可以在那里使用自定义域名。

并检查是否勾选了强制使用 HTTPS，以便在提供 chart 时使用 HTTPS。

这样的设置中，可以使用 master 分支来存储 chart 代码，并将 gh-pages 分支作为 chart 库，例如： `https://USERNAME.github.io/REPONAME`。演示 [TS Charts](https://github.com/technosophos/tscharts) 库可以通过 `https://technosophos.github.io/tscharts/` 访问。

### 普通的 web 服务器

要配置普通 Web 服务器来服务 Helm chart，只需执行以下操作：

- 将索引和 chart 置于服务器目录中
- 确保 `index.yaml` 可以在没有认证要求的情况下访问
- 确保 `yaml` 文件的正确内容类型（`text/yaml` 或 `text/x-yaml`）

例如，如果想在 `$WEBROOT/charts` 以外的目录为 chart 提供服务，请确保 Web 根目录中有一个 `charts/` 目录，并将索引文件和 chart 放入该文件夹内。

## 管理 chart 库
现在已有一个 chart 存储库，本指南的最后一部分将介绍如何维护该库中的 chart。

### 将 chart 存储在 chart 库中

现在已有一个 chart 存储库，让我们上传一个 chart 和一个索引文件到存储库。chart 库中的 chart 必须正确打包（`helm package chart-name/`）和版本（遵循 [SemVer 2](https://semver.org/) 标准）。

接下来的这些步骤是一个示例工作流程，也可以用你喜欢的任何工作流程来存储和更新 chart 库中的 chart。

准备好打包 chart 后，创建一个新目录，并将打包 chart 移动到该目录。

```bash
$ helm package docs/examples/alpine/
$ mkdir fantastic-charts
$ mv alpine-0.1.0.tgz fantastic-charts/
$ helm repo index fantastic-charts --url https://fantastic-charts.storage.googleapis.com
```

最后一条命令采用刚创建的本地目录的路径和远程 chart 库的 URL，并在给定的目录路径中生成 `index.yaml`。

现在可以使用同步工具或手动将 chart 和索引文件上传到 chart 库。如果使用 Google 云端存储，请使用 gsutil 客户端查看此示例工作流程。对于 GitHub，可以简单地将 chart 放入适当的目标分支中。

### 新添加 chart 添加到现有存储库

每次将新 chart 添加到存储库时，都必须重新生成索引。`helm repo index` 命令将 `index.yaml` 从头开始完全重建该文件，但仅包括它在本地找到的 chart。

可以使用 `--merge` 标志向现有 `index.yaml` 文件增量添加新 chart（在使用远程存储库（如 GCS）时，这是一个很好的选择）。运行 `helm repo index --help` 以了解更多信息，

确保上传修改后的 index.yaml 文件和 chart。如果生成了出处 provenance 文件，也要上传。

### 与他人分享 chart

准备好分享 chart 时，只需让别人知道存储库的 URL 是什么就可以了。

他们将通过 `helm repo add [NAME] [URL]` 命令将仓库添加到他们的 helm 客户端，并可以起一个带有任何想用来引用仓库的名字。

```bash
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com
$ helm repo list
fantastic-charts    https://fantastic-charts.storage.googleapis.com
```

如果 chart 由 HTTP 基本认证支持，也可以在此处提供用户名和密码：

```bash
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com --username my-username --password my-password
$ helm repo list
fantastic-charts    https://fantastic-charts.storage.googleapis.com
```

** 注意：** 如果存储库不包含有效信息库 `index.yaml` 文件，则添加不会成功。

之后，用户将能够搜索 chart。更新存储库后，他们可以使用该 helm repo update 命令获取最新的 chart 信息。

原理是`helm repo add`和`helm repo update`命令获取index.yaml文件并将它们存储在 `$HELM_HOME/repository/cache/`目录中。这是helm search 找到有关chart的信息的地方。
