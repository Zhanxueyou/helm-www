---
title: "术语表" 
description: "用于描述Helm体系结构的组件。"
weight: 9
---

# 术语表

## Chart

Helm包涵盖了将Kubernetes资源安装到Kubernetes集群所需要的足够多的信息。

Charts包含`Chart.yaml`文件和模板，默认值(`values.yaml`)，以及相关依赖。

Charts开发设计了良好定义的目录结构，并且打包成了一种称为 _chart archive_ 文件格式。

## Chart包

Chart包(_chart archive_)是被tar和gzip压缩(并且可选签名)的chart.

## Chart依赖 (子chart)

Chart可以依赖于其他的chart。 依赖可能会以以下两种方式出现:

- 软依赖： A chart may simply not function without another chart being
  installed in a cluster. Helm does not provide tooling for this case. In this
  case, dependencies may be managed separately.
- 硬依赖： A chart may contain (inside of its `charts/` directory)
  another chart upon which it depends. In this case, installing the chart will
  install all of its dependencies. In this case, a chart and its dependencies
  are managed as a collection.

When a chart is packaged (via `helm package`) all of its hard dependencies are
bundled with it.

## Chart版本

Charts are versioned according to the [SemVer 2 spec](https://semver.org). A
version number is required on every chart.

## Chart.yaml

Information about a chart is stored in a special file called `Chart.yaml`. Every
chart must have this file.

## Helm (以及helm)

Helm is the package manager for Kubernetes. As an operating system package
manager makes it easy to install tools on an OS, Helm makes it easy to install
applications and resources into Kubernetes clusters.

While _Helm_ is the name of the project, the command line client is also named
`helm`. By convention, when speaking of the project, _Helm_ is capitalized. When
speaking of the client, _helm_ is in lowercase.

## Helm配置文件 (XDG)

Helm stores its configuration files in XDG directories. These directories are
created the first time `helm` is run.

## Kube 配置 (KUBECONFIG)

The Helm client learns about Kubernetes clusters by using files in the _Kube
config_ file format. By default, Helm attempts to find this file in the place
where `kubectl` creates it (`$HOME/.kube/config`).

## 代码规范 (进行中)

规范(_lint_) 一个chart是去验证其遵照Helm chart的标准规范和要求。Helm提供了工具来处理，尤其是`helm lint`命令。

## 来源 (来源文件)

Helm chart可以由来源文件(_provenance file_)提供chart的出处及它所包含的内容。

Provenance files are one part of the Helm security story. A provenance contains
a cryptographic hash of the chart archive file, the Chart.yaml data, and a
signature block (an OpenPGP "clearsign" block). When coupled with a keychain,
this provides chart users with the ability to:

- Validate that a chart was signed by a trusted party
- Validate that the chart file has not been tampered with
- Validate the contents of a chart metadata (`Chart.yaml`)
- Quickly match a chart to its provenance data

Provenance files have the `.prov` extension, and can be served from a chart
repository server or any other HTTP server.

## 发布版本

When a chart is installed, the Helm library creates a _release_ to track that
installation.

A single chart may be installed many times into the same cluster, and create
many different releases. For example, one can install three PostgreSQL databases
by running `helm install` three times with a different release name.

## 版本号

A single release can be updated multiple times. A sequential counter is used to
track releases as they change. After a first `helm install`, a release will have
_release number_ 1. Each time a release is upgraded or rolled back, the release
number will be incremented.

## 回滚

A release can be upgraded to a newer chart or configuration. But since release
history is stored, a release can also be _rolled back_ to a previous release
number. This is done with the `helm rollback` command.

Importantly, a rolled back release will receive a new release number.

| 操作       | 版本号                                       |
|------------|------------------------------------------------------|
| install    | release 1                                            |
| upgrade    | release 2                                            |
| upgrade    | release 3                                            |
| rollback 1 | release 4 (但使用release 1的配置) |

The above table illustrates how release numbers increment across install,
upgrade, and rollback.

## Helm库 (或SDK)

The Helm Library (or SDK) refers to the Go code that interacts directly with the
Kubernetes API server to install, upgrade, query, and remove Kubernetes
resources. It can be imported into a project to use Helm as a client library
instead of a CLI.

## 仓库 (Repo, Chart Repository)

Helm charts may be stored on dedicated HTTP servers called _chart repositories_
(_repositories_, or just _repos_).

A chart repository server is a simple HTTP server that can serve an `index.yaml`
file that describes a batch of charts, and provides information on where each
chart can be downloaded from. (Many chart repositories serve the charts as well
as the `index.yaml` file.)

A Helm client can point to zero or more chart repositories. By default, Helm
clients are not configured with any chart repositories. Chart repositories can
be added at any time using the `helm repo add` command.

## Values (Values文件, values.yaml)

Values provide a way to override template defaults with your own information.

Helm Charts are "parameterized", which means the chart developer may expose
configuration that can be overridden at installation time. For example, a chart
may expose a `username` field that allows setting a user name for a service.

These exposed variables are called _values_ in Helm parlance.

Values can be set during `helm install` and `helm upgrade` operations, either by
passing them in directly, or by using a `values.yaml` file.
