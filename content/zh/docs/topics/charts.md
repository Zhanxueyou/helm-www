---
title: "Charts"
description: "阐述chart格式，并提供使用Helm构建chart的基本指导。"
weight: 1
---

Helm使用的包格式称为 _charts_。 chart就是一个描述Kubernetes相关资源的文件集合。单个chart可以用来部署一些简单的，类似于memcache pod，或者某些复杂的HTTP服务器以及web全栈应用、数据库、缓存等等。

Chart是作为特定目录布局的文件被创建的。它们可以打包到要部署的版本存档中。

如果你想下载和查看一个发布的chart，但不安装它，你可以用这个命令： `helm pull chartrepo/chartname`。

该文档解释说明了chart格式，并提供了用Helm构建chart的基本指导。

## Chart 文件结构

chart是一个组织在文件目录中的集合。目录名称就是chart名称（没有版本信息）。因而描述WordPress的chart可以存储在`wordpress/`目录中。

在这个目录中，Helm 期望可以匹配以下结构：

```text
wordpress/
  Chart.yaml          # 包含了chart信息的YAML文件
  LICENSE             # 可选: 包含chart许可证的纯文本文件
  README.md           # 可选: 可读的README文件
  values.yaml         # chart 默认的配置值
  values.schema.json  # 可选: 一个使用JSON结构的values.yaml文件
  charts/             # 包含chart依赖的其他chart
  crds/               # 自定义资源的定义
  templates/          # 模板目录， 当和values 结合时，可生成有效的Kubernetes manifest文件
  templates/NOTES.txt # 可选: 包含简要使用说明的纯文本文件
```

Helm保留使用 `charts/`，`crds/`， `templates/`目录，以及列举出的文件名。其他文件保持原样。

## Chart.yaml 文件

`Chart.yaml`文件是chart必需的。包含了以下字段：

```yaml
apiVersion: chart API 版本 （必需）
name: chart名称 （必需）
version: 语义化2 版本（必需）
kubeVersion: 兼容Kubernetes版本的语义化版本（可选）
description: 一句话对这个项目的描述（可选）
type: chart类型 （可选）
keywords:
  - 关于项目的一组关键字（可选）
home: 项目home页面的URL （可选）
sources:
  - 项目源码的URL列表（可选）
dependencies: # chart 必要条件列表 （可选）
  - name: chart名称 (nginx)
    version: chart版本 ("1.2.3")
    repository: 仓库URL ("https://example.com/charts") 或别名 ("@repo-name")
    condition: （可选） 解析为布尔值的yaml路径，用于启用/禁用chart (e.g. subchart1.enabled )
    tags: # （可选）
      - 用于一次启用/禁用 一组chart的tag
    enabled: （可选） 决定是否加载chart的布尔值
    import-values: # （可选）
      - ImportValue 保存源值到导入父键的映射。每项可以是字符串或者一对子/父列表项
    alias: （可选） chart中使用的别名。当你要多次添加相同的chart时会很有用
maintainers: # （可选）
  - name: 维护者名字 （每个维护者都需要）
    email: 维护者邮箱 （每个维护者可选）
    url: 维护者URL （每个维护者可选）
icon: 用做icon的SVG或PNG图片URL （可选）
appVersion: 包含的应用版本（可选）。不需要是语义化的
deprecated: 不被推荐的chart （可选，布尔值）
annotations:
  example: 按名称输入的批注列表 （可选）.
```

其他字段将被忽略。

### Chart和版本控制

每个chart都必须有个版本号。版本必须遵循 [语义化版本 2](https://semver.org/spec/v2.0.0.html) 标准。
不像经典Helm， Helm v2以及后续版本会使用版本号作为发布标记。仓库中的包通过名称加版本号标识。

比如 `nginx` chart的版本字段`version: 1.2.3`按照名称被设置为：

```text
nginx-1.2.3.tgz
```

更多复杂的语义化版本2 都是支持的，比如 `version: 1.2.3-alpha.1+ef365`。 但系统明确禁止非语义化版本名称。

**注意：** 鉴于经典Helm和部署管理器在使用chart时都非常倾向于GitHub，Helm v2 和后续版本不再依赖或需要GitHub甚至是Git。
因此，它完全不使用Git SHA进行版本控制。

`Chart.yaml`文件中的`version`字段被很多Helm工具使用，包括CLI。当生成一个包时，
`helm package`命令可以用`Chart.yaml`文件中找到的版本号作为包名中的token。
系统假设chart包名中的版本号可以与`Chart.yaml`文件中的版本号匹配。如果不满足这一假设会导致错误。

### `apiVersion` 字段

对于至少需要Helm 3的chart，`apiVersion` 字段应该是 `v2`。Chart支持之前`apiVersion` 设置为 `v1` 的Helm 版本，
并且在Helm 3中仍然可安装。

`v1` 到 `v2`的改变：

- `dependencies`字段定义了chart的依赖，针对于`v1` 版本的chart被放置在分隔开的`requirements.yaml` 文件中
（查看 [Chart 依赖](#Chart-依赖)).
- `type`字段, 用于识别应用和库类型的chart（查看 [Chart 类型](#chart-类型)).

### `appVersion` 字段

注意这个 `appVersion` 字段与 `version` 字段无关。它是指定应用程序版本的一种方式。
比如`drupal` chart 可以是 `appVersion: 8.2.1`， 表示包含在chart中（默认）的Drupal 版本是 `8.2.1`。
这个字段是信息字段，对chart版本的计算没有影响。

### `kubeVersion` 字段

可选的 `kubeVersion` 字段可以在支持的Kubernetes版本上定义语义化版本约束，Helm 在安装chart时会验证这个版本约束，
并在集群运行不支持的Kubernetes版本时显示失败。

版本约束可以包括空格分隔和比较运算符，比如：
```
>= 1.13.0 < 1.15.0
```
或者它们可以用或操作符 `||` 连接，比如：
```
>= 1.13.0 < 1.14.0 || >= 1.14.1 < 1.15.0
```
这个例子中排除了 `1.14.0` 版本，如果确定某些版本中的错误导致chart无法正常运行，这一点就很有意义。

除了版本约束外，使用运算符 `=` `!=` `>` `<` `>=` `<=` 支持一下速记符号：

 * 闭合间隔的连字符范围， `1.1 - 2.3.4` 等价于 `>= 1.1 <= 2.3.4`
 * 通配符 `x`， `X` 和 `*`， `1.2.x` 等价于 `>= 1.2.0 < 1.3.0`
 * 波浪符号~范围 （允许改变补丁版本）， `~1.2.3` 等价于 `>= 1.2.3 < 1.3.0`
 * 插入符号^范围 （允许改变次版本）， `^1.2.3` 等价于 `>= 1.2.3 < 2.0.0`

支持的语义化版本约束的细节说明请查看 [Masterminds/semver](https://github.com/Masterminds/semver)

### 已弃用的Chart

在Chart仓库管理chart时，有时需要废弃一个chart。 `Chart.yaml` 中可选的`deprecated`字段可以用来标记已弃用的chart。
如果**latest**版本被标记为已弃用，则所有的chart都会被认为是已弃用的。以后可以通过发布未标记为已弃用的新版本来重新使用chart名称。[kubernetes/charts](https://github.com/helm/charts) 项目遵循的弃用图标的工作流是：

1. 升级chart的 `Chart.yaml`  文件，将这个chart标记为已弃用， 并更改版本
2. 在chart仓库中发布新版的chart 
3. 从源仓库中移除这个chart （比如用 git）

### Chart 类型

`type`字段定义了chart的类型。有两种类型： `application` 和 `library`。 应用是默认类型，是可以完全操作的标准chart。
 [库类型 chart](http://helm.sh/docs/topics/library_charts.md) 提供针对chart构建的实用程序和功能。
 库类型chart与应用类型chart不同，因为它不能安装，通常不包含任何资源对象。

**注意：** 应用类型chart 可以作为库类型chart使用。可以通过将类型设置为 `library`来实现。
然后这个库就被渲染成了一个库类型chart，所有的实用程序和功能都可以使用。所有的资源对象不会被渲染。

## Chart 许可证，自述和注释

Chart也可以包含描述安装，配置和使用文件，以及chart许可证。

许可证（LICENSE）是一个包含了chart [license](https://en.wikipedia.org/wiki/Software_license) 的纯文本文件。
chart可以包含一个许可证，因为在模板里不只是配置，还可能有编码逻辑。如果需要，还可以为chart安装的应用程序提供单独的许可证。

chart的自述文件README文件应该使用Markdown格式(README.md)，一般应包含：

- chart提供的应用或服务的描述
- 运行chart的先决条件或要求
- `values.yaml`的可选项和默认值的描述
- 与chart的安装或配置相关的其他任何信息

`README.md` 文件会包含hub和用户接口显示的chart的详细信息。

chart也会包含一个简短的纯文本 `templates/NOTES.txt` 文件，这会在安装后及查看版本状态时打印出来。 这个文件会作为一个 [模板](#模板和Values)来评估，并用来显示使用说明，后续步骤，或者其他chart版本的相关信息。 比如，可以提供连接数据库的说明，web UI的访问。由于此文件是在运行`helm install`或`helm status`时打印到STDOUT的，因此建议保持内容简短，并指向自述文件以获取更多详细信息。 

## Chart 依赖

Helm 中，chart可能会依赖其他任意个cahrt。 这些依赖可以使用`Chart.yaml`文件中的`dependencies` 字段动态链接，或者被带入到 `charts/` 目录并手动配置。

### 使用 `dependencies` 字段管理依赖

当前chart依赖的其他chart会在`dependencies`字段定义为一个列表。

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: https://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: https://another.example.com/charts
```

- `name`字段是你需要的chart的名称
- `version`字段是你需要的chart的版本
- `repository`字段是chart仓库的完整URL。注意你必须使用`helm repo add`在本地添加仓库
- 你可以使用仓库的名称代替URL

```console
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com
```

```yaml
dependencies:
  - name: awesomeness
    version: 1.0.0
    repository: "@fantastic-charts"
```

&emsp;&emsp;一旦你定义好了依赖，运行 `helm dependency update` 就会使用你的依赖文件下载所有你指定的chart到你的`charts/`目录。

```console
$ helm dep up foochart
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "example" chart repository
...Successfully got an update from the "another" chart repository
Update Complete. Happy Helming!
Saving 2 charts
Downloading apache from repo https://example.com/charts
Downloading mysql from repo https://another.example.com/charts
```

当 `helm dependency update` 拉取chart时，会在`charts/`目录中形成一个chart包。因此对于上面的示例，会在chart目录中期望看到以下文件：

```text
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

#### 依赖的别名字段

除了上面的其他字段之外，每个需求项可以包含一个可选的字段 `alias`。
为依赖chart添加一个别名，会使用别名作为新依赖chart的名称。
需要使用其他名称访问chart时可以使用`alias`。

```yaml
# parentchart/Chart.yaml

dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-1
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-2
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
```

上述例子中，我们会获得`parentchart`所有的3个依赖项：

```text
subchart
new-subchart-1
new-subchart-2
```

手动完成的方式是将同一个chart用不同的名称复制/粘贴多次到`charts/`目录中。

#### 依赖中的tag和条件字段

除了上面的其他字段外，每个需求项可以包含可选字段 `tags` 和 `condition`。

所有的chart会默认加载。如果存在 `tags` 或者 `condition` 字段，它们将被评估并用于控制它们应用的chart的加载。

Condition - 条件字段field 包含一个或多个YAML路径（用逗号分隔）。 
如果这个路径在上层values中已存在并解析为布尔值，chart会基于布尔值启用或禁用chart。 
只会使用列表中找到的第一个有效路径，如果路径为未找到则条件无效。

Tags - tag字段是与chart关联的YAML格式的标签列表。在顶层value中，通过指定tag和布尔值，可以启用或禁用所有的带tag的chart。

```yaml
# parentchart/Chart.yaml

dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart1.enabled, global.subchart1.enabled
    tags:
      - front-end
      - subchart1
  - name: subchart2
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart2.enabled,global.subchart2.enabled
    tags:
      - back-end
      - subchart2
```

```yaml
# parentchart/values.yaml

subchart1:
  enabled: true
tags:
  front-end: false
  back-end: true
```

在上面的例子中，所有带 `front-end`tag的chart都会被禁用，但只要上层的value中
`subchart1.enabled` 路径被设置为 'true'，该条件会覆盖 `front-end`标签且 `subchart1` 会被启用。

一旦 `subchart2`使用了`back-end`标签并被设置为了 `true`，`subchart2`就会被启用。
也要注意尽管`subchart2` 指定了一个条件字段， 但是上层value没有相应的路径和value，因此这个条件不会生效。

##### 使用带有标签和条件的CLI

`--set` 参数一如既往可以用来设置标签和条件值。

```console
helm install --set tags.front-end=true --set subchart2.enabled=false
```

##### 标签和条件的解析

- **条件 （当设置在value中时）总是会覆盖标签** 第一个chart条件路径存在时会忽略后面的路径。
- 标签被定义为 '如果任意的chart标签是true，chart就可以启用'。
- 标签和条件值必须被设置在顶层value中。
- value中的`tags:`键必须是顶层键。 全局和嵌套的`tags:`表现在不支持了。

#### 通过依赖导入子Value

在某些情况下，允许子chart的值作为公共默认传递到父chart中是值得的。使用
`exports`格式的额外好处是它可是将来的工具可以自检用户可设置的值。

被导入的包含值的key可以在父chart的 `dependencies` 中的 `import-values` 字段以YAML列表形式指定。
列表中的每一项是从子chart中`exports`字段导入的key。

导入`exports` key中未包含的值，使用[子-父](#使用子-父格式)格式。两种格式的示例如下所述。

##### 使用导出格式

如果子chart的`values.yaml`文件中在根节点包含了`exports`字段，它的内容可以通过指定的可以被直接导入到父chart的value中，
如下所示：

```yaml
# parent's Chart.yaml file

dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    import-values:
      - data
```

```yaml
# child's values.yaml file

exports:
  data:
    myint: 99
```

只要我们再导入列表中指定了键`data`，Helm就会在子chart的`exports`字段查找`data`键并导入它的内容。

最终的父级value会包含我们的导出字段：

```yaml
# parent's values

myint: 99
```

注意父级键 `data` 没有包含在父级最终的value中，如果想指定这个父级键，要使用'子-父' 格式。

##### 使用子-父格式

要访问子chart中未包含的 `exports` 键的值，你需要指定要导入的值的源键(`child`)和父chart(`parent`)中值的目标路径。

下面示例中的`import-values` 指示Helm去拿到能再`child:`路径中找到的任何值，并拷贝到`parent:`的指定路径。

```yaml
# parent's Chart.yaml file

dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    ...
    import-values:
      - child: default.data
        parent: myimports
```

上面的例子中，在subchart1里面找到的`default.data`的值会被导入到父chart的`myimports`键中，细节如下：

```yaml
# parent's values.yaml file

myimports:
  myint: 0
  mybool: false
  mystring: "helm rocks!"
```

```yaml
# subchart1's values.yaml file

default:
  data:
    myint: 999
    mybool: true
```

父chart的结果值将会是这样：

```yaml
# parent's final values

myimports:
  myint: 999
  mybool: true
  mystring: "helm rocks!"
```

父chart中的最终值包含了从 subchart1中导入的`myint`和`mybool`字段。

### 通过`charts/`目录手动管理依赖

If more control over dependencies is desired, these dependencies can be
expressed explicitly by copying the dependency charts into the `charts/`
directory.

A dependency can be either a chart archive (`foo-1.2.3.tgz`) or an unpacked
chart directory. But its name cannot start with `_` or `.`. Such files are
ignored by the chart loader.

For example, if the WordPress chart depends on the Apache chart, the Apache
chart (of the correct version) is supplied in the WordPress chart's `charts/`
directory:

```yaml
wordpress:
  Chart.yaml
  # ...
  charts/
    apache/
      Chart.yaml
      # ...
    mysql/
      Chart.yaml
      # ...
```

The example above shows how the WordPress chart expresses its dependency on
Apache and MySQL by including those charts inside of its `charts/` directory.

**TIP:** _To drop a dependency into your `charts/` directory, use the `helm
pull` command_

### 使用依赖的操作部分

The above sections explain how to specify chart dependencies, but how does this
affect chart installation using `helm install` and `helm upgrade`?

Suppose that a chart named "A" creates the following Kubernetes objects

- namespace "A-Namespace"
- statefulset "A-StatefulSet"
- service "A-Service"

Furthermore, A is dependent on chart B that creates objects

- namespace "B-Namespace"
- replicaset "B-ReplicaSet"
- service "B-Service"

After installation/upgrade of chart A a single Helm release is created/modified.
The release will create/update all of the above Kubernetes objects in the
following order:

- A-Namespace
- B-Namespace
- A-Service
- B-Service
- B-ReplicaSet
- A-StatefulSet

This is because when Helm installs/upgrades charts, the Kubernetes objects from
the charts and all its dependencies are

- aggregrated into a single set; then
- sorted by type followed by name; and then
- created/updated in that order.

Hence a single release is created with all the objects for the chart and its
dependencies.

The install order of Kubernetes types is given by the enumeration InstallOrder
in kind_sorter.go (see [the Helm source
file](https://github.com/helm/helm/blob/484d43913f97292648c867b56768775a55e4bba6/pkg/releaseutil/kind_sorter.go)).

## 模板和Values

Helm Chart templates are written in the [Go template
language](https://golang.org/pkg/text/template/), with the addition of 50 or so
add-on template functions [from the Sprig
library](https://github.com/Masterminds/sprig) and a few other [specialized
functions]({{< ref "/docs/howto/charts_tips_and_tricks.md" >}}).

All template files are stored in a chart's `templates/` folder. When Helm
renders the charts, it will pass every file in that directory through the
template engine.

Values for the templates are supplied two ways:

- Chart developers may supply a file called `values.yaml` inside of a chart.
  This file can contain default values.
- Chart users may supply a YAML file that contains values. This can be provided
  on the command line with `helm install`.

When a user supplies custom values, these values will override the values in the
chart's `values.yaml` file.

### 模板文件

模板文件遵守书写Go模板的标准惯例（查看[文本/模板 Go 包文档documentation](https://golang.org/pkg/text/template/)了解跟多）。
模板文件的例子看起来像这样：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{ .Values.imageRegistry }}/postgres:{{ .Values.dockerTag }}
          imagePullPolicy: {{ .Values.pullPolicy }}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{ default "minio" .Values.storage }}
```

The above example, based loosely on
[https://github.com/deis/charts](https://github.com/deis/charts), is a template
for a Kubernetes replication controller. It can use the following four template
values (usually defined in a `values.yaml` file):

- `imageRegistry`: The source registry for the Docker image.
- `dockerTag`: The tag for the docker image.
- `pullPolicy`: The Kubernetes pull policy.
- `storage`: The storage backend, whose default is set to `"minio"`

All of these values are defined by the template author. Helm does not require or
dictate parameters.

To see many working charts, check out the [Kubernetes Charts
project](https://github.com/helm/charts)

### Predefined Values

Values that are supplied via a `values.yaml` file (or via the `--set` flag) are
accessible from the `.Values` object in a template. But there are other
pre-defined pieces of data you can access in your templates.

The following values are pre-defined, are available to every template, and
cannot be overridden. As with all values, the names are _case sensitive_.

- `Release.Name`: The name of the release (not the chart)
- `Release.Namespace`: The namespace the chart was released to.
- `Release.Service`: The service that conducted the release.
- `Release.IsUpgrade`: This is set to true if the current operation is an
  upgrade or rollback.
- `Release.IsInstall`: This is set to true if the current operation is an
  install.
- `Chart`: The contents of the `Chart.yaml`. Thus, the chart version is
  obtainable as `Chart.Version` and the maintainers are in `Chart.Maintainers`.
- `Files`: A map-like object containing all non-special files in the chart. This
  will not give you access to templates, but will give you access to additional
  files that are present (unless they are excluded using `.helmignore`). Files
  can be accessed using `{{ index .Files "file.name" }}` or using the
  `{{.Files.Get name }}` function. You can also access the contents of the file
  as `[]byte` using `{{ .Files.GetBytes }}`
- `Capabilities`: A map-like object that contains information about the versions
  of Kubernetes (`{{ .Capabilities.KubeVersion }}` and the supported Kubernetes
  API versions (`{{ .Capabilities.APIVersions.Has "batch/v1" }}`)

**NOTE:** Any unknown `Chart.yaml` fields will be dropped. They will not be
accessible inside of the `Chart` object. Thus, `Chart.yaml` cannot be used to
pass arbitrarily structured data into the template. The values file can be used
for that, though.

### Values files

Considering the template in the previous section, a `values.yaml` file that
supplies the necessary values would look like this:

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

A values file is formatted in YAML. A chart may include a default `values.yaml`
file. The Helm install command allows a user to override values by supplying
additional YAML values:

```console
$ helm install --generate-name --values=myvals.yaml wordpress
```

When values are passed in this way, they will be merged into the default values
file. For example, consider a `myvals.yaml` file that looks like this:

```yaml
storage: "gcs"
```

When this is merged with the `values.yaml` in the chart, the resulting generated
content will be:

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "gcs"
```

Note that only the last field was overridden.

**NOTE:** The default values file included inside of a chart _must_ be named
`values.yaml`. But files specified on the command line can be named anything.

**NOTE:** If the `--set` flag is used on `helm install` or `helm upgrade`, those
values are simply converted to YAML on the client side.

**NOTE:** If any required entries in the values file exist, they can be declared
as required in the chart template by using the ['required' function]({{< ref
"/docs/howto/charts_tips_and_tricks.md" >}})

Any of these values are then accessible inside of templates using the `.Values`
object:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{ .Values.imageRegistry }}/postgres:{{ .Values.dockerTag }}
          imagePullPolicy: {{ .Values.pullPolicy }}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{ default "minio" .Values.storage }}
```

### Scope, Dependencies, and Values

Values files can declare values for the top-level chart, as well as for any of
the charts that are included in that chart's `charts/` directory. Or, to phrase
it differently, a values file can supply values to the chart as well as to any
of its dependencies. For example, the demonstration WordPress chart above has
both `mysql` and `apache` as dependencies. The values file could supply values
to all of these components:

```yaml
title: "My WordPress Site" # Sent to the WordPress template

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

Charts at a higher level have access to all of the variables defined beneath. So
the WordPress chart can access the MySQL password as `.Values.mysql.password`.
But lower level charts cannot access things in parent charts, so MySQL will not
be able to access the `title` property. Nor, for that matter, can it access
`apache.port`.

Values are namespaced, but namespaces are pruned. So for the WordPress chart, it
can access the MySQL password field as `.Values.mysql.password`. But for the
MySQL chart, the scope of the values has been reduced and the namespace prefix
removed, so it will see the password field simply as `.Values.password`.

#### Global Values

As of 2.0.0-Alpha.2, Helm supports special "global" value. Consider this
modified version of the previous example:

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

The above adds a `global` section with the value `app: MyWordPress`. This value
is available to _all_ charts as `.Values.global.app`.

For example, the `mysql` templates may access `app` as `{{
.Values.global.app}}`, and so can the `apache` chart. Effectively, the values
file above is regenerated like this:

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  global:
    app: MyWordPress
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  global:
    app: MyWordPress
  port: 8080 # Passed to Apache
```

This provides a way of sharing one top-level variable with all subcharts, which
is useful for things like setting `metadata` properties like labels.

If a subchart declares a global variable, that global will be passed _downward_
(to the subchart's subcharts), but not _upward_ to the parent chart. There is no
way for a subchart to influence the values of the parent chart.

Also, global variables of parent charts take precedence over the global
variables from subcharts.

### Schema Files

Sometimes, a chart maintainer might want to define a structure on their values.
This can be done by defining a schema in the `values.schema.json` file. A schema
is represented as a [JSON Schema](https://json-schema.org/). It might look
something like this:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "image": {
      "description": "Container Image",
      "properties": {
        "repo": {
          "type": "string"
        },
        "tag": {
          "type": "string"
        }
      },
      "type": "object"
    },
    "name": {
      "description": "Service name",
      "type": "string"
    },
    "port": {
      "description": "Port",
      "minimum": 0,
      "type": "integer"
    },
    "protocol": {
      "type": "string"
    }
  },
  "required": [
    "protocol",
    "port"
  ],
  "title": "Values",
  "type": "object"
}
```

This schema will be applied to the values to validate it. Validation occurs when
any of the following commands are invoked:

- `helm install`
- `helm upgrade`
- `helm lint`
- `helm template`

An example of a `values.yaml` file that meets the requirements of this schema
might look something like this:

```yaml
name: frontend
protocol: https
port: 443
```

Note that the schema is applied to the final `.Values` object, and not just to
the `values.yaml` file. This means that the following `yaml` file is valid,
given that the chart is installed with the appropriate `--set` option shown
below.

```yaml
name: frontend
protocol: https
```

```console
helm install --set port=443
```

Furthermore, the final `.Values` object is checked against *all* subchart
schemas. This means that restrictions on a subchart can't be circumvented by a
parent chart. This also works backwards - if a subchart has a requirement that
is not met in the subchart's `values.yaml` file, the parent chart *must* satisfy
those restrictions in order to be valid.

### References

When it comes to writing templates, values, and schema files, there are several
standard references that will help you out.

- [Go templates](https://godoc.org/text/template)
- [Extra template functions](https://godoc.org/github.com/Masterminds/sprig)
- [The YAML format](https://yaml.org/spec/)
- [JSON Schema](https://json-schema.org/)

## Custom Resource Definitions (CRDs)

Kubernetes provides a mechanism for declaring new types of Kubernetes objects.
Using CustomResourceDefinitions (CRDs), Kubernetes developers can declare custom
resource types.

In Helm 3, CRDs are treated as a special kind of object. They are installed
before the rest of the chart, and are subject to some limitations.

CRD YAML files should be placed in the `crds/` directory inside of a chart.
Multiple CRDs (separated by YAML start and end markers) may be placed in the
same file. Helm will attempt to load _all_ of the files in the CRD directory
into Kubernetes.

CRD files _cannot be templated_. They must be plain YAML documents.

When Helm installs a new chart, it will upload the CRDs, pause until the CRDs
are made available by the API server, and then start the template engine, render
the rest of the chart, and upload it to Kubernetes. Because of this ordering,
CRD information is available in the `.Capabilities` object in Helm templates,
and Helm templates may create new instances of objects that were declared in
CRDs.

For example, if your chart had a CRD for `CronTab` in the `crds/` directory, you
may create instances of the `CronTab` kind in the `templates/` directory:

```text
crontabs/
  Chart.yaml
  crds/
    crontab.yaml
  templates/
    mycrontab.yaml
```

The `crontab.yaml` file must contain the CRD with no template directives:

```yaml
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
```

Then the template `mycrontab.yaml` may create a new `CronTab` (using templates
as usual):

```yaml
apiVersion: stable.example.com
kind: CronTab
metadata:
  name: {{ .Values.name }}
spec:
   # ...
```

Helm will make sure that the `CronTab` kind has been installed and is available
from the Kubernetes API server before it proceeds installing the things in
`templates/`.

### Limitations on CRDs

Unlike most objects in Kubernetes, CRDs are installed globally. For that reason,
Helm takes a very cautious approach in managing CRDs. CRDs are subject to the
following limitations:

- CRDs are never reinstalled. If Helm determines that the CRDs in the `crds/`
  directory are already present (regardless of version), Helm will not attempt
  to install or upgrade.
- CRDs are never installed on upgrade or rollback. Helm will only create CRDs on
  installation operations.
- CRDs are never deleted. Deleting a CRD automatically deletes all of the CRD's
  contents across all namespaces in the cluster. Consequently, Helm will not
  delete CRDs.

Operators who want to upgrade or delete CRDs are encouraged to do this manually
and with great care.

## Using Helm to Manage Charts

The `helm` tool has several commands for working with charts.

It can create a new chart for you:

```console
$ helm create mychart
Created mychart/
```

Once you have edited a chart, `helm` can package it into a chart archive for
you:

```console
$ helm package mychart
Archived mychart-0.1.-.tgz
```

You can also use `helm` to help you find issues with your chart's formatting or
information:

```console
$ helm lint mychart
No issues found
```

## Chart Repositories

A _chart repository_ is an HTTP server that houses one or more packaged charts.
While `helm` can be used to manage local chart directories, when it comes to
sharing charts, the preferred mechanism is a chart repository.

Any HTTP server that can serve YAML files and tar files and can answer GET
requests can be used as a repository server. The Helm team has tested some
servers, including Google Cloud Storage with website mode enabled, and S3 with
website mode enabled.

A repository is characterized primarily by the presence of a special file called
`index.yaml` that has a list of all of the packages supplied by the repository,
together with metadata that allows retrieving and verifying those packages.

On the client side, repositories are managed with the `helm repo` commands.
However, Helm does not provide tools for uploading charts to remote repository
servers. This is because doing so would add substantial requirements to an
implementing server, and thus raise the barrier for setting up a repository.

## Chart Starter Packs

The `helm create` command takes an optional `--starter` option that lets you
specify a "starter chart".

Starters are just regular charts, but are located in
`$XDG_DATA_HOME/helm/starters`. As a chart developer, you may author charts that
are specifically designed to be used as starters. Such charts should be designed
with the following considerations in mind:

- The `Chart.yaml` will be overwritten by the generator.
- Users will expect to modify such a chart's contents, so documentation should
  indicate how users can do so.
- All occurrences of `<CHARTNAME>` will be replaced with the specified chart
  name so that starter charts can be used as templates.

Currently the only way to add a chart to `$XDG_DATA_HOME/helm/starters` is to
manually copy it there. In your chart's documentation, you may want to explain
that process.
