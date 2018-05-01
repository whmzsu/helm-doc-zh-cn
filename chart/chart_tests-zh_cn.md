# Chart测试

一个chart包含许多一起工作的Kubernetes资源和组件。作为chart作者，可能需要编写一些测试来验证chart在安装时是否按预期工作。这些测试还有助于chart消费者了解chart应该做什么。

**测试**在Helm chart中的templates/目录，是一个pod定义，指定一个给定的命令来运行容器。容器应该成功退出（exit 0），测试被认为是成功的。该pod定义必须包含helm测试hook注释之一：`helm.sh/hook: test-success`或`helm.sh/hook: test-failure`。

示例测试：

- 验证来自values.yaml文件的配置是否正确注入。
- 确保用户名和密码正常工作
- 确保不正确的用户名和密码不起作用
- 断言服务已启动并正确进行负载平衡
- 等等

可以使用该helm test <RELEASE_NAME>命令在release中运行Helm中的预定义测试。对于chart使用者来说，这是一种很好的方式来检查他们发布的chart（或应用程序）是否按预期工作。

## Helm测试hook的分解

在Helm中，有两个测试hook：`test-success`和`test-failure`.

`test-success`表示测试pod应该成功完成。换句话说，容器中的容器应该exit 0. `test-failure`是一种断言测试容器不能成功完成的方式。如果pod中的容器未exit 0，则表示成功。

## 示例测试

下面是一个示例mariadb chart中helm测试pod定义的示例：

```
mariadb/
  Chart.yaml
  README.md
  values.yaml
  charts/
  templates/
  templates/tests/test-mariadb-connection.yaml
```

在`wordpress/templates/tests/test-mariadb-connection.yaml`中：

```
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-credentials-test"
  annotations:
    "helm.sh/hook": test-success
spec:
  containers:
  - name: {{ .Release.Name }}-credentials-test
    image: {{ .Values.image }}
    env:
      - name: MARIADB_HOST
        value: {{ template "mariadb.fullname" . }}
      - name: MARIADB_PORT
        value: "3306"
      - name: WORDPRESS_DATABASE_NAME
        value: {{ default "" .Values.mariadb.mariadbDatabase | quote }}
      - name: WORDPRESS_DATABASE_USER
        value: {{ default "" .Values.mariadb.mariadbUser | quote }}
      - name: WORDPRESS_DATABASE_PASSWORD
        valueFrom:
          secretKeyRef:
            name: {{ template "mariadb.fullname" . }}
            key: mariadb-password
    command: ["sh", "-c", "mysql --host=$MARIADB_HOST --port=$MARIADB_PORT --user=$WORDPRESS_DATABASE_USER --password=$WORDPRESS_DATABASE_PASSWORD"]
  restartPolicy: Never
```

## 在release上运行测试套件的步骤

1. `$ helm install mariadb`
```
NAME:   quirky-walrus
LAST DEPLOYED: Mon Feb 13 13:50:43 2017
NAMESPACE: default
STATUS: DEPLOYED
```

2. `$ helm test quirky-walrus`
```
RUNNING: quirky-walrus-credentials-test
SUCCESS: quirky-walrus-credentials-test
```

## 注意
- 可以在单个yaml文件中定义尽可能多的测试，也可以在templates/目录中的多个yaml文件中进行分布测试
- 提倡将测试套件嵌入到一个`tests/`目录下，比如`<chart-name>/templates/tests/`以便实现更多隔离
