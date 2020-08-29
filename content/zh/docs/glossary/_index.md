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

- 软依赖： 如果另一个chart没有在集群中安装，chart可能会无法使用。Helm未对这个案例提供工具。
这个案例中，依赖会被分别管理。
- 硬依赖： 一个chart可以包含 (在它的`charts/`目录中) 另一个它所依赖的chart。这个案例中，
安装chart的同时会安装所有依赖。chart和它的依赖会作为一个集合进行管理。

当一个chart(通过`helm package`)打包时所有的依赖都会和它绑定。

## Chart版本

Chart版本根据 [语义化版本2.0 细则](https://semver.org) 发布。每个chart都需要版本号。

## Chart.yaml

chart的信息说明被存储在一个特定文件`Chart.yaml`。每个chart都必须有这个文件。

## Helm (以及helm)

Helm 是Kubernetes的包管理器。作为一个操作系统包管理器，使其很容易在操作系统中安装工具。
Helm使得Kubernetes集群中安装应用和资源变得异常简单。

当 _Helm_ 是项目名称时, 命令行客户端也可以使用 `helm`。按照惯例，当指项目时，_Helm_ 
使用大写。当指命令行时, _helm_ 使用小写。

## Helm配置文件 (XDG)

Helm将配置文件存储在XDG目录中。`helm`第一次运行时这些目录会自动生成。

## Kube 配置 (KUBECONFIG)

Helm客户端会通过使用 _Kube config_ 文件格式来理解Kubernetes集群。
默认情况下，Helm会尝试在 `kubectl` 创建的 (`$HOME/.kube/config`) 目录中查找这些文件。

## 代码规范 (进行中)

规范(_lint_) 一个chart是去验证其遵照Helm chart的标准规范和要求。
Helm提供了工具来处理，尤其是`helm lint`命令。

## 来源 (来源文件)

Helm chart可以由来源文件(_provenance file_)提供chart的出处及它所包含的内容。

来源文件是Helm安全故事的一部分。一个来源包含chart包文件的加密哈希值，Chart.yaml数据，
和一个签名块（一个OpenPGP "clearsign" 块）。当再加上一个钥匙链时，可以为chart用户提供
以下能力：

- 验证chart被可信第三方签名
- 验证chart文件没有被篡改
- 验证chart的元数据内容(`Chart.yaml`)
- 快速匹配chart的数据来源

来源文件有`.prov`扩展名，可以由chart仓库服务器或其他HTTP服务器提供。

## 发布版本

chart安装之后，Helm库会创建一个 _release_ 来跟踪这个安装。

单个chart可以在同一个集群中安装多次，并能创建多个不同的版本。例如，可以使用 `helm install` 
命令以不同的版本安装三个PostgreSQL 数据库三次。

## 版本号

单个版本号可以被升级多次。通过连续技术来跟踪升级发布版本。在第一次`helm install`之后，一个版本
会有 _release number_ 1，每一次版本升级或回滚，版本号都会升级。

## 回滚

每一次发布会更新chart或者配置。当生成发布历史后，一次发布也可以被 _rolled back_ 之前的发布版本号。
回滚使用 `helm rollback` 命令实现。

重要的是, 每一次回滚版本会生成一个新的发布版本号。

| 操作       | 版本号                                       |
|------------|------------------------------------------------------|
| install    | release 1                                            |
| upgrade    | release 2                                            |
| upgrade    | release 3                                            |
| rollback 1 | release 4 (但使用release 1的配置) |

上面的列表阐明了发布版本号是怎样通过安装，升级和回滚递增的。

## Helm库 (或SDK)

Helm库（或SDK）涉及到go代码，可以直接与Kubernetes API服务交互进行安装、升级、查询
以及移除Kubernetes资源。可以被导入到项目中来替换CLI使用的Helm库。

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
