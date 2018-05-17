# Chart最佳实践指南

本指南涵盖了Helm Team创建chart的最佳实践。它着重于如何构建chart。

我们主要关注有可能公开部署的chart的最佳实践。我们知道许多chart仅供内部使用，这些chart的作者可能会由于为了其内部利益，可能不拘泥于我们的建议。

## 目录
- [一般约定](conventions-zh_cn.md)：了解一般chart约定。
- [values文件](values-zh_cn.md)：查看结构化`values.yaml`的最佳实践。
- [模板](templates-zh_cn.md):：学习一些编写模板的最佳技巧。
- [需求](requirements-zh_cn.md)：遵循`requirements.yaml`文件的最佳做法。
- [标签和注释](labels-zh_cn.md):：helm具有标签和注释的传统。
- Kubernetes资源：
 - [Pod及其规格](pods-zh_cn.md)：查看使用pod规格的最佳做法。
 - [基于角色的访问控制](rbac-zh_cn.md)：有关创建和使用服务帐户，角色和角色绑定的指导。
 ;- [第三方资源]：第三方资源（TPR）有其自己的相关最佳实践。
