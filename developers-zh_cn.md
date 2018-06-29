# 开发者指南

本指南解释了如何设置环境来开发 Helm 和 Tiller。

## 前提条件
- Go 的最新版本
- 最新版本的 Glide
- Kubernetes 集群和 kubectl（可选）
- gRPC 工具链
- Git

## 构建 Helm/tiller

我们使用 Make 来构建我们的程序。最简单的入门方法是：

```bash
$ make bootstrap build
```



注意：如果不从路径 `$GOPATH/src/k8s.io/helm` 运行，命令将会失败。目录 k8s.io 不应该是符号链接，否则 build 找不到相关的包。

这将构建 Helm 和 Tiller。`make bootstrap` 将尝试安装某些工具，如果它们不存在的话。

要运行所有测试（无需运行测试 `vendor/`），请运行 make test。

要在本地运行 Helm 和 Tiller，可以运行 `bin/helm` 或 `bin/tiller`。

- 已知 Helm 和 Tiller 可在 macOS 和大多数 Linux，包括 Alpine 上运行。
- Tiller 必须能够访问 Kubernetes 群集。它通过检查使用 kubectl 的 Kube 配置文件来了解群集信息。

### 手册页

手册页和 Markdown 文档已经在 `docs/` 预先建好。可以使用重新生成文档 make docs。

要将 Helm 手册页公开给 `man` 客户端，您可以将这些文件放入 `$MANPATH`：

```
$ export MANPATH=$GOPATH/src/k8s.io/helm/docs/man:$MANPATH
$ man helm
```

## gRPC 和 Protobuf

Helm 和 Tiller 使用 gRPC 进行通信。要开始使用 gRPC，需要如下准备

- 安装 protoc 编译 protobuf 文件。发布在 [这里](https://github.com/google/protobuf/releases)
- 运行 Helm 的 `make bootstrap` 来生成 p`rotoc-gen-go` 插件并将其放入 `bin/`。

请注意，需要使用 protobuf 3.2.0（`protoc --version`）的版本。protoc-gen-go 版本与 Kubernetes 使用 GRPC 的版本绑定。所以这个插件是在本地维护的。

虽然 gRPC 和 ProtoBuf 规范缩进时没有规定，但我们要求缩进样式与 Go 格式规范相匹配。也就是说，协议缓冲区应该使用基于标签的缩进，并且 rpc 声明应该遵循 Go 函数声明的风格。

### Helm API（HAPI）

我们使用 gRPC 作为 API 层。请参阅 `pkg/proto/hapi` 生成的 Go 代码以及 `_proto` 协议缓冲区定义。

要从 protobuf 源重新生成 Go 文件，使用 `make protoc`。

## Docker 镜像

要构建 Docker 镜像，请使用 `make docker-build`。

预构建的镜像已经在官方 Kubernetes Helm GCR registry 中提供。

## 运行本地群集

对于开发，我们强烈建议使用 [Kubernetes Minikube](https://github.com/kubernetes/minikube) 这个面向开发人员的发行版。安装完成后，可以使用 helm init 安装到群集中。请注意，用于开发的 Tiller 版本可能无法在 Google Cloud Container Registry 中使用。如果遇到镜像 Pull 错误，可以覆盖 Tiller 的版本。例：

```bash
helm init --tiller-image=gcr.io/kubernetes-helm/tiller:2.7.2
```

或使用最新版本：

```bash
helm init --canary-image
```

为了在 Tiller 上进行开发，在本地运行 Tiller 有时更方便，而不是将其打包到镜像中并在群集中运行。你可以通过告诉 Helm 客户端使用一个本地实例来做到这一点。

```bash
$ make build
$ bin/tiller
```

要配置 Helm 客户端，请使用 `--host` 标志或导出 `HELM_HOST` 环境变量：

```bash
$ export HELM_HOST=localhost:44134
$ helm install foo
```

（请注意，直接运行 Tiller 时不需要使用 `helm init`）

Tiller 应该在 >= 1.3 Kubernetes 群集上运行。

## 贡献指南

我们欢迎捐款。这个项目已经制定了一些指导方针，以确保（a）代码质量仍然很高，（b）项目保持一致，（c）贡献遵循开源法律要求。我们的目的不是为了给贡献者带来负担，而是为了构建优雅和高质量的开源代码，让我们的用户受益。

确保你已经阅读并理解了贡献指南：

https://github.com/kubernetes/helm/blob/master/CONTRIBUTING.md

### 代码的结构

Helm 项目的代码组织如下：

- 单独的程序位于 `cmd/`。`cmd/` 里面的代码 不是为库的重复使用而设计的。
- 共享库存储在 `pkg/`。
- 原始 ProtoBuf 文件存储在 `_proto/hapi`（hapi 表示 Helm 应用程序编程接口）。
- 从 proto 定义生成的 Go 文件存储在 `pkg/proto`。
- `scripts/` 目录包含许多实用程序脚本。其中大部分由 CI/CD 管道使用。
- `rootfs/` 文件夹用于 Docker 特定的文件。
- `docs/` 文件夹用于文档和示例。

Go 依赖关系由 Glide 管理 并存储在 `vendor/` 目录中。

### Git 约定

我们将 Git 用于版本控制系统。Master 分支是当前发展候选目录。发布版本有标记。

我们通过 GitHub Pull Requests（PR）接受对代码的更改。执行此操作的一个工作流程如下所示：

1. 转到 `$GOPATH/src/k8s.io` 目录，`git clone` `github.com/kubernetes/helm` 存储库。
2. 将该存储库 fork 到你自己的 GitHub 帐户
3. 将你的存储库添加为远程服务器 `$GOPATH/src/k8s.io/helm`
4. 创建一个新的工作分支（`git checkout -b feat/my-feature`）并在该分支上完成工作。
5. 当准备好让我们 review 时，将你的分支推送到 GitHub，然后给我们提交一个新的 PR 请求。

对于 Git 提交消息，我们遵循语义提交消息 [Semantic Commit Messages](http://karma-runner.github.io/0.13/dev/git-commit-msg.html)：

```
fix(helm): add --foo flag to 'helm install'

When 'helm install --foo bar' is run, this will print "foo" in the
output regardless of the outcome of the installation.

Closes #1234
```

常见提交类型：

- fix: 修复一个 bug 或错误
- feat: 添加一项新功能
- docs: 更改文档
- test: 改进测试
- ref: 重构现有的代码

常见范围：

- helm：Helm CLI
- tiller：Tiller 服务器
- proto：Protobuf 定义
- pkg/lint：lint 包。对于任何软件包遵循类似的约定
- `*`：两个或更多范围


更多内容：

- DEIS 准则 [Deis Guidelines](https://github.com/deis/workflow/blob/master/src/contributing/submitting-a-pull-request.md) 是这一部分的灵感启发。
- Karma Runner [定义](http://karma-runner.github.io/0.13/dev/git-commit-msg.html) 了语义提交消息的主意。

### Go 语言约定

我们非常密切地遵循 Go 编码风格标准。通常，运行 `go fmt` 会让你的代码更加漂亮。

我们通常也遵循由 `go lint` 和 `gometalinter` 推荐的约定。运行 `make test-style` 以测试样式一致性。

更多阅读：

- Effective Go [introduces formatting](https://golang.org/doc/effective_go.html#formatting)。
- Go Wiki 有一个关于格式化的非常好的文章 [formatting](https://github.com/golang/go/wiki/CodeReviewComments).。

### Protobuf 约定

由于项目主要是 Go 代码，因此我们尽可能将 Protobuf 文件格式化为 Go。目前 Protobuf 没有真正的格式规则或准则，但随着它们的出现，我们可能会选择遵循这些规则或准则。

标准：

- Tab 缩进，而不是空格。
- 间距规则遵循 Go 约定（线条末端的花括号，运算符周围的空格）。

约定：

- 文件应该用 `option go_package = "...";` 来指定它们的包
- 注释应该转化为良好的 Go 代码注释（因为 `protoc ` 会拷贝注释到目标源代码文件中）。
- RPC 功能在它们的请求 / 响应消息的同一文件中定义。
- 不推荐使用的RPC，消息和字段在注释中不推荐使用（// UpdateFoo DEPRECATED updates a foo.）。
