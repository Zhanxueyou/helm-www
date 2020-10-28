---
title: "库类型Chart"
description: "阐述库类型chart及使用案例"
weight: 4
---

库类型chart是一种[Helm chart](https://helm.sh/zh/docs/topics/charts)，定义了可以由其他chart中Helm
模板共享的chart原语或定义。这允许用户通过chart分享可复用得代码片段来避免重复并保持chart
[干燥](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)。

在Helm 3中引用了库chart，从形式上区别于Helm 2中chart维护的通用或辅助chart。
作为一个chart类型引入，可以提供：
- 一种明确区分通用和应用chart的方法
- 逻辑上阻止安装通用chart
- 通用chart中的未渲染模板可以包含版本组件

chart维护者可以定义一个通用的chart作为库并且现在可以确信Helm将以标准一致的方式处理chart。
也意味着通过改变chart类型来分享应用chart中的定义。

## 创建一个简单的库chart

像之前提到的，库chart是一种[Helm chart](https://helm.sh/zh/docs/topics/charts)类型。意味着你可以从创建脚手架chart开始：

```console
$ helm create mylibchart
Creating mylibchart
```

本示例中创建自己的模板需要先删除`templates`目录中的所有文件。

```console
$ rm -rf mylibchart/templates/*
```

不再需要values文件。

```console
$ rm -f mylibchart/values.yaml 
```

在创建通用代码之前，先快速回顾一下相关Helm概念。[已命名的模板](https://helm.sh/docs/chart_template_guide/named_templates/)
(有时称为局部模板或子模板)是定义在一个文件中的简单模板，并分配了一个名称。在`templates/`目录中，
所有以下划线开始的文件(_)不会输出到Kubernetes清单文件中。因此依照惯例，辅助模板和局部模板被放置在`_*.tpl`或`_*.yaml`文件中。 

这个示例中，我们要写一个通用的配置映射来创建一个空的配置映射源。在`mylibchart/templates/_configmap.yaml`文件中定义如下：

```yaml
{{- define "mylibchart.configmap.tpl" -}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name | printf "%s-%s" .Chart.Name }}
data: {}
{{- end -}}
{{- define "mylibchart.configmap" -}}
{{- include "mylibchart.util.merge" (append . "mylibchart.configmap.tpl") -}}
{{- end -}}
```

这个配置映射结构被定义在名为`mylibchart.configmap.tpl`的模板文件中。`data`是一个空源的配置映射，
这个文件中另一个命名的模板是`mylibchart.configmap`。这个模板包含了另一个模板`mylibchart.util.merge`，
会使用两个命名的模板作为参数，称为`mylibchart.configmap`和`mylibchart.configmap.tpl`。

复制方法`mylibchart.util.merge`是`mylibchart/templates/_util.yaml`文件中的一个命名模板。
是[通用Helm辅助Chart](#the-common-helm-helper-chart)的实用工具。因为它合并了两个模板并覆盖了两个模板的公共部分。

```yaml
{{- /*
mylibchart.util.merge will merge two YAML templates and output the result.
This takes an array of three values:
- the top context
- the template name of the overrides (destination)
- the template name of the base (source)
*/ -}}
{{- define "mylibchart.util.merge" -}}
{{- $top := first . -}}
{{- $overrides := fromYaml (include (index . 1) $top) | default (dict ) -}}
{{- $tpl := fromYaml (include (index . 2) $top) | default (dict ) -}}
{{- toYaml (merge $overrides $tpl) -}}
{{- end -}}
```

当chart希望使用通过配置自定义其通用代码时，这一点就非常重要。

最后，将chart类型修改为`library`。需要按以下方式编辑`mylibchart/Chart.yaml`：

```yaml
apiVersion: v2
name: mylibchart
description: A Helm chart for Kubernetes

# A chart can be either an 'application' or a 'library' chart.
#
# Application charts are a collection of templates that can be packaged into versioned archives
# to be deployed.
#
# Library charts provide useful utilities or functions for the chart developer. They're included as
# a dependency of application charts to inject those utilities and functions into the rendering
# pipeline. Library charts do not define any templates and therefore cannot be deployed.
# type: application
type: library

# This is the chart version. This version number should be incremented each time you make changes
# to the chart and its templates, including the app version.
version: 0.1.0

# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application.
appVersion: 1.16.0
```

这个库chart现在可以分享了，并且配置映射定义可以复用了。

此时，有必要去检测一下chart是否变成了库chart：

```console
$ helm install mylibchart mylibchart/
Error: library charts are not installable
```

## 使用简单的库chart

It is time to use the library chart. This means creating a scaffold chart again:

```console
$ helm create mychart
Creating mychart
```

Lets clean out the template files again as we want to create a ConfigMap only:

```console
$ rm -rf mychart/templates/*
```

When we want to create a simple ConfigMap in a Helm template, it could look
similar to the following:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name | printf "%s-%s" .Chart.Name }}
data:
  myvalue: "Hello World"
```

We are however going to re-use the common code already created in `mylibchart`.
The ConfigMap can be created in the file `mychart/templates/configmap.yaml` as
follows:

```yaml
{{- include "mylibchart.configmap" (list . "mychart.configmap") -}}
{{- define "mychart.configmap" -}}
data:
  myvalue: "Hello World"
{{- end -}}
```

You can see that it simplifies the work we have to do by inheriting the common
ConfigMap definition which adds standard properties for ConfigMap. In our
template we add the configuration, in this case the data key `myvalue` and its
value. The configuration override the empty resource of the common ConfigMap.
This is feasible because of the helper function `mylibchart.util.merge` we
mentioned in the previous section.

To be able to use the common code, we need to add `mylibchart` as a dependency.
Add the following to the end of the file `mychart/Chart.yaml`:

```yaml
# My common code in my library chart
dependencies:
- name: mylibchart
  version: 0.1.0
  repository: file://../mylibchart
```

This includes the library chart as a dynamic dependency from the filesystem
which is at the same parent path as our application chart. As we are including
the library chart as a dynamic dependency, we need to run `helm dependency
update`. It will copy the library chart into your `charts/` directory.

```console
$ helm dependency update mychart/
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Deleting outdated charts
```

We are now ready to deploy our chart. Before installing, it is worth checking
the rendered template first.

```console
$ helm install mydemo mychart/ --debug --dry-run
install.go:159: [debug] Original chart version: ""
install.go:176: [debug] CHART PATH: /root/test/helm-charts/mychart

NAME: mydemo
LAST DEPLOYED: Tue Mar  3 17:48:47 2020
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
mylibchart:
  global: {}
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
  annotations: {}
  create: true
  name: null
tolerations: []

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
data:
  myvalue: Hello World
kind: ConfigMap
metadata:
  labels:
    app: mychart
    chart: mychart-0.1.0
    release: mydemo
  name: mychart-mydemo
```

This looks like the ConfigMap we want with data override of `myvalue: Hello
World`. Lets install it:

```console
$ helm install mydemo mychart/
NAME: mydemo
LAST DEPLOYED: Tue Mar  3 17:52:40 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

We can retrieve the release and see that the actual template was loaded.

```console
$ helm get manifest mydemo
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
data:
  myvalue: Hello World
kind: ConfigMap
metadata:
  labels:
    app: mychart
    chart: mychart-0.1.0
    release: mydemo
  name: mychart-mydemo
  ```

## The Common Helm Helper Chart

Bit of a mouth full of a title but this
[chart](https://github.com/helm/charts/tree/master/incubator/common) is the
original pattern for common charts. It provides utilities that reflect best
practices of Kubernetes chart development. Best of all it can be used off the
bat by you when developing your charts to give you handy shared code.

Here is a quick way to use it. For more details, have a look at the
[README](https://github.com/helm/charts/blob/master/incubator/common/README.md).

Create a scaffold chart again:

```console
$ helm create demo
Creating demo
```

Lets use the common code from the helper chart. First, edit deployment
`demo/templates/deployment.yaml` as follows:

```yaml
{{- template "common.deployment" (list . "demo.deployment") -}}
{{- define "demo.deployment" -}}
## Define overrides for your Deployment resource here, e.g.
spec:
  replicas: {{ .Values.replicaCount }}
{{- end -}}
```

现在这个service文件`demo/templates/service.yaml`变成了下面这样：

```yaml
{{- template "common.service" (list . "demo.service") -}}
{{- define "demo.service" -}}
## Define overrides for your Service resource here, e.g.
# metadata:
#   labels:
#     custom: label
# spec:
#   ports:
#   - port: 8080
{{- end -}}
```

These templates show how inheriting the common code from the helper chart
simplifies your coding down to your configuration or customization of the
resources.

To be able to use the common code, we need to add `common` as a dependency. Add
the following to the end of the file `demo/Chart.yaml`:

```yaml
dependencies:
- name: common
  version: "^0.0.5"
  repository: "https://kubernetes-charts-incubator.storage.googleapis.com/"
```

Note: You will need to add the `incubator` repo to the Helm repository list
(`helm repo add`).

As we are including the chart as a dynamic dependency, we need to run `helm
dependency update`. It will copy the helper chart into your `charts/` directory.

As helper chart is using some Helm 2 constructs, you will need to add the
following to `demo/values.yaml` to enable the `nginx` image to be loaded as this
was updated in Helm 3 scaffold chart:

```yaml
image:
  tag: 1.16.0
```

现在可以出发了，立刻行动吧！
