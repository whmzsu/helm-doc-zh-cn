# 总结
本指南旨在为你提供chart开发人员对如何使用Helm模板语言的深入了解。本指南着重介绍模板开发的技术方面。

但是当谈到chart的实际日常开发时，本指南还没有涉及很多事情。以下是一些有用的指向其他文档的指南，这些指南将帮助您创建新chart：

- Kubernetes chart项目[Kubernetes Charts project](https://github.com/kubernetes/charts) 是chart不可缺少的来源。该项目也是chart开发中最佳实践的标准。
- Kubernetes用户指南[User's Guide](http://kubernetes.io/docs/user-guide/)提供了可以使用的各种资源类型的详细示例，从ConfigMaps和Secrets到DaemonSetkubernetes和Deployments。
- Helm chart指南[Charts Guide](../chart/charts-zh_cn.md)介绍了使用chart的工作流程。
- Helm Chart Hooks指南[Chart Hooks Guide](../chart/charts_hooks-zh_cn.md)解释了如何创建生命周期hook。
- Helm chart技巧和窍门文章[Chart Hooks Guide](../chart/charts_hooks-zh_cn.md) 提供了一些写chart的有用技巧。
- [Sprig documentation](https://github.com/Masterminds/sprig) 文档介绍了提供了六十余的模板功能。
- 在Go模板[Go template docs](https://godoc.org/text/template) 文档详细解释模板语法。
- Schelm工具[Schelm tool](https://github.com/databus23/schelm) 是用于调试chart一个很好的帮手工具。

有时候，问几个问题,并从经验丰富的开发人员那里获得答案会更容易。最好的地方是在Kubernetes `#Helm`Slack频道：

- [Kubernetes Slack](https://slack.k8s.io/): `#helm`

最后，如果在本文中发现错误或遗漏，想要推荐一些新内容或希望参与，请访问Helm项目[The Helm Project](https://github.com/kubernetes/helm)。
