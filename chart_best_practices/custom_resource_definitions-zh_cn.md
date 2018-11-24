# 自定义资源定义

最佳实践指南的这一部分涉及创建和使用自定义资源定义对象。

使用自定义资源定义（CRD）时，区分两个不同的部分很重要：

- 有一个 CRD 的声明。这是一个 YAML 文件，kind 类型为 `CustomResourceDefinition`
- 然后有资源使用 CRD。CRD 定义 `foo.example.com/v1`。任何拥有 `apiVersion: example.com/v1` 和种类 `Foo` 的资源都是使用 CRD 的资源。

## 在使用资源之前安装 CRD 声明

Helm 优化为尽可能快地将尽可能多的资源加载到 Kubernetes 中。通过设计，Kubernetes 可以采取一整套 manifests，并将它们全部启动在线（这称为 reconciliation 循环）。

但是与 CRD 有所不同。

对于 CRD，声明必须在该 CRDs 种类的任何资源可以使用之前进行注册。注册过程有时需要几秒钟。

### 方法 1：独立的 chart

一种方法是将 CRD 定义放在一个 chart 中，然后将所有使用该 CRD 的资源放入另一个 chart 中。

在这种方法中，每个 chart 必须单独安装。

### 方法 2：预安装 hook

要将这两者打包在一起，在 CRD 定义中添加一个 `pre-install` 钩子，以便在执行 chart 的其余部分之前完全安装它。

请注意，如果使用`pre-install` hook创建CRD ，则该CRD定义在`helm delete`运行时不会被删除。
