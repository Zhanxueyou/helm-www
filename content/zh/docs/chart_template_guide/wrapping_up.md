---
title: "下一步"
description: "收尾 - 一些能帮到你的指向其他的文档的链接。"
weight: 14
---

本指南旨在为chart开发者提供对如何使用Helm模板语言的强大理解能力。该模板聚焦于模板开发的技术层面。

但涉及到chart的实际日常开发时，很多内容本指南并没有覆盖到。这里有一些有用的文档链接帮助你创建新的chart:

- [Helm的Chart项目](https://github.com/helm/charts) 是不可或缺的chart资源。这个项目也定义了
chart开发最佳实践的标准。
- The Kubernetes [Documentation](https://kubernetes.io/docs/home/) provides
  detailed examples of the various resource kinds that you can use, from
  ConfigMaps and Secrets to DaemonSets and Deployments.
- The Helm [Charts Guide](../../topics/charts/) explains the workflow of using
  charts.
- The Helm [Chart Hooks Guide](../../topics/charts_hooks/) explains how to
  create lifecycle hooks.
- The Helm [Charts Tips and Tricks](../../howto/charts_tips_and_tricks/) article
  provides some useful tips for writing charts.
- The [Sprig documentation](https://github.com/Masterminds/sprig) documents more
  than sixty of the template functions.
- The [Go template docs](https://godoc.org/text/template) explain the template
  syntax in detail.
- The [Schelm tool](https://github.com/databus23/schelm) is a nice helper
  utility for debugging charts.

Sometimes it's easier to ask a few questions and get answers from experienced
developers. The best place to do this is in the [Kubernetes
Slack](https://kubernetes.slack.com) Helm channels:

- [#helm-users](https://kubernetes.slack.com/messages/helm-users)
- [#helm-dev](https://kubernetes.slack.com/messages/helm-dev)
- [#charts](https://kubernetes.slack.com/messages/charts)

Finally, if you find errors or omissions in this document, want to suggest some
new content, or would like to contribute, visit [The Helm
Project](https://github.com/helm/helm).
