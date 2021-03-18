---
title: "版本 Checklist"
description: "维护人员在发布下一个Helm版本时的checklist。"
weight: 2
---

# 维护人员发布Helm指南

是时候发布新的Helm了！作为Helm维护者发布版本，如果你的经验与这里的文档不同，那你就是
[更新版本checklist](https://github.com/helm/helm-www/blob/master/content/en/docs/community/release_checklist.md)
的最佳人选。

所有版本都将采用vX.Y.Z的形式，X是主版本号，Y是次版本号，Z是补丁发布号。该项目严格遵守 [语义化版本](https://semver.org/)，
因此遵循这一点非常重要。

Helm会提前宣布下个次版本发布的日期。应尽一切努力遵守宣布的日期。此外，在开始发布过程时，应该选择下一个发布的日期在发布过程中使用。

这些说明将涵盖三种不同版本的遵守发布过程的初始配置：

- 主版本 - 发布频率较低 - 有重大更新时
- 次版本 - 每3到4个月发布 - 无重大更新
- 补丁版本 - 每月发布 - 不需要指南中的所有步骤

[初始化配置](#initial-configuration)

1. [创建发布分支](#1-create-the-release-branch)
2. [主/次版本：在Git中更改版本号](#2-majorminor-releases-change-the-version-number-in-git)
3. [主/次版本：提交并推送发布分支](#3-majorminor-releases-commit-and-push-the-release-branch)
4. [主/次版本：创建一个发布候选](#4-majorminor-releases-create-a-release-candidate)
5. [主/次版本：迭代连续的候选版本](#5-majorminor-releases-iterate-on-successive-release-candidates)
6. [完成发布](#6-finalize-the-release)
7. [编写发布日志](#7-write-the-release-notes)
8. [PGP签名下载](#8-pgp-sign-the-downloads)
9. [发布版本](#9-publish-release)
10. [更新文档](#10-update-docs)
11. [告知社区](#11-tell-the-community)

## Initial Configuration

### 设置远程Git

需要注意的是该文档假设你的远程upstream仓库关联到了<https://github.com/helm/helm>。
如果不是（比如，如果你选择了“origin”或其他类似的替代），请确保根据本地环境调整列出的代码段。
如果你不确定使用了什么远程的upstream，使用`git remote -v`命令查看。

如果你没有[上游远程](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/configuring-a-remote-for-a-fork)，
可以类似这样添加：

```shell
git remote add upstream git@github.com:helm/helm.git
```

### 设置环境变量

在该文档中，我们还会引用一些环境变量，更便于设置。针对主、次版本，使用以下选项：

```shell
export RELEASE_NAME=vX.Y.0
export RELEASE_BRANCH_NAME="release-X.Y"
export RELEASE_CANDIDATE_NAME="$RELEASE_NAME-rc.1"
```

如果你在创建一个补丁版本，改用以下命令：

```shell
export PREVIOUS_PATCH_RELEASE=vX.Y.Z
export RELEASE_NAME=vX.Y.Z+1
export RELEASE_BRANCH_NAME="release-X.Y"
```

### 设置签名Key

我们还会通过对二进制文件和提供的签名文件进行哈希计算增加发布过程的安全性和认证。
使用[GitHub 和 GPG](https://help.github.com/en/articles/about-commit-signature-verification)来执行。
如果还没设置GPG可以按照以下步骤操作：

1. [安装 GPG](https://gnupg.org/index.html)
2. [生成 GPG key](https://help.github.com/en/articles/generating-a-new-gpg-key)
3. [将key添加到GitHub账户中](https://help.github.com/en/articles/adding-a-new-gpg-key-to-your-github-account)
4. [在Git中设置签名密钥](https://help.github.com/en/articles/telling-git-about-your-signing-key)

一旦你有了签名密钥，需要将其添加到仓库根目录中的KEYS文件中。文件中有添加密钥到KEY文件的说明。如果还没有，需要将公钥添加到keyserver。
如果使用了GnuPG，可以参照[Debian提供的说明](https://debian-administration.org/article/451/Submitting_your_GPG_key_to_a_keyserver)。

## 1. Create the Release Branch

### 主、次版本

主版本是为新特性及操作且*不具有向后兼容性*。次版本是为了不破坏向后兼容性的新特性。创建一个主版本或次版本，从主干分支创建`release-vX.Y.0`分支。

```shell
git fetch upstream
git checkout upstream/master
git checkout -b $RELEASE_BRANCH_NAME
```

这个新分支是发布版本的基础分支，会在后面不断迭代。

为GitHub上已存在的版本验证[helm/helm里程碑](https://github.com/helm/helm/milestones)。
确保针对这个版本的PR和issue都在这个里程碑中。

针对主版本和次版本，跳转到 2: [主/次版本：在Git中更改版本号](#2-majorminor-releases-change-the-version-number-in-git)。

### 补丁版本

补丁版本是一些已有版本中严格的cherry-picked修复。以创建`release-vX.Y.Z`分支开始：

```shell
git fetch upstream
git checkout -b $RELEASE_BRANCH_NAME upstream/$RELEASE_BRANCH_NAME
```

在这里可以cherry-pick出需要带到补丁版本中的提交：

```shell
# get the commits ids we want to cherry-pick
git log --oneline
# cherry-pick the commits starting from the oldest one, without including merge commits
git cherry-pick -x <commit-id>
```

挑出提交之后这个版本分支需要被推送。

```shell
git push upstream $RELEASE_BRANCH_NAME
```

推送分支会触发测试。创建tag之前确保测试是通过的。
这个新tag将成为补丁版本的基础。

针对补丁版本，创建[helm/helm里程碑](https://github.com/helm/helm/milestones)是可选的。

继续之前确保[helm 在 CircleCI](https://circleci.com/gh/helm/helm)通过CI。补丁版本可以跳过2-5步，
直接执行6 [完成发布](#6-finalize-the-release)。

## 2. Major/Minor releases: Change the Version Number in Git

当有主版本或次版本发布时，确保用新版本更新`internal/version/version.go`。

```shell
$ git diff internal/version/version.go
diff --git a/internal/version/version.go b/internal/version/version.go
index 712aae64..c1ed191e 100644
--- a/internal/version/version.go
+++ b/internal/version/version.go
@@ -30,7 +30,7 @@ var (
        // Increment major number for new feature additions and behavioral changes.
        // Increment minor number for bug fixes and performance enhancements.
        // Increment patch number for critical fixes to existing releases.
-       version = "v3.3"
+       version = "v3.4"

        // metadata is extra build time data
        metadata = ""
```

除了在`version.go`文件中更行版本，还需要更新使用了新版本的相关测试。

- `cmd/helm/testdata/output/version.txt`
- `cmd/helm/testdata/output/version-client.txt`
- `cmd/helm/testdata/output/version-client-shorthand.txt`
- `cmd/helm/testdata/output/version-short.txt`
- `cmd/helm/testdata/output/version-template.txt`
- `pkg/chartutil/capabilities_test.go`

```shell
git add .
git commit -m "bump version to $RELEASE_NAME"
```

这只会对$RELEASE_BRANCH_NAME更新。也许要在下个版本更新时推送到主干分支，就像 [3.2 更新到
3.3](https://github.com/helm/helm/pull/8411/files)，并将其添加到下一个版本的里程碑中。

```shell
# get the last commit id i.e. commit to bump the version
git log --format="%H" -n 1

# create new branch off master
git checkout master
git checkout -b bump-version-<release_version>

# cherry pick the commit using id from first command
git cherry-pick -x <commit-id>

# commit the change
git push origin bump-version-<release-version>
```

## 3. Major/Minor releases: Commit and Push the Release Branch

为了让他人开始测试，我们可以推送发布分支到upstream并开始测试过程。

```shell
git push upstream $RELEASE_BRANCH_NAME
```

继续之前确保 [helm 在CircleCI](https://circleci.com/gh/helm/helm)版本通过CI。

如果有人可用，让其他人在确保所有更改都已正确处理且所有该版本的提交都已存在，并提前对分支进行同行评审。

## 4. Major/Minor releases: Create a Release Candidate

Now that the release branch is out and ready, it is time to start creating and
iterating on release candidates.

```shell
git tag --sign --annotate "${RELEASE_CANDIDATE_NAME}" --message "Helm release ${RELEASE_CANDIDATE_NAME}"
git push upstream $RELEASE_CANDIDATE_NAME
```

CircleCI will automatically create a tagged release image and client binary to
test with.

For testers, the process to start testing after CircleCI finishes building the
artifacts involves the following steps to grab the client:

linux/amd64, using /bin/bash:

```shell
wget https://get.helm.sh/helm-$RELEASE_CANDIDATE_NAME-linux-amd64.tar.gz
```

darwin/amd64, using Terminal.app:

```shell
wget https://get.helm.sh/helm-$RELEASE_CANDIDATE_NAME-darwin-amd64.tar.gz
```

windows/amd64, using PowerShell:

```shell
PS C:\> Invoke-WebRequest -Uri "https://get.helm.sh/helm-$RELEASE_CANDIDATE_NAME-windows-amd64.tar.gz" -OutFile "helm-$ReleaseCandidateName-windows-amd64.tar.gz"
```

Then, unpack and move the binary to somewhere on your $PATH, or move it
somewhere and add it to your $PATH (e.g. /usr/local/bin/helm for linux/macOS,
C:\Program Files\helm\helm.exe for Windows).

## 5. Major/Minor releases: Iterate on Successive Release Candidates

Spend several days explicitly investing time and resources to try and break helm
in every possible way, documenting any findings pertinent to the release. This
time should be spent testing and finding ways in which the release might have
caused various features or upgrade environments to have issues, not coding.
During this time, the release is in code freeze, and any additional code changes
will be pushed out to the next release.

During this phase, the $RELEASE_BRANCH_NAME branch will keep evolving as you
will produce new release candidates. The frequency of new candidates is up to
the release manager: use your best judgement taking into account the severity of
reported issues, testers' availability, and the release deadline date. Generally
speaking, it is better to let a release roll over the deadline than to ship a
broken release.

Each time you'll want to produce a new release candidate, you will start by
adding commits to the branch by cherry-picking from master:

```shell
git cherry-pick -x <commit_id>
```

You will also want to push the branch to GitHub and ensure it passes CI.

After that, tag it and notify users of the new release candidate:

```shell
export RELEASE_CANDIDATE_NAME="$RELEASE_NAME-rc.2"
git tag --sign --annotate "${RELEASE_CANDIDATE_NAME}" --message "Helm release ${RELEASE_CANDIDATE_NAME}"
git push upstream $RELEASE_CANDIDATE_NAME
```

Once pushed to GitHub, check to ensure the branch with this tag builds in CI.

From here on just repeat this process, continuously testing until you're happy
with the release candidate. For a release candidate, we don't write the full notes,
but you can scaffold out some [release notes](#7-write-the-release-notes).

## 6. Finalize the Release

When you're finally happy with the quality of a release candidate, you can move
on and create the real thing. Double-check one last time to make sure everything
is in order, then finally push the release tag.

```shell
git checkout $RELEASE_BRANCH_NAME
git tag --sign --annotate "${RELEASE_NAME}" --message "Helm release ${RELEASE_NAME}"
git push upstream $RELEASE_NAME
```

Verify that the release succeeded in
[CircleCI](https://circleci.com/gh/helm/helm). If not, you will need to fix the
release and push the release again.

As the CI job will take some time to run, you can move on to writing release
notes while you wait for it to complete.

## 7. Write the Release Notes

We will auto-generate a changelog based on the commits that occurred during a
release cycle, but it is usually more beneficial to the end-user if the release
notes are hand-written by a human being/marketing team/dog.

If you're releasing a major/minor release, listing notable user-facing features
is usually sufficient. For patch releases, do the same, but make note of the
symptoms and who is affected.

The release notes should include the version and planned date of the next release.

An example release note for a minor release would look like this:

```markdown
## vX.Y.Z

Helm vX.Y.Z is a feature release. This release, we focused on <insert focal point>. Users are encouraged to upgrade for the best experience.

The community keeps growing, and we'd love to see you there!

- Join the discussion in [Kubernetes Slack](https://kubernetes.slack.com):
  - `#helm-users` for questions and just to hang out
  - `#helm-dev` for discussing PRs, code, and bugs
- Hang out at the Public Developer Call: Thursday, 9:30 Pacific via [Zoom](https://zoom.us/j/696660622)
- Test, debug, and contribute charts: [Artifact Hub helm charts](https://artifacthub.io/packages/search?kind=0)

## Notable Changes

- Kubernetes 1.16 is now supported including new manifest apiVersions
- Sprig was upgraded to 2.22

## Installation and Upgrading

Download Helm X.Y. The common platform binaries are here:

- [MacOS amd64](https://get.helm.sh/helm-vX.Y.Z-darwin-amd64.tar.gz) ([checksum](https://get.helm.sh/helm-vX.Y.Z-darwin-amd64.tar.gz.sha256sum) / CHECKSUM_VAL)
- [Linux amd64](https://get.helm.sh/helm-vX.Y.Z-linux-amd64.tar.gz) ([checksum](https://get.helm.sh/helm-vX.Y.Z-linux-amd64.tar.gz.sha256sum) / CHECKSUM_VAL)
- [Linux arm](https://get.helm.sh/helm-vX.Y.Z-linux-arm.tar.gz) ([checksum](https://get.helm.sh/helm-vX.Y.Z-linux-arm.tar.gz.sha256) / CHECKSUM_VAL)
- [Linux arm64](https://get.helm.sh/helm-vX.Y.Z-linux-arm64.tar.gz) ([checksum](https://get.helm.sh/helm-vX.Y.Z-linux-arm64.tar.gz.sha256sum) / CHECKSUM_VAL)
- [Linux i386](https://get.helm.sh/helm-vX.Y.Z-linux-386.tar.gz) ([checksum](https://get.helm.sh/helm-vX.Y.Z-linux-386.tar.gz.sha256) / CHECKSUM_VAL)
- [Linux ppc64le](https://get.helm.sh/helm-vX.Y.Z-linux-ppc64le.tar.gz) ([checksum](https://get.helm.sh/helm-vX.Y.Z-linux-ppc64le.tar.gz.sha256sum) / CHECKSUM_VAL)
- [Linux s390x](https://get.helm.sh/helm-vX.Y.Z-linux-s390x.tar.gz) ([checksum](https://get.helm.sh/helm-vX.Y.Z-linux-s390x.tar.gz.sha256sum) / CHECKSUM_VAL)
- [Windows amd64](https://get.helm.sh/helm-vX.Y.Z-windows-amd64.zip) ([checksum](https://get.helm.sh/helm-vX.Y.Z-windows-amd64.zip.sha256sum) / CHECKSUM_VAL)

The [Quickstart Guide](https://docs.helm.sh/using_helm/#quickstart-guide) will get you going from there. For **upgrade instructions** or detailed installation notes, check the [install guide](https://docs.helm.sh/using_helm/#installing-helm). You can also use a [script to install](https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3) on any system with `bash`.

## What's Next

- vX.Y.Z+1 will contain only bug fixes and is planned for <insert DATE>.
- vX.Y+1.0 is the next feature release and is planned for <insert DATE>. This release will focus on ...

## Changelog

- chore(*): bump version to v2.7.0 08c1144f5eb3e3b636d9775617287cc26e53dba4 (Adam Reese)
- fix circle not building tags f4f932fabd197f7e6d608c8672b33a483b4b76fa (Matthew Fisher)
```

A partially completed set of release notes including the changelog can be
created by running the following command:

```shell
export VERSION="$RELEASE_NAME"
export PREVIOUS_RELEASE=vX.Y.Z
make clean
make fetch-dist
make release-notes
```

This will create a good baseline set of release notes to which you should just
need to fill out the **Notable Changes** and **What's next** sections.

Feel free to add your voice to the release notes; it's nice for people to think
we're not all robots.

You should also double check the URLs and checksums are correct in the
auto-generated release notes.

Once finished, go into GitHub to [helm/helm
releases](https://github.com/helm/helm/releases) and edit the release notes for
the tagged release with the notes written here.
For target branch, set to $RELEASE_BRANCH_NAME.

It is now worth getting other people to take a look at the release notes before
the release is published. Send a request out to
[#helm-dev](https://kubernetes.slack.com/messages/C51E88VDG) for review. It is
always beneficial as it can be easy to miss something.

## 8. PGP Sign the downloads

While hashes provide a signature that the content of the downloads is what it
was generated, signed packages provide traceability of where the package came
from.

To do this, run the following `make` commands:

```shell
export VERSION="$RELEASE_NAME"
make clean		# if not already run
make fetch-dist	# if not already run
make sign
```

This will generate ascii armored signature files for each of the files pushed by
CI.

All of the signature files (`*.asc`) need to be uploaded to the release on
GitHub (attach binaries).

## 9. Publish Release

Time to make the release official!

After the release notes are saved on GitHub, the CI build is completed, and
you've added the signature files to the release, you can hit "Publish" on
the release. This publishes the release, listing it as "latest", and shows this
release on the front page of the [helm/helm](https://github.com/helm/helm) repo.

## 10. Update Docs

The [Helm website docs section](https://helm.sh/docs) lists the Helm versions
for the docs. Major, minor, and patch versions need to be updated on the site.
The date for the next minor release is also published on the site and must be
updated.
To do that create a pull request against the [helm-www
repository](https://github.com/helm/helm-www). In the `config.toml` file find
the proper `params.versions` section and update the Helm version, like in this
example of [updating the current
version](https://github.com/helm/helm-www/pull/676/files).  In the same
`config.toml` file, update the `params.nextversion` section.

Close the [helm/helm milestone](https://github.com/helm/helm/milestones) for
the release, if applicable.

Update the [version
skew](https://github.com/helm/helm-www/blob/master/content/en/docs/topics/version_skew.md)
for major and minor releases.

Update the release calendar [here](https://helm.sh/calendar/release):
* create an entry for the next minor release with a reminder for that day at 5pm GMT
* create an entry for the RC1 of the next minor release on the Monday of the week before the planned release, with a reminder for that day at 5pm GMT

## 11. Tell the Community

Congratulations! You're done. Go grab yourself a $DRINK_OF_CHOICE. You've earned
it.

After enjoying a nice $DRINK_OF_CHOICE, go forth and announce the new release
in Slack and on Twitter with a link to the [release on
GitHub](https://github.com/helm/helm/releases).

Optionally, write a blog post about the new release and showcase some of the new
features on there!
