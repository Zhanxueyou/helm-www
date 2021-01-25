---
title: "Chart Hook"
description: "详述如何使用chart hook"
weight: 2
---

Helm 提供了一个 _hook_ 机制允许chart开发者在发布生命周期的某些点进行干预。比如你可以使用hook用于：

- 安装时在加载其他chart之前加载配置映射或密钥
- 安装新chart之前执行备份数据库的任务，然后在升级之后执行第二个任务用于存储数据。
- 在删除发布之前执行一个任务以便在删除服务之前退出滚动。

钩子的工作方式与常规模板类似，但因为Helm对其不同的使用方式，会有一些特殊的注释。这部分会讲述钩子的基本使用模式。

## 可用的钩子

定义了以下钩子：

| 注释值            | 描述                                                                                                  |
| ---------------- | ----------------------------------------------------------------------------------------------------- |
| `pre-install`    | 在模板渲染之后，Kubernetes资源创建之前执行                                                               |
| `post-install`   | 在所有资源加载到Kubernetes之后执行                                                                      |
| `pre-delete`     | 在Kubernetes删除之前，执行删除请求                                                                      |
| `post-delete`    | 在所有的版本资源删除之后执行删除请求                                                                     |
| `pre-upgrade`    | 在模板渲染之后，资源更新之前执行一个升级请求                                                              |
| `post-upgrade`   | 所有资源升级之后执行一个升级请求                                                                         |
| `pre-rollback`   | 在模板渲染之后，资源回滚之前，执行一个回滚请求                                                            |
| `post-rollback`  | 在所有资源被修改之后执行一个回滚请求                                                                     |
| `test`           | 调用Helm test子命令时执行 ([test文档](https://helm.sh/zh/docs/topics/chart_tests/))                     |

_注意`crd-install`钩子已被移除以支持Helm 3的`crds/`目录。_

## 钩子和发布声明周期

钩子允许你在发布生命周期的关键节点上有机会执行操作。比如，考虑`helm install`的生命周期。默认的，生命周期看起来是这样：

1. 用户执行`helm install foo`
2. Helm库调用安装API
3. 在一些验证之后，库会渲染`foo`模板
4. 库会加载结果资源到Kubernetes
5. 库会返回发布对象（和其他数据）给客户端
6. 客户端退出

Helm 为`install`周期定义了两个钩子：`pre-install`和`post-install`。如果`foo` chart的开发者两个钩子都执行，
周期会被修改为这样：

1. 用户返回 `helm install foo`
2. Helm库调用安装API
3. 在 `crds/`目录中的CRD会被安装
4. 在一些验证之后，库会渲染`foo`模板
5. The library prepares to execute the `pre-install` hooks (loading hook
   resources into Kubernetes)
6. The library sorts hooks by weight (assigning a weight of 0 by default),
   by resource kind and finally by name in ascending order.
7. The library then loads the hook with the lowest weight first (negative to
   positive)
8. The library waits until the hook is "Ready" (except for CRDs)
9. The library loads the resulting resources into Kubernetes. Note that if the
   `--wait` flag is set, the library will wait until all resources are in a
   ready state and will not run the `post-install` hook until they are ready.
10. The library executes the `post-install` hook (loading hook resources)
11. The library waits until the hook is "Ready"
12. The library returns the release object (and other data) to the client
13. The client exits

What does it mean to wait until a hook is ready? This depends on the resource
declared in the hook. If the resource is a `Job` or `Pod` kind, Helm will wait
until it successfully runs to completion. And if the hook fails, the release
will fail. This is a _blocking operation_, so the Helm client will pause while
the Job is run.

For all other kinds, as soon as Kubernetes marks the resource as loaded (added
or updated), the resource is considered "Ready". When many resources are
declared in a hook, the resources are executed serially. If they have hook
weights (see below), they are executed in weighted order. 
Starting from Helm 3.2.0 hook resources with same weight are installed in the same 
order as normal non-hook resources. Otherwise, ordering is
not guaranteed. (In Helm 2.3.0 and after, they are sorted alphabetically. That
behavior, though, is not considered binding and could change in the future.) It
is considered good practice to add a hook weight, and set it to `0` if weight is
not important.

### Hook resources are not managed with corresponding releases

The resources that a hook creates are currently not tracked or managed as part
of the release. Once Helm verifies that the hook has reached its ready state, it
will leave the hook resource alone. Garbage collection of hook resources when
the corresponding release is deleted may be added to Helm 3 in the future, so
any hook resources that must never be deleted should be annotated with
`helm.sh/resource-policy: keep`.

Practically speaking, this means that if you create resources in a hook, you
cannot rely upon `helm uninstall` to remove the resources. To destroy such
resources, you need to either [add a custom `helm.sh/hook-delete-policy`
annotation](#hook-deletion-policies) to the hook template file, or [set the time
to live (TTL) field of a Job
resource](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/).

## Writing a Hook

Hooks are just Kubernetes manifest files with special annotations in the
`metadata` section. Because they are template files, you can use all of the
normal template features, including reading `.Values`, `.Release`, and
`.Template`.

For example, this template, stored in `templates/post-install-job.yaml`,
declares a job to be run on `post-install`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}"
  labels:
    app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
    app.kubernetes.io/instance: {{ .Release.Name | quote }}
    app.kubernetes.io/version: {{ .Chart.AppVersion }}
    helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
  annotations:
    # This is what defines this resource as a hook. Without this line, the
    # job is considered part of the release.
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    metadata:
      name: "{{ .Release.Name }}"
      labels:
        app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
        app.kubernetes.io/instance: {{ .Release.Name | quote }}
        helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    spec:
      restartPolicy: Never
      containers:
      - name: post-install-job
        image: "alpine:3.3"
        command: ["/bin/sleep","{{ default "10" .Values.sleepyTime }}"]

```

What makes this template a hook is the annotation:

```yaml
annotations:
  "helm.sh/hook": post-install
```

One resource can implement multiple hooks:

```yaml
annotations:
  "helm.sh/hook": post-install,post-upgrade
```

Similarly, there is no limit to the number of different resources that may
implement a given hook. For example, one could declare both a secret and a
config map as a pre-install hook.

When subcharts declare hooks, those are also evaluated. There is no way for a
top-level chart to disable the hooks declared by subcharts.

It is possible to define a weight for a hook which will help build a
deterministic executing order. Weights are defined using the following
annotation:

```yaml
annotations:
  "helm.sh/hook-weight": "5"
```

Hook weights can be positive or negative numbers but must be represented as
strings. When Helm starts the execution cycle of hooks of a particular Kind it
will sort those hooks in ascending order.

### Hook deletion policies

It is possible to define policies that determine when to delete corresponding
hook resources. Hook deletion policies are defined using the following
annotation:

```yaml
annotations:
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

You can choose one or more defined annotation values:

| Annotation Value       | Description                                                          |
| ---------------------- | -------------------------------------------------------------------- |
| `before-hook-creation` | Delete the previous resource before a new hook is launched (default) |
| `hook-succeeded`       | Delete the resource after the hook is successfully executed          |
| `hook-failed`          | Delete the resource if the hook failed during execution              |

If no hook deletion policy annotation is specified, the `before-hook-creation`
behavior applies by default.
