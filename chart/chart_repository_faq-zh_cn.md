# Chart库：常见问题

本节跟踪使用chart库时的一些常遇到的问题。

我们很乐意提供更好的帮助。要添加，更正或删除信息，提出问题或向我们发送PR。

## Fetching

**问：当我试图从我的自定义repo中获取chart时，为什么会出现错误`unsupported protocol scheme` ?**

答:(helm 版本小于2.5.0）这很可能是由于创建chart索引而未指定`--url`参数。尝试使用类似命令`helm repo index --url http://my-repo/charts`重建`index.yaml`，然后将其重新上传到自定义chart repo库。

这个问题在Helm 2.5.0中进行了更改。
