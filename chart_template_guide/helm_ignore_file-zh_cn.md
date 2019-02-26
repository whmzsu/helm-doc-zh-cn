#.helmignore 文件

`.helmignore` 文件用于指定不想包含在 helm chart 中的文件。

如果此文件存在，`helm package` 命令将在打包应用程序时忽略在 `.helmignore` 文件中指定的模式匹配的所有文件。

这有助于避免在 helmchart 中添加不需要或敏感的文件或目录。

`.helmignore` 文件支持 Unix shell glob 匹配，相对路径匹配和否定（以！为前缀）。每行只考虑一种模式。

这是一个示例 `.helmignore` 文件：

```
# comment
.git
*/temp*
*/*/temp*
temp?
```

** 我们非常欢迎你的帮助 ** 改进本文档。添加，更正或删除信息，[提出问题](https://github.com/helm/helm/issues) 或向我们发送 PR。
**如果对中文版的文档有错误，或者想改进优化** 请[访问此链接](https://github.com/whmzsu/helm-doc-zh-cn/issues)。
