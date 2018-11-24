# 同步 chart 库

** 注意：** 这个样例适用于于提供 chart 库的 Google Cloud Storage（GCS）存储 bucket。

## 前提条件

* 安装 [gsutil](https://cloud.google.com/storage/docs/gsutil) 工具。** 这个样例依赖于 gsutil rsync 功能。**
* 确保有权访问 Helm 客户端文件
* 可选：我们建议在 GCS 存储桶上设置对象版本控制，以防意外删除某些内容。

## 设置本地 chart 库目录

像我们在 [the chart repository guide](chart_repository-zh_cn.md) 中一样创建一个本地目录，并将打包的 chart 放入该目录中。

例如:
```bash
$ mkdir fantastic-charts
$ mv alpine-0.1.0.tgz fantastic-charts/
```

## 生成更新的 index.yaml

使用 Helm 通过将远程存储库的目录路径和 url 传递到 `helm repo index` 命令来生成更新的 index.yaml 文件，如下所示：

```bash
$ helm repo index fantastic-charts/ --url https://fantastic-charts.storage.googleapis.com
```

这将生成一个更新的 index.yaml 文件并放置在 `fantastic-charts/` 目录中。

## 同步本地和远程 chart 库

通过运行 `scripts/sync-repo.sh` 并传入本地目录名称和 GCS 存储桶名称, 将目录的内容上传到您的 GCS 存储桶。

例如：

```bash

$ pwd
/Users/funuser/go/src/github.com/kubernetes/helm
$ scripts/sync-repo.sh fantastic-charts/ fantastic-charts
Getting ready to sync your local directory (fantastic-charts/) to a remote repository at gs://fantastic-charts
Verifying Prerequisites....
Thumbs up! Looks like you have gsutil. Let’s continue.
Building synchronization state...
Starting synchronization
Would copy file://fantastic-charts/alpine-0.1.0.tgz to gs://fantastic-charts/alpine-0.1.0.tgz
Would copy file://fantastic-charts/index.yaml to gs://fantastic-charts/index.yaml
Are you sure you would like to continue with these changes?? [y/N]} y
Building synchronization state...
Starting synchronization
Copying file://fantastic-charts/alpine-0.1.0.tgz [Content-Type=application/x-tar]...
Uploading   gs://fantastic-charts/alpine-0.1.0.tgz:              740 B/740 B
Copying file://fantastic-charts/index.yaml [Content-Type=application/octet-stream]...
Uploading   gs://fantastic-charts/index.yaml:                    347 B/347 B
Congratulations your remote chart repository now matches the contents of fantastic-charts/

```


## 更新 chart 库

你可能需要保留 chart 库内容的本地副本，或者运行 gsutil rsync 将远程 chart 存储库的内容复制到本地目录。

例如：

```bash
$ gsutil rsync -d -n gs://bucket-name local-dir/    # the -n flag does a dry run
Building synchronization state...
Starting synchronization
Would copy gs://bucket-name/alpine-0.1.0.tgz to file://local-dir/alpine-0.1.0.tgz
Would copy gs://bucket-name/index.yaml to file://local-dir/index.yaml

$ gsutil rsync -d gs://bucket-name local-dir/       # performs the copy actions
Building synchronization state...
Starting synchronization
Copying gs://bucket-name/alpine-0.1.0.tgz...
Downloading file://local-dir/alpine-0.1.0.tgz:                        740 B/740 B
Copying gs://bucket-name/index.yaml...
Downloading file://local-dir/index.yaml:                              346 B/346 B
```

有用的网址：

* 文档 [gsutil rsync](https://cloud.google.com/storage/docs/gsutil/commands/rsync#description)
* [Chart 库指南](chart_repository-zh_cn.md)
* 文档[object versioning and concurrency control](https://cloud.google.com/storage/docs/gsutil/addlhelp/ObjectVersioningandConcurrencyControl#overview)
