## 项目的历史
Kubernetes Helm是[Helm
Classic](https://github.com/helm/helm)和GCS Deployment Manager的Kubernetes移植的合并结果。该项目由Google和Deis联合创建，尽管它现在是CNCF的一部分。许多公司现在定期为Helm做出贡献。

与Helm Classic的区别：

- Helm现在有一个客户端（`helm`）和一个服务端（`tiller`）。服务端在Kubernetes内部运行，并管理资源。
- Helm的chart格式更好：
  - 依赖关系是不可变的，并存储在chart的`charts/` 目录中。
  - chart使用[SemVer 2](http://semver.org/spec/v2.0.0.html)版本标准化
  - chart可以从目录或chart归档文件中加载
  - Helm支持Go模板，无需运行`generate` 或`template`命令。
  - Helm可以轻松配置release - 并与团队的其他成员共享配置。
- Helm chart存储库现在使用普通的HTTP（S）而不是Git / GitHub。不再有任何GitHub依赖。
  - chart存储库服务是一个简单的HTTP服务器
  - chart由版本引用
  - `helm serve`命令将运行本地chart服务器，也可以轻松使用对象存储（S3，GCS）或常规Web服务器。
  - 而且仍然可以从本地目录加载chart。
- Helm工作空间不存在了。现在可以在想要工作的文件系统上的任何地方工作。
