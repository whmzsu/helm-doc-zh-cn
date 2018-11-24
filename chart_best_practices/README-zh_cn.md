# Chart 最佳实践指南

本指南涵盖了 Helm Team 创建 chart 的最佳实践。它着重于如何构建 chart。

我们主要关注有可能公开部署的 chart 的最佳实践。我们知道许多 chart 仅供内部使用，这些 chart 的作者可能会由于为了其内部利益，可能不拘泥于我们的建议。

## 目录
- [一般约定](conventions-zh_cn.md)：了解 chart 一般约定。
- [values 文件](values-zh_cn.md)：查看结构化 `values.yaml` 的最佳实践。
- [Template](templates-zh_cn.md):：学习一些编写模板的最佳技巧。
- [Requirement](requirements-zh_cn.md)：遵循 `requirements.yaml` 文件的最佳做法。
- [标签和注释](labels-zh_cn.md):：helm 具有标签和注释的传统。
- Kubernetes 资源：
  - [Pod 及其规格](pods-zh_cn.md)：查看使用 pod 规格的最佳做法。
  - [基于角色的访问控制](rbac-zh_cn.md)：有关创建和使用服务帐户，角色和角色绑定的指导。
  - [自定义资源](custom_resource_definitions-zh_cn.md)：自定义资源（CRDs）有其自己的相关最佳实践。
