---
title: "开发者指南"
description: "开发Helm的环境设置说明。"
weight: 1
---

该指南说明如何为开发Helm配置环境。

## 先决条件

- Go最新版本
- Kubernetes集群及kubectl（可选）
- Git

## 构建 Helm

我们使用Make构建程序。最简单的开始方式是：

```console
$ make
```

注意： 如果不是从`$GOPATH/src/helm.sh/helm`路径执行就会失败。目录`helm.sh`不应该是软链接，否则
`build` 找不到相关的包。

有必要的话要先安装依赖，重新构建 `vendor/` 目录，并验证配置。然后会编译 `helm` 并将其放到`bin/helm`。

在本地执行Helm，可以执行 `bin/helm`。

- Helm运行在 macOS 和大多数发行版上，包括 Alpine。

## 执行测试

要运行所有测试（除了`vendor/`），执行 `make test`。作为先决条件，需要安装
[golangci-lint](https://golangci-lint.run)。

## 贡献指南

我们欢迎你的贡献。 该项目已经设置了一些指南，为了保证 (a) 代码的高质量，(b) 保持项目一致，
(c) 贡献遵循开源法律要求。我们的目的不是为贡献者增加负担，但是要构建优雅和高质量的开源代码，
这样我们的用户才能从中受益。

确保你已经阅读并理解主要贡献指南：

https://github.com/helm/helm/blob/master/CONTRIBUTING.md

### 代码结构

Helm项目的代码组织如下：

- 独立的程序位于 `cmd/`。`cmd/` 中的代码不是为库复用设计的。
- 共享的库放在 `pkg/`。
- `scripts/` 目录包含很多实用程序脚本。大多数用于CI/CD流水线。

Go依赖管理在不断变化，而且在Helm生命周期中很可能发生变化。我们建议开发者 _不要_ 手动管理依赖。
而是建议依靠项目的 `Makefile` 来处理。使用Helm 3时，建议使用Go 1.13及更新版本。

### 编写文档

从Helm 3开始，文档已经移动到了它自己的仓库中。当编制新特性时，请编写随附文档并提交到
[helm-www](https://github.com/helm/helm-www) 仓库。

### Git 约定

We use Git for our version control system. The `master` branch is the home of
the current development candidate. Releases are tagged. If you are working on
patches for Helm v2, the branch `dev-v2` is the base branch from which Helm 2
releases are cut.

We accept changes to the code via GitHub Pull Requests (PRs). One workflow for
doing this is as follows:

1. Go to your `$GOPATH/src` directory, then `mkdir helm.sh; cd helm.sh` and `git
   clone` the `github.com/helm/helm` repository.
2. Fork that repository into your GitHub account
3. Add your repository as a remote for `$GOPATH/src/helm.sh/helm`
4. Create a new working branch (`git checkout -b feat/my-feature`) and do your
   work on that branch.
5. When you are ready for us to review, push your branch to GitHub, and then
   open a new pull request with us.

For Git commit messages, we follow the [Semantic Commit
Messages](https://karma-runner.github.io/0.13/dev/git-commit-msg.html):

```
fix(helm): add --foo flag to 'helm install'

When 'helm install --foo bar' is run, this will print "foo" in the
output regardless of the outcome of the installation.

Closes #1234
```

Common commit types:

- fix: Fix a bug or error
- feat: Add a new feature
- docs: Change documentation
- test: Improve testing
- ref: refactor existing code

Common scopes:

- helm: The Helm CLI
- pkg/lint: The lint package. Follow a similar convention for any package
- `*`: two or more scopes

Read more:
- The [Deis
  Guidelines](https://github.com/deis/workflow/blob/master/src/contributing/submitting-a-pull-request.md)
  were the inspiration for this section.
- Karma Runner
  [defines](https://karma-runner.github.io/0.13/dev/git-commit-msg.html) the
  semantic commit message idea.

### Go 约定

We follow the Go coding style standards very closely. Typically, running `go
fmt` will make your code beautiful for you.

We also typically follow the conventions recommended by `go lint` and
`gometalinter`. Run `make test-style` to test the style conformance.

Read more:

- Effective Go [introduces
  formatting](https://golang.org/doc/effective_go.html#formatting).
- The Go Wiki has a great article on
  [formatting](https://github.com/golang/go/wiki/CodeReviewComments).

If you run the `make test` target, not only will unit tests be run, but so will
style tests. If the `make test` target fails, even for stylistic reasons, your
PR will not be considered ready for merging.
