## 项目的历史
Kubernetes Helm 是 [Helm
Classic](https://github.com/helm/helm) 和 GCS Deployment Manager 的 Kubernetes 移植的合并结果。该项目由 Google 和 Deis 联合创建，尽管它现在是 CNCF 的一部分。许多公司现在定期为 Helm 做出贡献。

与 Helm Classic 的区别：

- Helm 现在有一个客户端（`helm`）和一个服务端（`tiller`）。服务端在 Kubernetes 内部运行，并管理资源。
- Helm 的 chart 格式更好：
  - 依赖关系是不可变的，并存储在 chart 的 `charts/` 目录中。
  - chart 使用 [SemVer 2](https://semver.org/spec/v2.0.0.html) 版本标准化
  - chart 可以从目录或 chart 归档文件中加载
  - Helm 支持 Go 模板，无需运行 `generate` 或 `template` 命令。
  - Helm 可以轻松配置 release - 并与团队的其他成员共享配置。
- Helm chart 存储库现在使用普通的 HTTP（S）而不是 Git / GitHub。不再有任何 GitHub 依赖。
  - chart 存储库服务是一个简单的 HTTP 服务器
  - chart 由版本引用
  - `helm serve` 命令将运行本地 chart 服务器，也可以轻松使用对象存储（S3，GCS）或常规 Web 服务器。
  - 而且仍然可以从本地目录加载 chart。
- Helm工作空间不存在了。现在可以在想要工作的文件系统上的任何地方工作。
