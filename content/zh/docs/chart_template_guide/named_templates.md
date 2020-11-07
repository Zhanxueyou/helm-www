---
title: "命名模板"
description: "如何定义命名模板"
weight: 9
---

此时需要越过模板，开始创建其他内容了。该部分我们会看到如何在一个文件中定义
_命名模板_，并在其他地方使用。_命名模板_ (有时称作一个 _部分_ 或一个
_子模板_)仅仅是在文件内部定义的模板，并使用了一个名字。有两种创建方式和几种不同的使用方法。

在[流控制](http://helm.sh/docs/chart_template_guide/control_structures)部分，
我们介绍了三种声明和管理模板的方法：`define`，`template`，和`block`。在这部分，我们将使用者三种操作并介绍一种特殊用途的
`include`方法，类似于`template`操作。

命名模板时要记住一个重要细节：**模板名称是全局的**。如果您想声明两个相同名称的模板，哪个最后加载就使用哪个。
因为在子chart中的模板和顶层模板一起编译，命名时要注意 _chart特定名称_。

一个常见的命名惯例是用chart名称作为模板前缀：`{{ define "mychart.labels" }}`。使用特定chart名称作为前缀可以避免可能因为
两个不同chart使用了相同名称的模板而引起的冲突。

## 局部的和`_`文件

目前为止，我们已经使用了单个文件，且单个文件中包含了单个模板。但Helm的模板语言允许你创建命名的嵌入式模板，
这样就可以在其他位置按名称访问。

在编写模板细节之前，文件的命名惯例需要注意：

* `templates/`中的大多数文件被视为包含Kubernetes清单
* `NOTES.txt`是个例外
* 命名以下划线(`_`)开始的文件则假定 _没有_ 包含清单内容。这些文件不会渲染为Kubernetes对象定义，但在其他chart模板中都可用。

这些文件用来存储局部和辅助对象，实际上当我们第一次创建`mychart`时，会看到一个名为`_helpers.tpl`的文件，这个文件时模板局部的默认位置。

## 用`define`和`template`声明和使用模板

`define`操作允许我们在模板文件中创建一个命名模板，语法如下：

```yaml
{{ define "MY.NAME" }}
  # body of template here
{{ end }}
```

比如我们可以定义一个模板封装Kubernetes的标签：

```yaml
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

现在我们将模板嵌入到了已有的配置映射中，然后使用`template`包含进来：

```yaml
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

当模板引擎读取该文件时，它会存储`mychart.labels`的引用直到`template "mychart.labels"`被调用。
然后会按行渲染模板，因此结果类似这样：

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: running-panda-configmap
  labels:
    generator: helm
    date: 2016-11-02
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

注意：`define`不会有输出，除非像本示例一样用模板调用它。

按照惯例，Helm chart将这些模板放置在局部文件中，一般是`_helpers.tpl`。把这个方法移到那里：

```yaml
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

按照惯例`define`方法会有个简单的文档块(`{{/* ... */}}`)来描述要做的事。

尽管这个定义是在`_helpers.tpl`中，但它仍能访问`configmap.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

如上所述，**模板名称是全局的**。 As a result of this, if two
templates are declared with the same name the last occurrence will be the one
that is used. Since templates in subcharts are compiled together with top-level
templates, it is best to name your templates with _chart specific names_. A
popular naming convention is to prefix each defined template with the name of
the chart: `{{ define "mychart.labels" }}`.

## 设置模板范围

In the template we defined above, we did not use any objects. We just used
functions. Let's modify our defined template to include the chart name and chart
version:

```yaml
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
{{- end }}
```

If we render this, the result will not be what we expect:

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: moldy-jaguar-configmap
  labels:
    generator: helm
    date: 2016-11-02
    chart:
    version:
```

What happened to the name and version? They weren't in the scope for our defined
template. When a named template (created with `define`) is rendered, it will
receive the scope passed in by the `template` call. In our example, we included
the template like this:

```yaml
{{- template "mychart.labels" }}
```

No scope was passed in, so within the template we cannot access anything in `.`.
This is easy enough to fix, though. We simply pass a scope to the template:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" . }}
```

Note that we pass `.` at the end of the `template` call. We could just as easily
pass `.Values` or `.Values.favorite` or whatever scope we want. But what we want
is the top-level scope.

Now when we execute this template with `helm install --dry-run --debug
plinking-anaco ./mychart`, we get this:

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: plinking-anaco-configmap
  labels:
    generator: helm
    date: 2016-11-02
    chart: mychart
    version: 0.1.0
```

Now `{{ .Chart.Name }}` resolves to `mychart`, and `{{ .Chart.Version }}`
resolves to `0.1.0`.

## The `include` function

Say we've defined a simple template that looks like this:

```yaml
{{- define "mychart.app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}"
{{- end -}}
```

Now say I want to insert this both into the `labels:` section of my template,
and also the `data:` section:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{ template "mychart.app" . }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ template "mychart.app" . }}
```

The output will not be what we expect:

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: measly-whippet-configmap
  labels:
    app_name: mychart
app_version: "0.1.0+1478129847"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
app_version: "0.1.0+1478129847"
```

Note that the indentation on `app_version` is wrong in both places. Why? Because
the template that is substituted in has the text aligned to the right. Because
`template` is an action, and not a function, there is no way to pass the output
of a `template` call to other functions; the data is simply inserted inline.

To work around this case, Helm provides an alternative to `template` that will
import the contents of a template into the present pipeline where it can be
passed along to other functions in the pipeline.

Here's the example above, corrected to use `indent` to indent the `mychart_app`
template correctly:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{ include "mychart.app" . | indent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ include "mychart.app" . | indent 2 }}
```

Now the produced YAML is correctly indented for each section:

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-mole-configmap
  labels:
    app_name: mychart
    app_version: "0.1.0+1478129987"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
  app_version: "0.1.0+1478129987"
```

> It is considered preferable to use `include` over `template` in Helm templates
> simply so that the output formatting can be handled better for YAML documents.

Sometimes we want to import content, but not as templates. That is, we want to
import files verbatim. We can achieve this by accessing files through the
`.Files` object described in the next section.
