# 基于角色的访问控制

最佳实践指南的这一部分讨论了chart清单中RBAC资源的创建和格式化。

RBAC资源是：

- ServiceAccount (namespaced)
- Role (namespaced)
- ClusterRole
- RoleBinding (namespaced)
- ClusterRoleBinding

## YAML配置
RBAC和ServiceAccount配置应该在单独的密钥下进行。他们是不同的东西。将YAML中的这两个概念拆分出来可以消除混淆并使其更清晰。


```yaml
rbac:
  # Specifies whether RBAC resources should be created
  create: true

serviceAccount:
  # Specifies whether a ServiceAccount should be created
  create: true
  # The name of the ServiceAccount to use.
  # If not set and create is true, a name is generated using the fullname template
  name
```

此结构可以扩展到需要多个ServiceAccounts的更复杂的chart。

```yaml
serviceAccounts:
  client:
    create: true
    name:
  server:
    create: true
    name:
```

## RBAC资源应该默认创建

`rbac.create`应该是一个布尔值，控制是否创建RBAC资源。默认应该是`true`。想要管理RBAC访问控制的用户可以将此值设置为`false`（在这种情况下请参阅下文）。

## 使用RBAC资源
`serviceAccount.name`应设置为由chart创建的访问控制资源使用的S`erviceAccount`的名称。如果`serviceAccount.create`为true，则应该创建一个带有该名称的S`erviceAccount`。如果名称未设置，则使用该`fullname`模板生成名称，如果`serviceAccount.create`为false，则不应创建该名称，但它仍应与相同的资源相关联，以便稍后通过手动创建的RBAC资源将引用它从而功能正常。如果`serviceAccount.create`为false且名称未指定，则使用默认的ServiceAccount。

为ServiceAccount使用以下helper模板。

```yaml
{{/*
Create the name of the service account to use
*/}}
{{- define "mychart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create -}}
    {{ default (include "mychart.fullname" .) .Values.serviceAccount.name }}
{{- else -}}
    {{ default "default" .Values.serviceAccount.name }}
{{- end -}}
{{- end -}}
```
