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

在Helm 3中，是用老的manifest生成新的补丁，活动状态和新的manifest。Helm 意识到老的状态是3，而现有活动状态是0，
并且新的manifest希望改回3，因此会生成一个补丁将状态改回3。

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

在Helm 2中，Helm 在新老manifest之间生成了一个`containers`对象的补丁。生成补丁的过程中不考虑集群的活动状态。

集群的活动状态被修改成了这样:

```yaml
containers:
- name: server
  image: nginx:2.1.0
```

sidecar pod从活动状态中移除了。更多的恐慌袭来。

在Helm 3中，Helm 在新的manifest、活动状态和老manifest之间生成了一个`containers`对象的补丁。会注意到新的manifest
将镜像tag更新为`2.1.0`, 但是活动状态中包含了一个sidecar容器。

集群的活动状态被修改成了下面这样：

```yaml
containers:
- name: server
  image: nginx:2.1.0
- name: my-injected-sidecar
  image: my-cool-mesh:1.0.0
```

### 发布名称现在限制在namespace范围内

&emsp;&emsp;随着Tiller的移除， 每个版本的信息需要保存在某个地方。在Helm 2中，是存储在Tiller相同的命名空间中。 
实际上这意味着一个发布版本使用一个名称，其他发布不能使用相同的名称，即使在不同的命名空间中也不行。

&emsp;&emsp;在Helm 3中，特定的版本信息作为发布本身存储在相同的命名空间中。意味着用户现在可以在两个分开的命名空间中使用
`helm install wordpress stable/wordpress`，并且每个都能使用 `helm list` 改变当前命名空间。
 (例如 `helm list --namespace foo`)。

&emsp;&emsp;与本地集群命名空间更好的一致性，使得 `helm list` 命令不再需要默认列出所有发布版本的列表。
取而代之的是，仅仅会在命名空间中列出当前kubernetes上下文的版本。
(也就是说运行`kubectl config view --minify`时会显示命名空间). 也就意味着您在执行`helm list`时必须提供
 `--all-namespaces` 标识才能获得和Helm 2同样的结果。

### 作为默认存储器的密钥

In Helm 3, 密钥被作为[默认存储驱动](/docs/topics/advanced/#storage-backends)使用。
Helm 2默认使用ConfigMaps记录版本信息。在Helm 2.7.0中，新的存储后台使用密钥来存储版本信息，
现在是Helm 3的默认设置。

Changing to Secrets as the Helm 3 default allows for additional security in
protecting charts in conjunction with the release of Secret encryption in
Kubernetes.

[Encrypting secrets at
rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) became
available as an alpha feature in Kubernetes 1.7 and became stable as of
Kubernetes 1.13. This allows users to encrypt Helm release metadata at rest, and
so it is a good starting point that can be expanded later into using something
like Vault.

### 更改了Go的path导入

In Helm 3, Helm switched the Go import path over from `k8s.io/helm` to
`helm.sh/helm/v3`. If you intend to upgrade to the Helm 3 Go client libraries,
make sure to change your import paths.

### Capabilities

The `.Capabilities` built-in object available during the rendering stage has
been simplified.

[Built-in Objects](/docs/chart_template_guide/builtin_objects/)

### 使用Json格式验证Chart Values

A JSON Schema can now be imposed upon chart values. This ensures that values
provided by the user follow the schema laid out by the chart maintainer,
providing better error reporting when the user provides an incorrect set of
values for a chart.

Validation occurs when any of the following commands are invoked:

* `helm install`
* `helm upgrade`
* `helm template`
* `helm lint`

See the documentation on [Schema files](/docs/topics/charts#schema-files) for
more information.

### Consolidation of `requirements.yaml` into `Chart.yaml`

The Chart dependency management system moved from requirements.yaml and
requirements.lock to Chart.yaml and Chart.lock. We recommend that new charts
meant for Helm 3 use the new format. However, Helm 3 still understands Chart API
version 1 (`v1`) and will load existing `requirements.yaml` files

In Helm 2, this is how a `requirements.yaml` looked:

```yaml
dependencies:
- name: mariadb
  version: 5.x.x
  repository: https://kubernetes-charts.storage.googleapis.com/
  condition: mariadb.enabled
  tags:
    - database
```

In Helm 3, the dependency is expressed the same way, but now from your
`Chart.yaml`:

```yaml
dependencies:
- name: mariadb
  version: 5.x.x
  repository: https://kubernetes-charts.storage.googleapis.com/
  condition: mariadb.enabled
  tags:
    - database
```

Charts are still downloaded and placed in the `charts/` directory, so subcharts
vendored into the `charts/` directory will continue to work without
modification.

### Name (or --generate-name) is now required on install

In Helm 2, if no name was provided, an auto-generated name would be given. In
production, this proved to be more of a nuisance than a helpful feature. In Helm
3, Helm will throw an error if no name is provided with `helm install`.

For those who still wish to have a name auto-generated for you, you can use the
`--generate-name` flag to create one for you.

### Pushing Charts to OCI Registries

This is an experimental feature introduced in Helm 3. To use, set the
environment variable `HELM_EXPERIMENTAL_OCI=1`.

At a high level, a Chart Repository is a location where Charts can be stored and
shared. The Helm client packs and ships Helm Charts to a Chart Repository.
Simply put, a Chart Repository is a basic HTTP server that houses an index.yaml
file and some packaged charts.

While there are several benefits to the Chart Repository API meeting the most
basic storage requirements, a few drawbacks have started to show:

- Chart Repositories have a very hard time abstracting most of the security
  implementations required in a production environment. Having a standard API
  for authentication and authorization is very important in production
  scenarios.
- Helm’s Chart provenance tools used for signing and verifying the integrity and
  origin of a chart are an optional piece of the Chart publishing process.
- In multi-tenant scenarios, the same Chart can be uploaded by another tenant,
  costing twice the storage cost to store the same content. Smarter chart
  repositories have been designed to handle this, but it’s not a part of the
  formal specification.
- Using a single index file for search, metadata information, and fetching
  Charts has made it difficult or clunky to design around in secure multi-tenant
  implementations.

Docker’s Distribution project (also known as Docker Registry v2) is the
successor to the Docker Registry project. Many major cloud vendors have a
product offering of the Distribution project, and with so many vendors offering
the same product, the Distribution project has benefited from many years of
hardening, security best practices, and battle-testing.

Please have a look at `helm help chart` and `helm help registry` for more
information on how to package a chart and push it to a Docker registry.

For more info, please see [this page](/docs/topics/registries/).

### 移除了`helm serve`

`helm serve` ran a local Chart Repository on your machine for development
purposes. However, it didn't receive much uptake as a development tool and had
numerous issues with its design. In the end, we decided to remove it and split
it out as a plugin.

For a similar experience to `helm serve`, have a look at the local filesystem
storage option in
[ChartMuseum](https://chartmuseum.com/docs/#using-with-local-filesystem-storage)
and the [servecm plugin](https://github.com/jdolitsky/helm-servecm).


### Library chart支持

Helm 3 supports a class of chart called a “library chart”. This is a chart that
is shared by other charts, but does not create any release artifacts of its own.
A library chart’s templates can only declare `define` elements. Globally scoped
non-`define` content is simply ignored. This allows users to re-use and share
snippets of code that can be re-used across many charts, avoiding redundancy and
keeping charts [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).

Library charts are declared in the dependencies directive in Chart.yaml, and are
installed and managed like any other chart.

```yaml
dependencies:
  - name: mylib
    version: 1.x.x
    repository: quay.io
```

We’re very excited to see the use cases this feature opens up for chart
developers, as well as any best practices that arise from consuming library
charts.

### Chart.yaml api版本 bump

With the introduction of library chart support and the consolidation of
requirements.yaml into Chart.yaml, clients that understood Helm 2's package
format won't understand these new features. So, we bumped the apiVersion in
Chart.yaml from `v1` to `v2`.

`helm create` now creates charts using this new format, so the default
apiVersion was bumped there as well.

Clients wishing to support both versions of Helm charts should inspect the
`apiVersion` field in Chart.yaml to understand how to parse the package format.

### XDG Base Directory Support

[The XDG Base Directory
Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)
is a portable standard defining where configuration, data, and cached files
should be stored on the filesystem.

In Helm 2, Helm stored all this information in `~/.helm` (affectionately known
as `helm home`), which could be changed by setting the `$HELM_HOME` environment
variable, or by using the global flag `--home`.

In Helm 3, Helm now respects the following environment variables as per the XDG
Base Directory Specification:

- `$XDG_CACHE_HOME`
- `$XDG_CONFIG_HOME`
- `$XDG_DATA_HOME`

Helm plugins are still passed `$HELM_HOME` as an alias to `$XDG_DATA_HOME` for
backwards compatibility with plugins looking to use `$HELM_HOME` as a scratchpad
environment.

Several new environment variables are also passed in to the plugin's environment
to accommodate this change:

- `$HELM_PATH_CACHE` for the cache path
- `$HELM_PATH_CONFIG` for the config path
- `$HELM_PATH_DATA` for the data path

Helm plugins looking to support Helm 3 should consider using these new
environment variables instead.

### CLI Command Renames

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

### Automatically creating namespaces

When creating a release in a namespace that does not exist, Helm 2 created the
namespace.  Helm 3 follows the behavior of other Kubernetes tooling and returns
an error if the namespace does not exist.  Helm 3 will create the namespace if
you explicitly specify `--create-namespace` flag.

## Installing

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

## Troubleshooting

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
