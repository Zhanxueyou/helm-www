---
title: "Values 文件"
description: "--values参数操作说明"
weight: 4
---

在上一部分我们了解了Helm模板提供的内置对象。其中一个是`Values`。该对象提供了对传递到chart的值的访问方法，
其内容源包括了多个位置：

- chart中的`values.yaml`文件
- 如果是子chart，就是父chart中的`values.yaml`文件
- 使用`-f`参数(`helm install -f myvals.yaml ./mychart`)传递到 `helm install` 或
`helm upgrade`的values文件
- 使用`--set` (比如`helm install --set foo=bar ./mychart`)传递的单个参数

以上列表有明确顺序：默认使用`values.yaml`，可以被父chart的`values.yaml`覆盖，继而被用户提供values文件覆盖，
最后会被`--set`参数覆盖。

values文件是普通的YAML文件。现在编辑`mychart/values.yaml`然后编辑配置映射ConfigMap模板。

删除`values.yaml`中的默认内容，仅设置一个参数：

```yaml
favoriteDrink: coffee
```

现在可以在模板中使用它：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favoriteDrink }}
```

注意最后一行，`favoriteDrink`是`Values`的一个属性: `{{ .Values.favoriteDrink }}`。

看看是如何渲染的：

```console
$ helm install geared-marsupi ./mychart --dry-run --debug
install.go:158: [debug] Original chart version: ""
install.go:175: [debug] CHART PATH: /home/bagratte/src/playground/mychart

NAME: geared-marsupi
LAST DEPLOYED: Wed Feb 19 23:21:13 2020
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
favoriteDrink: coffee

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: good-puppy-configmap
data:
  myvalue: "Hello World"
  drink: coffee
```

由于默认的`values.yaml`文件中设置了`favoriteDrink`的值为`coffee`，则这个显示在了模板中。
可以在调用`helm install`时设置`--set`，很容易就能覆盖这个值。

```console
$ helm install solid-vulture ./mychart --dry-run --debug --set favoriteDrink=slurm
install.go:158: [debug] Original chart version: ""
install.go:175: [debug] CHART PATH: /home/bagratte/src/playground/mychart

NAME: solid-vulture
LAST DEPLOYED: Wed Feb 19 23:25:54 2020
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
favoriteDrink: slurm

COMPUTED VALUES:
favoriteDrink: slurm

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: good-puppy-configmap
data:
  myvalue: "Hello World"
  drink: slurm
```

Since `--set` has a higher precedence than the default `values.yaml` file, our
template generates `drink: slurm`.

Values files can contain more structured content, too. For example, we could
create a `favorite` section in our `values.yaml` file, and then add several keys
there:

```yaml
favorite:
  drink: coffee
  food: pizza
```

Now we would have to modify the template slightly:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food }}
```

While structuring data this way is possible, the recommendation is that you keep
your values trees shallow, favoring flatness. When we look at assigning values
to subcharts, we'll see how values are named using a tree structure.

## Deleting a default key

If you need to delete a key from the default values, you may override the value
of the key to be `null`, in which case Helm will remove the key from the
overridden values merge.

For example, the stable Drupal chart allows configuring the liveness probe, in
case you configure a custom image. Here are the default values:

```yaml
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  initialDelaySeconds: 120
```

If you try to override the livenessProbe handler to `exec` instead of `httpGet`
using `--set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt]`, Helm will
coalesce the default and overridden keys together, resulting in the following
YAML:

```yaml
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  exec:
    command:
    - cat
    - docroot/CHANGELOG.txt
  initialDelaySeconds: 120
```

However, Kubernetes would then fail because you can not declare more than one
livenessProbe handler. To overcome this, you may instruct Helm to delete the
`livenessProbe.httpGet` by setting it to null:

```sh
helm install stable/drupal --set image=my-registry/drupal:0.1.0 --set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt] --set livenessProbe.httpGet=null
```

At this point, we've seen several built-in objects, and used them to inject
information into a template. Now we will take a look at another aspect of the
template engine: functions and pipelines.
