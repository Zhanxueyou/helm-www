---
title: "弃用的 Kubernetes API"
description: "解释Helm不推荐使用的Kubernetes API"
---

Kubernetes是一个API驱动系统，且API会随着时间的推移而变化，以反映对问题理解的不断推移。这是系统及API的普遍做法。
API推移的一个重要部分时良好的弃用策略和通知用户更改API是如何实现的。换句话说，你的API使用者需要提前知道要发布的API
删除或更改了什么。这消除了重大改变对用户造成的恐惧。

[Kubernetes弃用策略](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)
文档描述了如何处理API版本的变化。弃用策略声明了在弃用声明之后支持的API版本的时间范围。因此关注弃用声明并知道API何时被移除很重要。
有助于将影响降到最低。

这是一个声明示例， [针对Kubernetes 1.6弃用的API版本](https://kubernetes.io/blog/2019/07/18/api-deprecations-in-1-16/)，
而且是在版本发布的几个月之前公布。在这之前，这些API版本可能已经宣布不再使用了。这表明一个好的策略可以通知用户API的版本支持。

Helm模板定义Kubernetes对象时指定了一个 [Kubernetes
API组](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-groups)，类似于Kubernetes的manifest文件。
在模板的`apiVersion`字段指定并标识了Kubernetes对象的API版本。这意味着Helm用户和chart维护者需要关注Kubernetes的API版本
何时会被弃用且在哪个Kubernetes版本中被移除。

## Chart Maintainers

你应该审核chart，检查Kubernetes中已弃用或已删除的Kubernetes API版本。如果API版本不再被支持，应该更新为支持版本并发布新的
chart版本。API版本由`kind`和`apiVersion`字段定义。比如，Kubernetes 1.16 中有个被移除的`Deployment`对象API版本：

```yaml
apiVersion: apps/v1beta1
kind: Deployment
```

## Helm用户

你应该审核你使用的chart(类似于[chart维护者](#chart-maintainers))，并识别所有的chart中Kubernetes版本弃用或移除的API版本。
针对确定的chart，需要检查（有支持的API版本的）chart最新的版本，或者手动更新。

另外，你还需要审核已经部署的chart（即Helm版本）还有没有弃用或移除的API版本。可以使用`helm get manifest`获取详细信息。

将Helm更新为支持的API取决于你通过以下方式找到的：

1. 如果你只找到弃用的API版本，则：

    - 执行`helm upgrade`升级Kubernetes API版本支持的chart版本
    - 在升级中添加一个描述，在当前版本之前不执行Helm版本回滚

2. 如果你发现了在Kubernetes版本中被移除的API版本，则：

    - 如果你运行的Kubernetes版本中API版本依然可用（比如，你在Kubernetes 1.15 且你发现使用的API会在1.16中移除）：
      - 遵循第1步的步骤
    - 否则（比如，你运行的Kubernetes 版本中某些API版本通过`helm get manifest`显示不可用）：
      - 需要编辑存储在集群中的版本清单，更新API版本到支持的API。查看[更新版本清单的API版本](#updating-api-versions-of-a-release-manifest)

> 注意：在所有使用支持的API更新Helm版本的场景中，决不应该将发布版本回滚到API版本支持的之前的版本

> 建议：最佳实践是将正在使用的弃用版本升级到支持的API版本，在升级Kubernetes 集群之前删除这些API版本。

If you don't update a release as suggested previously, you will have an error
similar to the following when trying to upgrade a release in a Kubernetes
version where its API version(s) is/are removed:

```
Error: UPGRADE FAILED: current release manifest contains removed kubernetes api(s)
for this kubernetes version and it is therefore unable to build the kubernetes
objects for performing the diff. error from kubernetes: unable to recognize "":
no matches for kind "Deployment" in version "apps/v1beta1"
```

Helm fails in this scenario because it attempts to create a diff patch between
the current deployed release (which contains the Kubernetes APIs that are
removed in this Kubernetes version) against the chart you are passing with the
updated/supported API versions. The underlying reason for failure is that when
Kubernetes removes an API version, the Kubernetes Go client library can no
longer parse the deprecated objects and Helm therefore fails when calling the
library. Helm unfortunately is unable to recover from this situation and is no
longer able to manage such a release. See [Updating API Versions of a Release
Manifest](#updating-api-versions-of-a-release-manifest) for more details on how
to recover from this scenario.

## Updating API Versions of a Release Manifest

The manifest is a property of the Helm release object which is stored in the
data field of a Secret (default) or ConfigMap in the cluster. The data field
contains a gzipped object which is base 64 encoded (there is an additional base
64 encoding for a Secret). There is a Secret/ConfigMap per release
version/revision in the namespace of the release.

You can use the Helm [mapkubeapis](https://github.com/hickeyma/helm-mapkubeapis)
plugin to perform the update of a release to supported APIs. Check out the
readme for more details.

Alternatively, you can follow these manual steps to perform an update of the API
versions of a release manifest. Depending on your configuration you will follow
the steps for the Secret or ConfigMap backend.

- Get the name of the Secret or Configmap associated with the latest deployed
  release:
  - Secrets backend: `kubectl get secret -l
    owner=helm,status=deployed,name=<release_name> --namespace
    <release_namespace> | awk '{print $1}' | grep -v NAME`
  - ConfigMap backend: `kubectl get configmap -l
    owner=helm,status=deployed,name=<release_name> --namespace
    <release_namespace> | awk '{print $1}' | grep -v NAME`
- Get latest deployed release details:
  - Secrets backend: `kubectl get secret <release_secret_name> -n
    <release_namespace> -o yaml > release.yaml`
  - ConfigMap backend: `kubectl get configmap <release_configmap_name> -n
    <release_namespace> -o yaml > release.yaml`
- Backup the release in case you need to restore if something goes wrong:
  - `cp release.yaml release.bak`
  - In case of emergency, restore: `kubectl apply -f release.bak -n
    <release_namespace>`
- Decode the release object: 
  - Secrets backend:`cat release.yaml | grep -oP '(?<=release: ).*' | base64 -d
    | base64 -d | gzip -d > release.data.decoded`
  - ConfigMap backend: `cat release.yaml | grep -oP '(?<=release: ).*' | base64
    -d | gzip -d > release.data.decoded`
- Change API versions of the manifests. Can use any tool (e.g. editor) to make
  the changes. This is in the `manifest` field of your decoded release object
  (`release.data.decoded`)
- Encode the release object:
  - Secrets backend: `cat release.data.decoded | gzip | base64 | base64`
  - ConfigMap backend: `cat release.data.decoded | gzip | base64`
- Replace `data.release` property value in the deployed release file
  (`release.yaml`) with the new encoded release object
- Apply file to namespace: `kubectl apply -f release.yaml -n
  <release_namespace>`
- Perform a `helm upgrade` with a version of the chart with supported Kubernetes
  API versions
- Add a description in the upgrade, something along the lines to not perform a
  rollback to a Helm version prior to this current version
