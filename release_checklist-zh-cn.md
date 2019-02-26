# 版本 Checklist
（感觉很通用，挺适合左右社区合作开发的软件项目）
** IMPORTANT 重要 ** : 如果你的经历与此文档不同, 请更新此文档以保持最新.

## 版本会
作为版本过程的一部分，每周开发者的两个电话会议将被选为 “版本会”。

### 开始版本周期

版本后的第一个开发人员电话会议将用作版本会议，以启动下一个版本周期。在此会议期间，必须确定以下项目：

- 版本日期
- 此版本的目标 / 目标
- 版本经理（谁来剪彩（冻结？））
- 社区的任何其他重要细节

所有这些信息都应添加到给定版本的 GitHub 里程碑中。在选择是否向特定版本添加问题和 PR 时，这应该为社区和维护者提供一套明确的指导原则。

### 结束（几乎）版本周期

最接近计划版本日期前两周的开发人员电话会议将用于审查应该推送到版本中的任何剩余的 PR。这是一个讨论我们是否应该在冻结版本和任何其他问题之前等待的地方。在本次会议结束时，如果版本日期尚未推出，则应该冻结第一个 RC。在此会议和版本日期之间的后续开发人员电话会议应该留出一些时间来查看是否发现了任何错误。达到版本日期后，可以冻结最终版本。

## 维护者发布 Helm 指南

如果你负责 Helm 的版本发布？？太棒了，下面是需要做的事情
![TODO: Nothing](images/nothing.png)

开玩笑的哈 :trollface:

所有版本的格式均为 vX.YZ，其中 X 是主版本号，Y 是次版本号，Z 是补丁版本号。该项目严格遵循 [semantic versioning](https://semver.org/)，因此遵循此步骤至关重要。

请务必注意，本文档假定存储库中与 “https://github.com/helm/helm” 对应的 git 远程名称为 “upstream”。如果不是（例如，如果您选择将其命名为 “origin” 或类似名称），请务必相应地调整列出的本地环境片段。如果不确定上游远程命名的是什么，请使用命令 git remote -v 来查找。

如果没有远端上游，可以使用以下方法轻松添加：

```shell
git remote add upstream git@github.com:helm/helm.git
```

在本文档中，我们将引用一些环境变量，你可能希望为方便起见而设置这些变量。对于主要 / 次要版本，请使用以下内容：

```shell
export RELEASE_NAME=vX.Y.0
export RELEASE_BRANCH_NAME="release-X.Y"
export RELEASE_CANDIDATE_NAME="$RELEASE_NAME-rc.1"
```

如果要创建修补程序版本，则可能需要使用以下代码：

```shell
export PREVIOUS_PATCH_RELEASE=vX.Y.Z
export RELEASE_NAME=vX.Y.Z+1
export RELEASE_BRANCH_NAME="release-X.Y"
export RELEASE_CANDIDATE_NAME="$RELEASE_NAME-rc.1"
```

## 1. 创建版本分支

### 主要 / 次要版本


主要版本用于新功能添加和行为更改，从而 * 破坏了向后兼容性 * 。次要版本适用于不会破坏向后兼容性的新功能添加。要创建主要版本或次要版本，请首先从 master 创建分支 release-vX.Y.0 。

```shell
git fetch upstream
git checkout upstream/master
git checkout -b $RELEASE_BRANCH_NAME
```

这个新的分支将成为版本的基础，我们将在稍后进行迭代。

### 补丁版本

补丁版本是现有版本的一些关键修复程序。首先从最新的补丁版本创建分支 release-vX.Y.Z。

```shell
git fetch upstream --tags
git checkout $PREVIOUS_PATCH_RELEASE
git checkout -b $RELEASE_BRANCH_NAME
```

从这里，我们可以挑选我们想要在补丁版本中的提交：

```shell
# get the commits ids we want to cherry-pick
git log --oneline
# cherry-pick the commits starting from the oldest one, without including merge commits
git cherry-pick -x <commit-id>
git cherry-pick -x <commit-id>
```

这个新的分支将成为发布的基础，我们将在稍后进行迭代。

## 2. 更改 Git 中的版本号


在进行次要发布时，请确保使用新版本更新 pkg/version/version.go。

```
$ git diff pkg/version/version.go
diff --git a/pkg/version/version.go b/pkg/version/version.go
index 2109a0a..6f5a1a4 100644
--- a/pkg/version/version.go
+++ b/pkg/version/version.go
@@ -26,7 +26,7 @@ var (
        // Increment major number for new feature additions and behavioral changes.
        // Increment minor number for bug fixes and performance enhancements.
        // Increment patch number for critical fixes to existing releases.
-       Version = "v2.6"
+       Version = "v2.7"

        // BuildMetadata is extra build time data
        BuildMetadata = "unreleased"
```

```
git add .
git commit -m "bump version to $RELEASE_CANDIDATE_NAME"
```

## 3. 提交并推送发布分支

为了让其他人开始测试，我们现在可以将发布分支推送到上游并开始测试过程.

```shell
git push upstream $RELEASE_BRANCH_NAME
```

确保在 [helm on CircleCI](https://circleci.com/gh/helm/helm) 上检查 helm 并确保在继续之前版本通过了 CI。

如果有其他人，请让其他人对该分支进行同行审查，然后再继续确保已完成所有正确的更改并且所有版本的提交已完成。

## 4. 创建候选发布版

现在发布分支已经准备就绪，现在是时候开始创建和迭代发布候选版本了。

```shell
git tag --sign --annotate "${RELEASE_CANDIDATE_NAME}" --message "Helm release ${RELEASE_CANDIDATE_NAME}"
git push upstream $RELEASE_CANDIDATE_NAME
```

CircleCI 将自动创建标记的发布镜像和客户端二进制文件以进行测试。

对于测试人员，在 CircleCI 完成构建工件之后开始测试的过程涉及以下步骤从 Google 云端存储中获取客户端的：

linux/amd64，使用 / bin/bash：

```shell
wget https://kubernetes-helm.storage.googleapis.com/helm-$RELEASE_CANDIDATE_NAME-linux-amd64.tar.gz
```

darwin/amd64, 使用 Terminal.app:

```shell
wget https://kubernetes-helm.storage.googleapis.com/helm-$RELEASE_CANDIDATE_NAME-darwin-amd64.tar.gz
```

windows/amd64, 使用 PowerShell:

```shell
PS C:\> Invoke-WebRequest -Uri "https://kubernetes-helm.storage.googleapis.com/helm-$RELEASE_CANDIDATE_NAME-windows-amd64.zip" -OutFile "helm-$ReleaseCandidateName-windows-amd64.zip"
```

然后，解压缩并将二进制文件移动到 $PATH 上的某个位置，或将其移动到某处并将其添加到 $PATH（例如 / usr/ local/bin/helm 用于 linux macOS，C\Program Files\helm\helm.exe 用于 Windows）。

## 5. 迭代连续发布的候选版本

花几天时间明确投入时间和资源，尝试以各种可能的方式破坏 helm，记录与版本相关的任何发现。这个时间应该用于测试并找到版可能导致各种功能或升级环境出现问题的方式，而不是编码。在此期间，该版本将在代码冻结中，并且任何其他代码更改将被推送到下一个版本。

在此阶段，$RELEASE_BRANCH_NAME 分支将继续发展，因为将生成新的候选版本。新候选版本的频率取决于版本经理：使用最佳判断，考虑报告问题的严重性，测试人员的可用性以及版本截止日期。一般来说，最好让版本时间超过截止日期，而不是发布不可靠版本。

Each time you'll want to produce a new release candidate, you will start by
adding commits to the branch by cherry-picking from master:

每次你想要创建新的候选版本时，可以通过从 master 中挑选来的分支添加提交 commits：

```shell
git cherry-pick -x <commit_id>
```

还需要将步骤 2 和 3 中的版本号和 CHANGELOG 更新为单独的提交。

之后，标记它并通知用户新的候选版本：

```shell
export RELEASE_CANDIDATE_NAME="$RELEASE_NAME-rc.2"
git tag --sign --annotate "${RELEASE_CANDIDATE_NAME}" --message "Helm release ${RELEASE_CANDIDATE_NAME}"
git push upstream $RELEASE_CANDIDATE_NAME
```

从这里开始重复这个过程，不断测试，直到你对发布候选版本感到满意为止。

## 6. 完成版本

如果对候选版本的质量感到满意，那么可以继续前进并创建最终的东西。最后一次仔细检查以确保一切正常，然后最后推送版本标签。

```shell
git checkout $RELEASE_BRANCH_NAME
git tag --sign --annotate "${RELEASE_NAME}" --message "Helm release ${RELEASE_NAME}"
git push upstream $RELEASE_NAME
```

## 7. PGP 签署下载

虽然哈希提供了下载内容就是生成内容的签名，但签名包提供了包来自何处的可追溯性。

为此，请运行以下 make 命令：

```shell
make clean
make fetch-dist
make sign
```

这将为 CI 推送的每个文件生成 ascii 签名文件。

所有签名文件都需要上传到 GitHub 上的版本。


## 8. 编写版本说明


我们将根据版本周期中发生的提交自动生成更改日志，但如果版本说明是由 人 / 营销团队 / 狗 手写的，则通常对最终用户更有利。

如果要发布主要 / 次要版本，列出值得注意的面向用户的功能通常就足够了。对于修补程序版本，请执行相同操作，但请记下症状和受影响的人员。

次要版本的示例发行说明如下所示：

```markdown
## vX.Y.Z

Helm vX.Y.Z is a feature release. This release, we focused on <insert focal point>. Users are encouraged to upgrade for the best experience.

The community keeps growing, and we'd love to see you there!

- Join the discussion in [Kubernetes Slack](https://kubernetes.slack.com):
  - `#helm-users` for questions and just to hang out
  - `#helm-dev` for discussing PRs, code, and bugs
- Hang out at the Public Developer Call: Thursday, 9:30 Pacific via [Zoom](https://zoom.us/j/696660622)
- Test, debug, and contribute charts: [GitHub/helm/charts](https://github.com/helm/charts)

## Features and Changes

- Major
- features
- list
- here

## Installation and Upgrading

Download Helm X.Y. The common platform binaries are here:

- [MacOS amd64](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-darwin-amd64.tar.gz) ([checksum](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-darwin-amd64.tar.gz.sha256) / CHECKSUM_VAL)
- [Linux amd64](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-amd64.tar.gz) ([checksum](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-amd64.tar.gz.sha256) / CHECKSUM_VAL)
- [Linux arm](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-arm.tar.gz) ([checksum](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-arm.tar.gz.sha256) / CHECKSUM_VAL)
- [Linux arm64](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-arm64.tar.gz) ([checksum](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-arm64.tar.gz.sha256) / CHECKSUM_VAL)
- [Linux i386](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-386.tar.gz) ([checksum](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-386.tar.gz.sha256) / CHECKSUM_VAL)
- [Linux ppc64le](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-ppc64le.tar.gz) ([checksum](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-ppc64le.tar.gz.sha256) / CHECKSUM_VAL)
- [Linux s390x](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-s390x.tar.gz) ([checksum](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-linux-s390x.tar.gz.sha256) / CHECKSUM_VAL)
- [Windows amd64](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-windows-amd64.zip) ([checksum](https://storage.googleapis.com/kubernetes-helm/helm-vX.Y.Z-windows-amd64.zip.sha256) / CHECKSUM_VAL)

Once you have the client installed, upgrade Tiller with `helm init --upgrade`.

The [Quickstart Guide](https://docs.helm.sh/using_helm/#quickstart-guide) will get you going from there. For **upgrade instructions** or detailed installation notes, check the [install guide](https://docs.helm.sh/using_helm/#installing-helm). You can also use a [script to install](https://raw.githubusercontent.com/helm/helm/master/scripts/get) on any system with `bash`.

## What's Next

- vX.Y.Z+1 will contain only bug fixes.
- vX.Y+1.Z is the next feature release. This release will focus on ...

## Changelog

### Features
- ref(*): kubernetes v1.11 support efadbd88035654b2951f3958167afed014c46bc6 (Adam Reese)
- feat(helm): add $HELM_KEY_PASSPHRASE environment variable for signing helm charts (#4778) 1e26b5300b5166fabb90002535aacd2f9cc7d787

### Bug fixes
- fix circle not building tags f4f932fabd197f7e6d608c8672b33a483b4b76fa (Matthew Fisher)

### Code cleanup
- ref(kube): Gets rid of superfluous Sprintf call 3071a16f5eb3a2b646d9795617287cc26e53dba4  (Taylor Thomas)
- chore(*): bump version to v2.7.0 08c1144f5eb3e3b636d9775617287cc26e53dba4 (Adam Reese)

### Documentation Changes
- docs(release_checklist): fix changelog generation command (#4694) 8442851a5c566a01d9b4c69b368d64daa04f6a7f (Matthew Fisher)
```

可以使用以下命令生成发行说明底部的更改日志：

```shell
PREVIOUS_RELEASE=vX.Y.Z
git log --no-merges --pretty=format:'- %s %H (%aN)' $PREVIOUS_RELEASE..$RELEASE_NAME
```

生成更改日志后，需要对更改进行分类，如上例所示。

完成后，进入 GitHub 并使用此处写的注释编辑标记版本的发行说明。

请记住将上一步中生成的 ascii 签名附加到发行说明中。

## 9. 传福音

恭喜！你完成了。去抓一个 $DRINK_OF_CHOICE。这是你应得的。

在享受了一个不错的 $DRINK_OF_CHOICE 之后，请前往并宣布 Slack 和 Twitter 上新版本的喜讯。您还应该通知 helm 社区中的任何关键合作伙伴，例如 homebrew formula 维护者，孵化项目的所有者（例如 ChartMuseum）和任何其他感兴趣的团体。

（可选）写一篇关于新版本的博客文章，并在那里展示一些新功能！
