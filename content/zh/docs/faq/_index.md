---
title: "常见问答(FAQ)"
weight: 8
---

# 常见问答

> Helm 2和Helm 3的主要区别是什么?  
> 该页面为最常见的问题提供帮助。

**我们喜欢您的帮助** 使得该文档变得更好。添加、修改或移除信息,
 [提问题](https://github.com/helm/helm-www/issues)或者向我们提交pull request。

## 从Helm 2发生的变化

这里会展示一个对Helm3主要引入的变化的详尽列表。

### 移除了Tiller

&emsp;&emsp;在Helm 2的开发周期中，我们引入了Tiller。Tiller在团队协作中共享集群时扮演了重要角色。
它使得不同的操作员与相同的版本进行交互称为了可能。

&emsp;&emsp;Kubernetes 1.6默认使用了基于角色的访问控制（RBAC），在生产环境对Tiller的锁定使用变得难于管理。
由于大量可能的安全策略，我们的立场是提供一个自由的默认配置。这样可以允许新手用户可以乐于尝试Helm
和Kubernetes而不需要深挖安全控制。 不幸的是这种自由的配置会授予用户他们不该有的权限。DevOps和SRE
在安装多用户集群时不得不去学习额外的操作步骤。

&emsp;&emsp;在听取了社区成员在特定场景使用Helm之后，我们发现Tiller的版本管理系统不需要依赖于集群内部用户去维护
状态或者作为一个Helm版本信息的中心hub。取而代之的是，我们可以简单地从Kubernetes API server获取信息，
在Chart客户端处理并在Kubernetes中存储安装记录。

Tiller的首要目标可以在没有Tiller的情况下实现，因此针对于 Helm 3 我们做的首要决定之一就是完全移除Tiller。

&emsp;&emsp;随着Tiller的消失，Helm的安全模块从根本上被简化。Helm 3 现在支持所有Kubernetes流行的安全、
身份和授权特性。Helm的权限通过你的
[kubeconfig文件](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)进行评估。
集群管理员可以限制用户权限，只要他们觉得合适，
无论什么粒度都可以做到。版本发布记录和Helm的剩余保留功能仍然会被记录在集群中。

### 改进升级策略: 三路策略合并补丁

Helm 2 使用了一种双路策略合并补丁。在升级过程中，会对比最近一次的chart manifest和提出的
chart manifest(通过`helm upgrade`提供)。升级会对比两个chart的不同来决定哪些更改会应用到Kubernetes资源中。
如果更改是集群外带的（比如通过`kubectl edit`），则不会被考虑。结果就是资源不会回滚到之前的状态：
因为Helm只考虑最后一次应用的chart manifest作为它的当前状态，如果chart状态没有更改，则资源的活动状态不会更改。

现在Helm 3中，我们使用一种三路策略来合并补丁。Helm在生成一个补丁时会考虑之前老的manifest的活动状态。

#### 示例

让我们通过一些常见的例子来看看变化带来的影响。

##### 回滚已经改变的活动状态

你的团队正好在Kubernetes上使用Helm部署了生产环境应用。chart包含了一个部署对象使用了三套副本：

```console
$ helm install myapp ./myapp
```

&emsp;&emsp;一个开发新人加入了团队。当他们第一点观察生产环境集群时，发生了一个像是咖啡洒在了键盘上一样的严重事故，
他们使用 `kubectl scale` 对生产环境部署进行缩容，将副本数从3降到了0 。

```console
$ kubectl scale --replicas=0 deployment/myapp
```

&emsp;&emsp;团队里面的另一个人看到线上环境已经挂了就决定回滚这个版本到之前的状态：

```console
$ helm rollback myapp
```

发生了什么？

在Helm 2中，会生成一个补丁并对比老的manifest和新的manifest。因为这是一个回滚，manifest是一样的。
Helm会认为新老manifest没有区别，因此没有需要更改的内容。副本统计数量继续保持为0。恐慌就接踵而至。

在Helm 3中，是用老的manifest生成新的补丁，活动状态和新的manifest。
Helm 意识到老的状态是3，而现有活动状态是0，并且新的manifest希望改回3，因此会生成一个补丁将状态改回3。

##### 活动状态已更改的情况下升级

很多服务网格和其他基于controller的应用向Kubernetes对象中注入数据。比如sidecar、label和其他信息。
之前如果你从Chart渲染给定的manifest如下:

```yaml
containers:
- name: server
  image: nginx:2.0.0
```

并且另一个应用修改活动状态如下：

```yaml
containers:
- name: server
  image: nginx:2.0.0
- name: my-injected-sidecar
  image: my-cool-mesh:1.0.0
```

现在你想升级`nginx`镜像到`2.1.0`。因此用指定的manifest升级chart：

```yaml
containers:
- name: server
  image: nginx:2.1.0
```

发生了什么？

在Helm 2中，Helm 在新老manifest之间生成了一个`containers`对象的补丁。
生成补丁的过程中不考虑集群的活动状态。

集群的活动状态被修改成了这样:

```yaml
containers:
- name: server
  image: nginx:2.1.0
```

sidecar pod从活动状态中移除了。更多的恐慌袭来。

在Helm 3中，Helm 在新的manifest、活动状态和老manifest之间生成了一个`containers`对象的补丁。
会注意到新的manifest将镜像tag更新为`2.1.0`, 但是活动状态中包含了一个sidecar容器。

集群的活动状态被修改成了下面这样：

```yaml
containers:
- name: server
  image: nginx:2.1.0
- name: my-injected-sidecar
  image: my-cool-mesh:1.0.0
```

### 发布名称现在限制在namespace范围内

&emsp;&emsp;随着Tiller的移除， 每个版本的信息需要保存在某个地方。
在Helm 2中，是存储在Tiller相同的命名空间中。 
实际上这意味着一个发布版本使用一个名称，其他发布不能使用相同的名称，
即使在不同的命名空间中也不行。

&emsp;&emsp;在Helm 3中，特定的版本信息作为发布本身存储在相同的命名空间中。
意味着用户现在可以在两个分开的命名空间中使用`helm install wordpress stable/wordpress`，
并且每个都能使用 `helm list` 改变当前命名空间。 (例如 `helm list --namespace foo`)。

&emsp;&emsp;与本地集群命名空间更好的一致性，使得 `helm list` 命令不再需要默认列出所有发布版本的列表。
取而代之的是，仅仅会在命名空间中列出当前kubernetes上下文的版本。
(也就是说运行`kubectl config view --minify`时会显示命名空间). 也就意味着您在执行`helm list`时必须提供
 `--all-namespaces` 标识才能获得和Helm 2同样的结果。

### 作为默认存储器的密钥

In Helm 3, 密钥被作为[默认存储驱动](/docs/topics/advanced/#storage-backends)使用。
Helm 2默认使用ConfigMaps记录版本信息。在Helm 2.7.0中，新的存储后台使用密钥来存储版本信息，
现在是Helm 3的默认设置。

Helm 3默认允许更改密钥作为额外的安全措施在Kubernetes中和密钥加密一起保护chart。

[静态加密密钥](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
在Kubernetes 1.7中作为alpha特性可以使用了，在Kubernetes 1.13中变成了稳定特性。
这允许用户静态加密Helm的发布元数据，同时也是一个类似Vault的以后可扩展的良好起点。

### 更改了Go的path导入

在Helm 3中，Helm将Go的import路径从`k8s.io/helm`切换到了`helm.sh/helm/v3`。如果你打算
升级到Helm 3的Go客户端库，确保你已经更改了import路径。

### Capabilities

`.Capabilities`内置对象会在已经简化的渲染阶段生效。

[内置对象](/docs/chart_template_guide/builtin_objects/)

### 使用Json格式验证Chart Values

chart values现在可以使用JSON结构了。这保证用户提供value可以按照chart维护人员设置的结构排列，
并且当用户提供了错误的chart value时会有更好错误提示。

当调用以下命令时会进行JSON格式验证：

* `helm install`
* `helm upgrade`
* `helm template`
* `helm lint`

查看 [格式文档](/docs/topics/charts#schema-files) 了解更多信息。

### 将 `requirements.yaml` 合并到了 `Chart.yaml`

Chart依赖体系从 requirements.yaml 和 requirements.lock 移动到 Chart.yaml
和 Chart.lock。我们推荐在Helm 3的新chart中使用新格式。不过Helm 3 依然可以识别
Chart API 版本1 (`v1`) 并且会加载已有的 `requirements.yaml` 文件。

Helm 2中，`requirements.yaml` 看起来是这样的:

```yaml
dependencies:
- name: mariadb
  version: 5.x.x
  repository: https://kubernetes-charts.storage.googleapis.com/
  condition: mariadb.enabled
  tags:
    - database
```

Helm 3中， 依赖使用了同样的表达方式，现在`Chart.yaml`是这样的：

```yaml
dependencies:
- name: mariadb
  version: 5.x.x
  repository: https://kubernetes-charts.storage.googleapis.com/
  condition: mariadb.enabled
  tags:
    - database
```

Chart会依然下载和放置在 `charts/` 目录， 因此 `charts/` 目录中的子chart不作修改即可继续工作。

### Name (或者 --generate-name) 安装时是必需的

Helm 2中，如果没有提供名称， 会自动生成一个名称。在生产环境，这被证明是一个麻烦事而不是一个有用的特性。
而在Helm 3中，如果 `helm install` 没有提供name，会抛异常。

如果仍然需要一个自动生成的名称，您可以使用 `--generate-name` 创建。

### 推送Chart到OCI注册中心

这是一个Helm 3 中的实验性特性。使用时需要设置环境变量 `HELM_EXPERIMENTAL_OCI=1`。

Chart仓库在较高层次上是一个存储和分发Chart的地址。Heml客户端打包并将Chart推送到Chart仓库中。
简单来说，Chart仓库就是一个基本的HTTP服务器用来存放index.yaml文件和打包的chart。

Chart 仓库API满足最基本的需求有一些好处，但是有些缺点开始显现出来：

- Chart 仓库很难在生产环境抽象出大部分的安全性实现。在生产环境有一个认证和授权的标准API就显得格外重要。
- Helm Chart的初始化工具用来签名和验证chart的完整性和来源，在chart的发布过程中是可选的。
- 在多客户场景中，同一个chart可以被其他客户上传，同样的内容会被存储两次。chart仓库可以更加智能地处理
这个问题，但并不是正式规范的一部分。
- 在安全的多客户实现中使用单一的索引文件进行搜索、元数据信息存放和获取chart会变得困难和笨拙。

Docker的分发项目（也称作Docker注册中心 v2）是Docker 注册项目的继承者。 很多主要的云供应商都提供项目
分发，很多供应商都提供相同的产品， 这个分发项目得益于多年的强化、安全性实践和对抗测试。

请查看 `helm help chart` 和 `helm help registry` 了解如何打包chart并推送到Docker注册中心的更多信息。

更多信息请查看 [注册中心](/docs/topics/registries/)页面。

### 移除了`helm serve`

`helm serve` 命令可以在你本地机器运行一个Chart仓库用于开发目的。
然而作为一个开发工具并没有受到太多利用，并且设计上有很多问题。最终我们决定移除它，
拆分成了一个插件。

对于 `helm serve` 的类似经历，可以查看本地文件系统存储选项在 
[ChartMuseum](https://chartmuseum.com/docs/#using-with-local-filesystem-storage) 
和 [servecm plugin](https://github.com/jdolitsky/helm-servecm).


### Library chart支持

Helm 3 支持的一类chart称为 “library chart”。 这是一个被其他chart共享的chart，
但是它自己不能创建发布组件。library chart的模板只能声明 `define` 元素。 全局范围
内的非`define`内容会被简单忽略。这允许用户复用和共享可在多个chart中重复使用的代码片段。
避免冗余和保留chart [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)。

Library chart在 Chart.yaml的依赖指令中声明，安装和管理与其他chart一致。

```yaml
dependencies:
  - name: mylib
    version: 1.x.x
    repository: quay.io
```

我们很高兴对开发者开放了这个特性的使用案例，以及library charts使用的最佳实践。

### Chart.yaml api版本切换

随着对library chart的支持以及requirements.yaml合并到Chart.yaml，客户端可以识别Helm 2的包格式而不理解
这些新特性。因此我们将Chart.yaml的apiVersion从 `v1` 切换到了 `v2`。

`helm create` 现在使用使用新格式创建chart，默认的apiVersion也切换到了这里。

客户端希望同时支持两个版本，进而能够检查Chart.yaml的`apiVersion`字段去理解如何解析包格式。

### XDG 基本目录支持

[XDG 基本目录规范](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)
是一个定义了配置、数据和缓存文件应该存储在文件系统什么位置的可移植标准。

Helm 2中，Helm 将所有的信息存储在 `~/.helm` (被亲切地称为`helm home`)，可以通过设置 `$HELM_HOME` 
环境变量修改，或者使用全局参数 `--home`。

Helm 3中，Helm现在遵守XDG 基本目录规范使用以下环境变量：

- `$XDG_CACHE_HOME`
- `$XDG_CONFIG_HOME`
- `$XDG_DATA_HOME`

Helm 插件仍然使用 `$HELM_HOME` 作为 `$XDG_DATA_HOME` 的别名，以便希望将`$HELM_HOME`作为过渡环境的变量来保证向后兼容性。

一些新的环境变量也通过插件环境变量来兼容以下更改：

- `$HELM_PATH_CACHE` 针对缓存路径
- `$HELM_PATH_CONFIG` 针对配置路径
- `$HELM_PATH_DATA` 针对data路径

Helm插件期望支持Heml 3 就建议使用新的环境变量。

### CLI 命令重新命名

In order to better align the verbiage from other package managers, `helm delete`
was re-named to `helm uninstall`. `helm delete` is still retained as an alias to
`helm uninstall`, so either form can be used.

In Helm 2, in order to purge the release ledger, the `--purge` flag had to be
provided. This functionality is now enabled by default. To retain the previous
behavior, use `helm uninstall --keep-history`.

Additionally, several other commands were re-named to accommodate the same
conventions:

- `helm inspect` -> `helm show`
- `helm fetch` -> `helm pull`

These commands have also retained their older verbs as aliases, so you can
continue to use them in either form.

### 自动创建namespace

When creating a release in a namespace that does not exist, Helm 2 created the
namespace.  Helm 3 follows the behavior of other Kubernetes tooling and returns
an error if the namespace does not exist.  Helm 3 will create the namespace if
you explicitly specify `--create-namespace` flag.

## 安装

### Why aren't there native packages of Helm for Fedora and other Linux distros?

The Helm project does not maintain packages for operating systems and
environments. The Helm community may provide native packages and if the Helm
project is made aware of them they will be listed. This is how the Homebrew
formula was started and listed. If you're interested in maintaining a package,
we'd love it.

### Why do you provide a `curl ...|bash` script?

There is a script in our repository (`scripts/get-helm-3`) that can be executed
as a `curl ..|bash` script. The transfers are all protected by HTTPS, and the
script does some auditing of the packages it fetches. However, the script has
all the usual dangers of any shell script.

We provide it because it is useful, but we suggest that users carefully read the
script first. What we'd really like, though, are better packaged releases of
Helm.

### How do I put the Helm client files somewhere other than their defaults?

Helm uses the XDG structure for storing files. There are environment variables
you can use to override these locations:

- `$XDG_CACHE_HOME`: set an alternative location for storing cached files.
- `$XDG_CONFIG_HOME`: set an alternative location for storing Helm
  configuration.
- `$XDG_DATA_HOME`: set an alternative location for storing Helm data.

Note that if you have existing repositories, you will need to re-add them with
`helm repo add...`.


## 卸载

### 我想删除我本地Helm. 全部文件在什么位置？

Along with the `helm` binary, Helm stores some files in the following locations:

- $XDG_CACHE_HOME
- $XDG_CONFIG_HOME
- $XDG_DATA_HOME

The following table gives the default folder for each of these, by OS:

| Operating System | Cache Path                  | Configuration Path               | Data Path                 |
|------------------|-----------------------------|----------------------------------|---------------------------|
| Linux            | `$HOME/.cache/helm `        | `$HOME/.config/helm `            | `$HOME/.local/share/helm` |
| macOS            | `$HOME/Library/Caches/helm` | `$HOME/Library/Preferences/helm` | `$HOME/Library/helm `     |
| Windows          | `%TEMP%\helm  `             | `%APPDATA%\helm `                | `%APPDATA%\helm`          |

## 故障排除

### On GKE (Google Container Engine) I get "No SSH tunnels currently open"

```
Error: Error forwarding ports: error upgrading connection: No SSH tunnels currently open. Were the targets able to accept an ssh-key for user "gke-[redacted]"?
```

Another variation of the error message is:


```
Unable to connect to the server: x509: certificate signed by unknown authority
```

The issue is that your local Kubernetes config file must have the correct
credentials.

When you create a cluster on GKE, it will give you credentials, including SSL
certificates and certificate authorities. These need to be stored in a
Kubernetes config file (Default: `~/.kube/config` so that `kubectl` and `helm`
can access them.

### After migration from Helm 2, `helm list` shows only some (or none) of my releases

It is likely that you have missed the fact that Helm 3 now uses cluster
namespaces throughout to scope releases. This means that for all commands
referencing a release you must either:

* rely on the current namespace in the active kubernetes context (as described
  by the `kubectl config view --minify` command),
* specify the correct namespace using the `--namespace`/`-n` flag, or
* for the `helm list` command, specify the `--all-namespaces`/`-A` flag

This applies to `helm ls`, `helm uninstall`, and all other `helm` commands
referencing a release.
