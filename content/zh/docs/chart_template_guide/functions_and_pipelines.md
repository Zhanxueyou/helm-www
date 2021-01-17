---
title: "模板函数和流水线"
description: "使用函数和管道"
weight: 5
---

到目前为止，我们已经知道了如何将信息放入模板中。 但是这些信息未被修改就放入了模板中。
有时我们希望以一种更有用的方式来转换所提供的数据。

让我们从一个最佳实践开始：将`.Values`对象中的字符串注入模板时，应该引用这些字符串。可以通过
调用模板指令中的`quote`函数来实现：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

模板函数遵循的语法是 `functionName arg1 arg2...`。在上面的代码片段中，`quote .Values.favorite.drink`
调用了`quote`函数并只传递了一个参数。

Helm 有超过60个可用函数。其中有些通过[Go模板语言](https://godoc.org/text/template)本身定义。
其他大部分都是[Sprig 模板库](https://masterminds.github.io/sprig/)。我们通过示例看到其中很多内容。

> 当我们讨论"Helm模板语言"时，好像它是特定于Helm的，实际上是由Go模板语言，一些额外的函数以及用于
向模板公开某些对象的包装器组合而成。很多Go模板的资源有助于你学习模板。

## 管道符

模板语言其中一个强大功能是 _管道_ 概念。借鉴UNIX的概念，管道符是将一系列的
模板语言紧凑地表示为一系列转换的工具。换句话说，管道符是按顺序完成一系列任务的有效方式。
现在用管道符重写上述示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | quote }}
```

In this example, instead of calling `quote ARGUMENT`, we inverted the order. We
"sent" the argument to the function using a pipeline (`|`):
`.Values.favorite.drink | quote`. Using pipelines, we can chain several
functions together:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

> Inverting the order is a common practice in templates. You will see `.val |
> quote` more often than `quote .val`. Either practice is fine.

When evaluated, that template will produce this:

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: trendsetting-p-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

Note that our original `pizza` has now been transformed to `"PIZZA"`.

When pipelining arguments like this, the result of the first evaluation
(`.Values.favorite.drink`) is sent as the _last argument to the function_. We
can modify the drink example above to illustrate with a function that takes two
arguments: `repeat COUNT STRING`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | repeat 5 | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

The `repeat` function will echo the given string the given number of times, so
we will get this for output:

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: melting-porcup-configmap
data:
  myvalue: "Hello World"
  drink: "coffeecoffeecoffeecoffeecoffee"
  food: "PIZZA"
```

## Using the `default` function

One function frequently used in templates is the `default` function: `default
DEFAULT_VALUE GIVEN_VALUE`. This function allows you to specify a default value
inside of the template, in case the value is omitted. Let's use it to modify the
drink example above:

```yaml
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

If we run this as normal, we'll get our `coffee`:

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: virtuous-mink-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

Now, we will remove the favorite drink setting from `values.yaml`:

```yaml
favorite:
  #drink: coffee
  food: pizza
```

Now re-running `helm install --dry-run --debug fair-worm ./mychart` will produce
this YAML:

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fair-worm-configmap
data:
  myvalue: "Hello World"
  drink: "tea"
  food: "PIZZA"
```

In an actual chart, all static default values should live in the `values.yaml`,
and should not be repeated using the `default` command (otherwise they would be
redundant). However, the `default` command is perfect for computed values, which
can not be declared inside `values.yaml`. For example:

```yaml
drink: {{ .Values.favorite.drink | default (printf "%s-tea" (include "fullname" .)) }}
```

In some places, an `if` conditional guard may be better suited than `default`.
We'll see those in the next section.

Template functions and pipelines are a powerful way to transform information and
then insert it into your YAML. But sometimes it's necessary to add some template
logic that is a little more sophisticated than just inserting a string. In the
next section we will look at the control structures provided by the template
language.

## Using the `lookup` function

The `lookup` function can be used to _look up_ resources in a running cluster.
The synopsis of the lookup function is `lookup apiVersion, kind, namespace, name
-> resource or resource list`.

| parameter  | type   |
|------------|--------|
| apiVersion | string |
| kind       | string |
| namespace  | string |
| name       | string |

Both `name` and `namespace` are optional and can be passed as an empty string
(`""`).

The following combination of parameters are possible:

| Behavior                               | Lookup function                            |
|----------------------------------------|--------------------------------------------|
| `kubectl get pod mypod -n mynamespace` | `lookup "v1" "Pod" "mynamespace" "mypod"`  |
| `kubectl get pods -n mynamespace`      | `lookup "v1" "Pod" "mynamespace" ""`       |
| `kubectl get pods --all-namespaces`    | `lookup "v1" "Pod" "" ""`                  |
| `kubectl get namespace mynamespace`    | `lookup "v1" "Namespace" "" "mynamespace"` |
| `kubectl get namespaces`               | `lookup "v1" "Namespace" "" ""`            |

When `lookup` returns an object, it will return a dictionary. This dictionary
can be further navigated to extract specific values.

The following example will return the annotations present for the `mynamespace`
object:

```go
(lookup "v1" "Namespace" "" "mynamespace").metadata.annotations
```

When `lookup` returns a list of objects, it is possible to access the object
list via the `items` field:

```go
{{ range $index, $service := (lookup "v1" "Service" "mynamespace" "").items }}
    {{/* do something with each service */}}
{{ end }}
```

When no object is found, an empty value is returned. This can be used to check
for the existence of an object.

The `lookup` function uses Helm's existing Kubernetes connection configuration
to query Kubernetes. If any error is returned when interacting with calling the
API server (for example due to lack of permission to access a resource), helm's
template processing will fail.

Keep in mind that Helm is not supposed to contact the Kubernetes API Server
during a `helm template` or a `helm install|update|delete|rollback --dry-run`,
so the `lookup` function will return an empty list (i.e. dict) in such a case.

## Operators are functions

For templates, the operators (`eq`, `ne`, `lt`, `gt`, `and`, `or` and so on) are
all implemented as functions. In pipelines, operations can be grouped with
parentheses (`(`, and `)`).

Now we can turn from functions and pipelines to flow control with conditions,
loops, and scope modifiers.
