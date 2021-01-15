---
title: "从这里开始吧"
weight: 2
description: "Chart模板的快速指南。"
---

指南的该部分，我们会创建一个chart并添加第一个模板。创建的chart会在后续指南中用到。

接下来，让我们简单看一下Helm chart。

## Charts

如[Charts 指南](https://helm.sh/zh/docs/topics/charts)所述， Helm chart的结构如下：

```shell
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```

`templates/` 目录包括了模板文件。当Helm评估chart时，会通过模板渲染引擎将所有文件发送到`templates/`目录中。
然后收集模板的结果并发送给Kubernetes。

`values.yaml` 文件也导入到了模板。这个文件包含了chart的 _默认值_ 。这些值会在用户执行`helm install` 或 `helm upgrade`时被覆盖。

`Chart.yaml` 文件包含了该chart的描述。你可以从模板中访问它。`charts/`目录 _可以_ 包含其他的chart(称之为 _子chart_)。
指南稍后我们会看到当涉及模板渲染时这些是如何工作的。

## 入门 Chart

在本指南中我们会创建一个名为`mychart`的chart，然后会在chart中创建一些模板。

```console
$ helm create mychart
Creating mychart
```

### 快速查看 `mychart/templates/`

如果你看看 `mychart/templates/` 目录，会注意到一些文件已经存在了：

- `NOTES.txt`: chart的"帮助文本"。这会在你的用户执行`helm install`时展示给他们。
- `deployment.yaml`: 创建Kubernetes
  [工作负载](https://kubernetes.io/docs/user-guide/deployments/)的基本清单
- `service.yaml`: 为你的工作负载创建一个[service终端](https://kubernetes.io/docs/user-guide/services/)基本清单。
- `_helpers.tpl`: 放置可以通过chart复用的模板辅助对象

然后我们要做的是... _把它们全部删掉！_ 这样我们就可以从头开始学习我们的教程。我们在开始时会创造自己的`NOTES.txt`和`_helpers.tpl`。

```console
$ rm -rf mychart/templates/*
```

编制生产环境级别的chart时，有这些chart的基础版本会很有用。因此在日常编写中，你可能不想删除它们。

## 第一个模板

第一个创建的模板是`ConfigMap`。Kubernetes中，配置映射只是用于存储配置数据的容器。其他组件，比如pod，可以访问配置映射中的数据。

因为配置映射是基础资源，对我们来说是很好的起点。

让我们以创建一个名为 `mychart/templates/configmap.yaml`的文件开始：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

**TIP:** Template names do not follow a rigid naming pattern. However, we
recommend using the suffix `.yaml` for YAML files and `.tpl` for helpers.

The YAML file above is a bare-bones ConfigMap, having the minimal necessary
fields. In virtue of the fact that this file is in the `mychart/templates/`
directory, it will be sent through the template engine.

It is just fine to put a plain YAML file like this in the `mychart/templates/`
directory. When Helm reads this template, it will simply send it to Kubernetes
as-is.

With this simple template, we now have an installable chart. And we can install
it like this:

```console
$ helm install full-coral ./mychart
NAME: full-coral
LAST DEPLOYED: Tue Nov  1 17:36:01 2016
NAMESPACE: default
STATUS: DEPLOYED
REVISION: 1
TEST SUITE: None
```

Using Helm, we can retrieve the release and see the actual template that was
loaded.

```console
$ helm get manifest full-coral

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

The `helm get manifest` command takes a release name (`full-coral`) and prints
out all of the Kubernetes resources that were uploaded to the server. Each file
begins with `---` to indicate the start of a YAML document, and then is followed
by an automatically generated comment line that tells us what template file
generated this YAML document.

From there on, we can see that the YAML data is exactly what we put in our
`configmap.yaml` file.

Now we can uninstall our release: `helm uninstall full-coral`.

### Adding a Simple Template Call

Hard-coding the `name:` into a resource is usually considered to be bad
practice. Names should be unique to a release. So we might want to generate a
name field by inserting the release name.

**TIP:** The `name:` field is limited to 63 characters because of limitations to
the DNS system. For that reason, release names are limited to 53 characters.
Kubernetes 1.3 and earlier limited to only 24 characters (thus 14 character
names).

Let's alter `configmap.yaml` accordingly.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

The big change comes in the value of the `name:` field, which is now
`{{ .Release.Name }}-configmap`.

> A template directive is enclosed in `{{` and `}}` blocks.

The template directive `{{ .Release.Name }}` injects the release name into the
template. The values that are passed into a template can be thought of as
_namespaced objects_, where a dot (`.`) separates each namespaced element.

The leading dot before `Release` indicates that we start with the top-most
namespace for this scope (we'll talk about scope in a bit). So we could read
`.Release.Name` as "start at the top namespace, find the `Release` object, then
look inside of it for an object called `Name`".

The `Release` object is one of the built-in objects for Helm, and we'll cover it
in more depth later. But for now, it is sufficient to say that this will display
the release name that the library assigns to our release.

Now when we install our resource, we'll immediately see the result of using this
template directive:

```console
$ helm install clunky-serval ./mychart
NAME: clunky-serval
LAST DEPLOYED: Tue Nov  1 17:45:37 2016
NAMESPACE: default
STATUS: DEPLOYED
REVISION: 1
TEST SUITE: None
```

You can run `helm get manifest clunky-serval` to see the entire generated YAML.

Note that the config map inside kubernetes name is `clunky-serval-configmap`
instead of `mychart-configmap` previously.

At this point, we've seen templates at their most basic: YAML files that have
template directives embedded in `{{` and `}}`. In the next part, we'll take a
deeper look into templates. But before moving on, there's one quick trick that
can make building templates faster: When you want to test the template
rendering, but not actually install anything, you can use `helm install --debug
--dry-run goodly-guppy ./mychart`. This will render the templates. But instead
of installing the chart, it will return the rendered template to you so you can
see the output:

```console
$ helm install --debug --dry-run goodly-guppy ./mychart
install.go:149: [debug] Original chart version: ""
install.go:166: [debug] CHART PATH: /Users/ninja/mychart

NAME: goodly-guppy
LAST DEPLOYED: Thu Dec 26 17:24:13 2019
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
affinity: {}
fullnameOverride: ""
image:
  pullPolicy: IfNotPresent
  repository: nginx
imagePullSecrets: []
ingress:
  annotations: {}
  enabled: false
  hosts:
  - host: chart-example.local
    paths: []
  tls: []
nameOverride: ""
nodeSelector: {}
podSecurityContext: {}
replicaCount: 1
resources: {}
securityContext: {}
service:
  port: 80
  type: ClusterIP
serviceAccount:
  create: true
  name: null
tolerations: []

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: goodly-guppy-configmap
data:
  myvalue: "Hello World"

```

Using `--dry-run` will make it easier to test your code, but it won't ensure
that Kubernetes itself will accept the templates you generate. It's best not to
assume that your chart will install just because `--dry-run` works.

In the [Chart Template Guide](_index.md), we take the basic chart we defined
here and explore the Helm template language in detail. And we'll get started
with built-in objects.
