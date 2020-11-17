---
title: "使用Helm"
description: "阐述Helm的基础用法。"
weight: 3
---

该指南描述了使用Helm在Kubernetes集群中管理包的基本方法。
此刻假定您已经[安装](https://helm.sh/zh/docs/intro/install)了Helm客户端。

如果只是对一些快捷命令感兴趣，您可能希望从[快速开始指南](https://helm.sh/zh/docs/intro/quickstart)开始。
本章节将介绍Helm命令细节，以及如何使用他们。

## 三大概念

*Chart* 是一个Helm包，涵盖了需要在Kubernetes集群中运行应用，工具或者服务的资源定义。
把它想象成Kubernetes对应的Homebrew公式，Apt dpkg，或者是Yum RPM文件。

*仓库* 是归集和分享chart的地方。类似于Perl的 [CPAN 归档](https://www.cpan.org)或者[Fedora
包数据库](https://fedorahosted.org/pkgdb2/)，只针对于Kubernetes包。

*发布* 是在Kubernetes集群中运行的chart实例。一个chart经常在同一个集群中被重复安装。每次安装都会生成新的
_发布_。比如MySQL，如果想让两个数据库运行在集群中，可以将chart安装两次。每一个都会有自己的 _发布版本_，并有自己的 _发布名称_。

有了这些概念之后，就可以将Helm解释为：

Helm在Kubernetes中安装的每一个 _charts_，都会创建一个新的 _发布_，想查找新chart，可以在Helm chart _仓库_ 搜索。

## 'helm search'：查找chart

Helm有强大的搜索命令。可以搜索两类不同资源：

- `helm search hub` 搜索[Artifact Hub](https://artifacthub.io)，改仓库列出了来自不同仓库的大量chart。
- `helm search repo` 搜索已经(用`helm repo add`)加入到本地helm客户端的仓库。该命名只搜索本地数据，不需要连接网络。

可以运行`helm search hub`搜索公共可用的chart：

```console
$ helm search hub wordpress
URL                                                 CHART VERSION APP VERSION DESCRIPTION
https://hub.helm.sh/charts/bitnami/wordpress        7.6.7         5.2.4       Web publishing platform for building blogs and ...
https://hub.helm.sh/charts/presslabs/wordpress-...  v0.6.3        v0.6.3      Presslabs WordPress Operator Helm Chart
https://hub.helm.sh/charts/presslabs/wordpress-...  v0.7.1        v0.7.1      A Helm chart for deploying a WordPress site on ...
```

上述搜索从Artifact Hub中搜到了所有的`wordpress`的charts。

不过滤的话，`helm search hub`会展示所有可用chart。

使用`helm search repo`可以找到所有已经添加到仓库的chart名称：

```console
$ helm repo add brigade https://brigadecore.github.io/charts
"brigade" has been added to your repositories
$ helm search repo brigade
NAME                          CHART VERSION APP VERSION DESCRIPTION
brigade/brigade               1.3.2         v1.2.1      Brigade provides event-driven scripting of Kube...
brigade/brigade-github-app    0.4.1         v0.2.1      The Brigade GitHub App, an advanced gateway for...
brigade/brigade-github-oauth  0.2.0         v0.20.0     The legacy OAuth GitHub Gateway for Brigade
brigade/brigade-k8s-gateway   0.1.0                     A Helm chart for Kubernetes
brigade/brigade-project       1.0.0         v1.0.0      Create a Brigade project
brigade/kashti                0.4.0         v0.4.0      A Helm chart for Kubernetes
```

Helm搜索使用字符串模糊匹配，因此输入部分名称也可以：

```console
$ helm search repo kash
NAME            CHART VERSION APP VERSION DESCRIPTION
brigade/kashti  0.4.0         v0.4.0      A Helm chart for Kubernetes
```

搜索是查找可用包的有效方式。一旦您找到了想要安装的包，使用`helm install`安装即可。

## 'helm install'：安装一个包

安装一个新包，使用`helm install`命令。最简单的方式有两个参数：查找到发布名称和chart名称。

```console
$ helm install happy-panda stable/mariadb
WARNING: This chart is deprecated
NAME: happy-panda
LAST DEPLOYED: Fri May  8 17:46:49 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
This Helm chart is deprecated

...

Services:

  echo Master: happy-panda-mariadb.default.svc.cluster.local:3306
  echo Slave:  happy-panda-mariadb-slave.default.svc.cluster.local:3306

Administrator credentials:

  Username: root
  Password : $(kubectl get secret --namespace default happy-panda-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run happy-panda-mariadb-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mariadb:10.3.22-debian-10-r27 --namespace default --command -- bash

  2. To connect to master service (read/write):

      mysql -h happy-panda-mariadb.default.svc.cluster.local -uroot -p my_database

  3. To connect to slave service (read-only):

      mysql -h happy-panda-mariadb-slave.default.svc.cluster.local -uroot -p my_database

To upgrade this helm chart:

  1. Obtain the password as described on the 'Administrator credentials' section and set the 'rootUser.password' parameter as shown below:

      ROOT_PASSWORD=$(kubectl get secret --namespace default happy-panda-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)
      helm upgrade happy-panda stable/mariadb --set rootUser.password=$ROOT_PASSWORD

```

现在`mariadb` chart已经安装。注意安装chart会创建一个新的发布对象。上述发布的名称是：`happy-panda`。
（如果想让Helm为你生成一个名称，去掉发布名称并加上`--generate-name`）

安装过程中，`helm`客户端会打印资源创建，发布状态以及额外需要额外处理的配置步骤等有效信息。

Helm不会等所有资源都运行了才推出。许多chart需要的Docker镜像大小超过了600M，并且在集群安装可能会花很长时间。

为了跟踪发布状态，或者再看看配置信息，可以使用`helm status`：

```console
$ helm status happy-panda
NAME: happy-panda
LAST DEPLOYED: Fri May  8 17:46:49 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
This Helm chart is deprecated

...

Services:

  echo Master: happy-panda-mariadb.default.svc.cluster.local:3306
  echo Slave:  happy-panda-mariadb-slave.default.svc.cluster.local:3306

Administrator credentials:

  Username: root
  Password : $(kubectl get secret --namespace default happy-panda-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run happy-panda-mariadb-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mariadb:10.3.22-debian-10-r27 --namespace default --command -- bash

  2. To connect to master service (read/write):

      mysql -h happy-panda-mariadb.default.svc.cluster.local -uroot -p my_database

  3. To connect to slave service (read-only):

      mysql -h happy-panda-mariadb-slave.default.svc.cluster.local -uroot -p my_database

To upgrade this helm chart:

  1. Obtain the password as described on the 'Administrator credentials' section and set the 'rootUser.password' parameter as shown below:

      ROOT_PASSWORD=$(kubectl get secret --namespace default happy-panda-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)
      helm upgrade happy-panda stable/mariadb --set rootUser.password=$ROOT_PASSWORD
```

上述显示了发布的当前状态。

### 安装之前自定义chart

此处安装只会使用chartd默认配置。很多时候，您想使用自己的配置自定义chart。

使用`helm show values`查看chart的可配置项：

```console
$ helm show values stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/helm.sh/helm/mariadb-0.3.0.tgz
## Bitnami MariaDB image version
## ref: https://hub.docker.com/r/bitnami/mariadb/tags/
##
## Default: none
imageTag: 10.1.14-r3

## Specify a imagePullPolicy
## Default to 'Always' if imageTag is 'latest', else set to 'IfNotPresent'
## ref: https://kubernetes.io/docs/user-guide/images/#pre-pulling-images
##
# imagePullPolicy:

## Specify password for root user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#setting-the-root-password-on-first-run
##
# mariadbRootPassword:

## Create a database user
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-user-on-first-run
##
# mariadbUser:
# mariadbPassword:

## Create a database
## ref: https://github.com/bitnami/bitnami-docker-mariadb/blob/master/README.md#creating-a-database-on-first-run
##
# mariadbDatabase:
# ...
```

可以在YAML格式文件中覆盖这些配置，然后在安装时调用这个文件。

```console
$ echo '{mariadbUser: user0, mariadbDatabase: user0db}' > config.yaml
$ helm install -f config.yaml stable/mariadb --generate-name
```

上述命令会创建一个名为`user0`的MariaDB默认用户，并授予该用户最新创建的`user0db`库的访问权限，但其他配置会使用chart的默认配置。

安装时有两种方式传递配置数据：

- `--values` (或`-f`)：指定一个重写的YAML文件。可以指定多个，最右边的文件优先
- `--set`: 使用命令行指定覆盖内容

如果两个都使用，`--set`的值会以高优先级合并到`--values`中。用`--set`指定的覆盖内容会在配置映射中保存。
`--set`配置的值可以只用`helm get values <release-name>`在给定的发布中看到。`--set`指定的值会被`helm upgrade`运行时`--reset-values`指定的值清空。

#### `--set`的格式和限制

`--set`选项附带0个或多个名字/值 对。最简单的使用方式是：`--set name=value`。 相对应的YAML是：

```yaml
name: value
```

多行可以使用`,`分隔，`--set a=b,c=d`则变为：

```yaml
a: b
c: d
```

还支持更负载的表达式，如：`--set outer.inner=value`会翻译成这样：

```yaml
outer:
  inner: value
```

列表可以用中括号括起来表示，比如：`--set name={a, b, c}`翻译为：

```yaml
name:
  - a
  - b
  - c
```

从Helm 2.5.0开始，可以使用数组索引语法访问列表项。比如`--set servers[0].port=80`变成了：

```yaml
servers:
  - port: 80
```

这种方式可以设置多个值，`--set servers[0].port=80,servers[0].host=example`变成了：

```yaml
servers:
  - port: 80
    host: example
```

有时需要在`--set`行使用特殊符号。可以使用反斜杠转义字符，`--set name=value1\,value2`会变成：

```yaml
name: "value1,value2"
```

类似的，可以转义点序列，当chart使用`toYaml`方法解析注释、标签和节点选择器时会很有用。
`--set nodeSelector."kubernetes\.io/role"=master`则会变成：

```yaml
nodeSelector:
  kubernetes.io/role: master
```

深度嵌套是数据结构很难使用`--set`表达。建议chart设计者设计`values.yaml`文件格式时考虑`--set`用法(查看[Values
文件](https://helm.sh/docs/chart_template_guide/values_files/)了解更多)。

### 更多安装方法

`helm install`命令可以从以下这些源安装：

- chart仓库(如上所述)
- 本地chart包 (`helm install foo foo-0.1.1.tgz`)
- 解压的chart目录 (`helm install foo path/to/foo`)
- 完整URL(`helm install foo https://example.com/charts/foo-1.2.3.tgz`)

## 'helm upgrade' 和 'helm rollback'：升级版本和恢复失败

当chart新版本发布时，或者您想改变发布的配置，可以使用`helm upgrade`命令。

升级采用已有版本并根据您提供的信息进行升级。由于Kubernetes的chart会很大且很复杂，Helm会尝试执行最小增量升级。
这样只会升级自最新版发生改变的部分。

```console
$ helm upgrade -f panda.yaml happy-panda stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/helm.sh/helm/mariadb-0.3.0.tgz
happy-panda has been upgraded. Happy Helming!
Last Deployed: Wed Sep 28 12:47:54 2016
Namespace: default
Status: DEPLOYED
...
```

上面这个例子中，`happy-panda`发布使用了同样的chart升级，但用了一个新的YAML文件：

```yaml
mariadbUser: user1
```

我们可以使用`helm get values`查看新内容是否生效。

```console
$ helm get values happy-panda
mariadbUser: user1
```

`helm get`在查看集群内的发布时很有用。正如我们上面看到的，显示了`panda.yaml`的新值已经部署到了集群。

现在，如果有内容在发布中未按计划执行，使用`helm rollback [RELEASE] [REVISION]`能很容易回滚到上个版本。

```console
$ helm rollback happy-panda 1
```

上述回滚了happy-panda到第一个发布版本。 发布版本是增量修订。每次安装、升级或者回滚，修订号都会自增加1。
第一个版本号始终是1。我们可以使用`helm history [RELEASE]`查看某个版本的修订号。

## 安装/升级/回滚的有用项

There are several other helpful options you can specify for customizing the
behavior of Helm during an install/upgrade/rollback. Please note that this is
not a full list of cli flags. To see a description of all flags, just run `helm
<command> --help`.

- `--timeout`: A [Go duration](https://golang.org/pkg/time/#ParseDuration) value
  to wait for Kubernetes commands to complete. This defaults to `5m0s`.
- `--wait`: Waits until all Pods are in a ready state, PVCs are bound,
  Deployments have minimum (`Desired` minus `maxUnavailable`) Pods in ready
  state and Services have an IP address (and Ingress if a `LoadBalancer`) before
  marking the release as successful. It will wait for as long as the `--timeout`
  value. If timeout is reached, the release will be marked as `FAILED`. Note: In
  scenarios where Deployment has `replicas` set to 1 and `maxUnavailable` is not
  set to 0 as part of rolling update strategy, `--wait` will return as ready as
  it has satisfied the minimum Pod in ready condition.
- `--no-hooks`: This skips running hooks for the command
- `--recreate-pods` (only available for `upgrade` and `rollback`): This flag
  will cause all pods to be recreated (with the exception of pods belonging to
  deployments). (DEPRECATED in Helm 3)

## 'helm uninstall': Uninstalling a Release

When it is time to uninstall a release from the cluster, use the `helm
uninstall` command:

```console
$ helm uninstall happy-panda
```

This will remove the release from the cluster. You can see all of your currently
deployed releases with the `helm list` command:

```console
$ helm list
NAME            VERSION UPDATED                         STATUS          CHART
inky-cat        1       Wed Sep 28 12:59:46 2016        DEPLOYED        alpine-0.1.0
```

From the output above, we can see that the `happy-panda` release was
uninstalled.

In previous versions of Helm, when a release was deleted, a record of its
deletion would remain. In Helm 3, deletion removes the release record as well.
If you wish to keep a deletion release record, use `helm uninstall
--keep-history`. Using `helm list --uninstalled` will only show releases that
were uninstalled with the `--keep-history` flag.

The `helm list --all` flag will show you all release records that Helm has
retained, including records for failed or deleted items (if `--keep-history` was
specified):

```console
$  helm list --all
NAME            VERSION UPDATED                         STATUS          CHART
happy-panda     2       Wed Sep 28 12:47:54 2016        UNINSTALLED     mariadb-0.3.0
inky-cat        1       Wed Sep 28 12:59:46 2016        DEPLOYED        alpine-0.1.0
kindred-angelf  2       Tue Sep 27 16:16:10 2016        UNINSTALLED     alpine-0.1.0
```

Note that because releases are now deleted by default, it is no longer possible
to rollback an uninstalled resource.

## 'helm repo': Working with Repositories

Helm 3 no longer ships with a default chart repository. The `helm repo` command
group provides commands to add, list, and remove repositories.

You can see which repositories are configured using `helm repo list`:

```console
$ helm repo list
NAME            URL
stable          https://charts.helm.sh/stable
mumoshu         https://mumoshu.github.io/charts
```

And new repositories can be added with `helm repo add`:

```console
$ helm repo add dev https://example.com/dev-charts
```

Because chart repositories change frequently, at any point you can make sure
your Helm client is up to date by running `helm repo update`.

Repositories can be removed with `helm repo remove`.

## Creating Your Own Charts

The [Chart Development Guide]({{< ref "../topics/charts.md" >}}) explains how
to develop your own charts. But you can get started quickly by using the `helm
create` command:

```console
$ helm create deis-workflow
Creating deis-workflow
```

Now there is a chart in `./deis-workflow`. You can edit it and create your own
templates.

As you edit your chart, you can validate that it is well-formed by running `helm
lint`.

When it's time to package the chart up for distribution, you can run the `helm
package` command:

```console
$ helm package deis-workflow
deis-workflow-0.1.0.tgz
```

And that chart can now easily be installed by `helm install`:

```console
$ helm install deis-workflow ./deis-workflow-0.1.0.tgz
...
```

Charts that are packaged can be loaded into chart repositories. See the
documentation for your chart repository server to learn how to upload.

Note: The `stable` repository is managed on the [Kubernetes Charts GitHub
repository](https://github.com/helm/charts). That project accepts chart source
code, and (after audit) packages those for you.

## Conclusion

This chapter has covered the basic usage patterns of the `helm` client,
including searching, installation, upgrading, and uninstalling. It has also
covered useful utility commands like `helm status`, `helm get`, and `helm repo`.

For more information on these commands, take a look at Helm's built-in help:
`helm help`.

In the [next chapter](../howto/charts_tips_and_tricks/), we look at the process of developing charts.
