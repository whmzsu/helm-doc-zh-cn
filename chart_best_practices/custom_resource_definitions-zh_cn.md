# 自定义资源定义

最佳实践指南的这一部分涉及创建和使用自定义资源定义对象。

使用自定义资源定义（CRD）时，区分两个不同的部分很重要：

- 有一个CRD的声明。这是一个YAML文件，kind类型为`CustomResourceDefinition`
- 然后有资源使用 CRD。CRD定义`foo.example.com/v1`。任何拥有`apiVersion: example.com/v1`和种类`Foo`的资源都是使用CRD的资源。

## 在使用资源之前安装CRD声明

Helm优化为尽可能快地将尽可能多的资源加载到Kubernetes中。通过设计，Kubernetes可以采取一整套manifests，并将它们全部启动在线（这称为reconciliation循环）。

但是与CRD有所不同。

对于CRD，声明必须在该CRDs种类的任何资源可以使用之前进行注册。注册过程有时需要几秒钟。

### 方法1：独立的chart

一种方法是将CRD定义放在一个chart中，然后将所有使用该CRD的资源放入另一个chart中。

在这种方法中，每个chart必须单独安装。

### 方法2：预安装hook

要将这两者打包在一起，在CRD定义中添加一个`pre-install`钩子，以便在执行chart的其余部分之前完全安装它。

请注意，如果使用`pre-install` hook创建CRD ，则该CRD定义在`helm delete`运行时不会被删除。
