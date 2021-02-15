---
title: "Helm插件指南"
description: "介绍如何使用和创建插件来扩展Helm功能。"
weight: 12
---

Helm插件是一个可以通过`helm` CLI访问的工具，但不是Helm的内置代码。

已有插件可以搜索[GitHub](https://github.com/search?q=topic%3Ahelm-plugin&type=Repositories)。

该指南描述如何使用和创建插件。

## 概述

Helm插件是与Helm无缝集成的附加工具。插件提供一种扩展Helm核心特性集的方法，但不需要每个新的特性都用Go编写并加入核心工具中。

Helm插件有以下特性：

- 可以在不影响Helm核心工具的情况下添加和移除。
- 可以用任意编程语言编写。
- 与Helm集成，并展示在`helm help`和其他地方。

Helm插件存在与`$HELM_PLUGINS`。你可以找到该变量的当前值，包括不设置环境变量的默认值，使用`helm env`命令。

Helm插件模型部分基于Git的插件模型。为此，你有时可能听到`helm`已插件为基础被用作_porcelain_ 层。这是一种Helm提供用户体验和顶级处理逻辑的简写方式。
而插件执行所需操作的“细节工作”。

## 安装一个插件

插件使用 `$ helm plugin install <path|url>` 命令安装插件。你可以在本地文件系统上传一个路径或远程仓库url给插件。The `helm plugin install`
命令会克隆或拷贝给定路径的插件到 `$HELM_PLUGINS`。

```console
$ helm plugin install https://github.com/adamreese/helm-env
```

如果是插件tar包，仅需解压插件到`$HELM_PLUGINS`目录。也可以用tar包的url直接安装：
`helm plugin install https://domain/path/to/plugin.tar.gz`。

## 构建插件

在很多方面，插件类似于chart。每个插件有个顶级目录和一个`plugin.yaml`文件。

```console
$HELM_PLUGINS/
  |- keybase/
      |
      |- plugin.yaml
      |- keybase.sh

```

上述示例中，`keybase`插件包含在`keybase`目录中。有两个文件：`plugin.yaml`（必需）和一个可执行脚本，`keybase.sh`（可选）。

插件的核心是一个简单的YAML文件`plugin.yaml`。下面是一个插件YAML，用于添加对Keybase操作的支持：

```yaml
name: "last"
version: "0.1.0"
usage: "get the last release name"
description: "get the last release name"
ignoreFlags: false
command: "$HELM_BIN --host $TILLER_HOST list --short --max 1 --date -r"
platformCommand:
  - os: linux
    arch: i386
    command: "$HELM_BIN list --short --max 1 --date -r"
  - os: linux
    arch: amd64
    command: "$HELM_BIN list --short --max 1 --date -r"
  - os: windows
    arch: amd64
    command: "$HELM_BIN list --short --max 1 --date -r"
```

`name`是插件名称。当Helm执行此插件时使用此名称。（比如，`helm NAME`会调用此插件）。

上述示例中，_`name`应该匹配目录名称_，意味着`keybase`目录中应该包含 `name: keybase` 插件。

`name`的限制：

- `name` 无法复用现有的 `helm` 顶级命令。
- `name` 的字符必须限制为ASCII a-z， A-Z， 0-9， `_` 和 `-`。

`version` 是插件的语义化2的版本。 `usage` 和 `description` 用于生成命令的帮助文本。

`ignoreFlags` 开关告诉 Helm _不要_ 给插件传递的参数。因此如果一个插件使用 `helm myplugin --foo`调用且
`ignoreFlags: true`，那么`--foo`会被悄悄忽略。

最后，尤其重要的是 `platformCommand` 或 `command` 是插件调用时执行的命令。`platformCommand` 部分定义了命令在
系统/架构的特定变体。以下规则用于决定使用哪个命令：

- 如果`platformCommand`存在，会优先被搜索。
- 如果`os` 和 `arch` 匹配了当前平台，搜索会停止并使用命令。
- 如果`os`匹配且没有匹配 `arch` ，命令会被使用。
- 如果没有匹配`platformCommand`，会使用默认的`command`。
- 如果没有匹配 `platformCommand` 且不存在 `command`，Helm会报错退出。

环境变量会在插件执行前被插入。上述模式说明了表示插件所在位置的首选方法。

有一些使用插件命令的策略：

- 如果插件中包含可执行文件，可执行文件针对于 `platformCommand:`或`command:`命令，应该打包到插件目录中。
- `platformCommand:` 或者 `command:` 行会在执行之前展开任何环境变量。`$HELM_PLUGIN_DIR`会指向插件目录。
- 命令本身不是在shell中执行的。 所以不能一行一个shell脚本。
- Helm在环境变量中插入很多配置。查看环境变量获取可用信息。
- Helm对插件语言不做任何假设。你想写什么写什么。
- `-h` 和 `--help`命令负责实现特定的帮助文本。Helm会在`helm help` 和 `helm help myplugin`中使用`usage`
  和 `description`，但不处理`helm myplugin --help`。

## 下载插件

默认情况下，Helm能够使用HTTP/S拉取chart。从Helm 2.4.0开始，插件有一种能力从任意来源下载chart。

插件应该在`plugin.yaml`（顶层的）文件中声明这个特殊能力：

```yaml
downloaders:
- command: "bin/mydownloader"
  protocols:
  - "myprotocol"
  - "myprotocols"
```

如果这个插件已经安装，Helm可以通过调用`command`可以使用指定的协议方案与存储仓库进行交互。
特殊仓库的添加与常规仓库类似：`helm repo add favorite myprotocol://example.com/`。
特殊仓库的的规则也和常规仓库相同：为了发现和缓存chart的可用列表，Helm必须下载`index.yaml`文件。

已定义的命令可以通过以下结构调用： `command certFile keyFile caFile full-URL`。SSL证书有仓库定义，存储在
`$HELM_REPOSITORY_CONFIG`(即：`$HELM_CONFIG_HOME/repositories.yaml`)。
下载器插件将原始内容使用stdout输出并使用stderr报告错误。

下载器命令也支持子命令和参数，允许在`plugin.yaml`指定，比如`bin/mydownloader subcommand -d`。
如果你想在相同的可执行文件中执行主要的插件命令和下载器命令，这就变得很有用，但每个命令都有不同的子命令。

## 环境变量

当Helm执行插件时，会传递外部环境变量给插件，且会加入一些额外的环境变量。

像 `KUBECONFIG` 这样的变量，如果是在外部环境中设置的，则是为插件设置的。

要保证设置以下变量：

- `HELM_PLUGINS`: 插件目录路径。
- `HELM_PLUGIN_NAME`: `helm`调用的插件名称。
- `HELM_PLUGIN_DIR`: 包含插件的目录。
- `HELM_BIN`: The path to the `helm` command (as executed by the user).
- `HELM_DEBUG`: Indicates if the debug flag was set by helm.
- `HELM_REGISTRY_CONFIG`: The location for the registry configuration (if
  using). Note that the use of Helm with registries is an experimental feature.
- `HELM_REPOSITORY_CACHE`: The path to the repository cache files.
- `HELM_REPOSITORY_CONFIG`: The path to the repository configuration file.
- `HELM_NAMESPACE`: The namespace given to the `helm` command (generally using
  the `-n` flag).
- `HELM_KUBECONTEXT`: The name of the Kubernetes config context given to the
  `helm` command.

Additionally, if a Kubernetes configuration file was explicitly specified, it
will be set as the `KUBECONFIG` variable

## 参数解析说明

当执行插件时，Helm会解析自己的全局参数。这些参数都不会传递给插件。

- `--debug`: 如果指定， `$HELM_DEBUG` 设置为 `1`
- `--registry-config`: 链接到了 `$HELM_REGISTRY_CONFIG`
- `--repository-cache`: 链接到了 `$HELM_REPOSITORY_CACHE`
- `--repository-config`: 链接到了 `$HELM_REPOSITORY_CONFIG`
- `--namespace` and `-n`: 链接到了 `$HELM_NAMESPACE`
- `--kube-context`: 链接到了 `$HELM_KUBECONTEXT`
- `--kubeconfig`: 链接到了 `$KUBECONFIG`

Plugins _should_ display help text and then exit for `-h` and `--help`. In all
other cases, plugins may use flags as appropriate.

## 提供shell自动补全

As of Helm 3.2, a plugin can optionally provide support for shell
auto-completion as part of Helm's existing auto-completion mechanism.

### 静态自动补全

If a plugin provides its own flags and/or sub-commands, it can inform Helm of
them by having a `completion.yaml` file located in the plugin's root directory.
The `completion.yaml` file has the form:

```yaml
name: <pluginName>
flags:
- <flag 1>
- <flag 2>
validArgs:
- <arg value 1>
- <arg value 2>
commands:
  name: <commandName>
  flags:
  - <flag 1>
  - <flag 2>
  validArgs:
  - <arg value 1>
  - <arg value 2>
  commands:
     <and so on, recursively>
```

注意：

1. All sections are optional but should be provided if applicable.
1. Flags should not include the `-` or `--` prefix.
1. Both short and long flags can and should be specified. A short flag need not
   be associated with its corresponding long form, but both forms should be
   listed.
1. Flags need not be ordered in any way, but need to be listed at the correct
   point in the sub-command hierarchy of the file.
1. Helm's existing global flags are already handled by Helm's auto-completion
   mechanism, therefore plugins need not specify the following flags `--debug`,
   `--namespace` or `-n`, `--kube-context`, and `--kubeconfig`, or any other
   global flag.
1. The `validArgs` list provides a static list of possible completions for the
   first parameter following a sub-command.  It is not always possible to
   provide such a list in advance (see the [Dynamic
   Completion](#dynamic-completion) section below), in which case the
   `validArgs` section can be omitted.

The `completion.yaml` file is entirely optional.  If it is not provided, Helm
will simply not provide shell auto-completion for the plugin (unless [Dynamic
Completion](#dynamic-completion) is supported by the plugin).  Also, adding a
`completion.yaml` file is backwards-compatible and will not impact the behavior
of the plugin when using older helm versions.

As an example, for the [`fullstatus
plugin`](https://github.com/marckhouzam/helm-fullstatus) which has no
sub-commands but accepts the same flags as the `helm status` command, the
`completion.yaml` file is:

```yaml
name: fullstatus
flags:
- o
- output
- revision
```

A more intricate example for the [`2to3
plugin`](https://github.com/helm/helm-2to3), has a `completion.yaml` file of:

```yaml
name: 2to3
commands:
- name: cleanup
  flags:
  - config-cleanup
  - dry-run
  - l
  - label
  - release-cleanup
  - s
  - release-storage
  - tiller-cleanup
  - t
  - tiller-ns
  - tiller-out-cluster
- name: convert
  flags:
  - delete-v2-releases
  - dry-run
  - l
  - label
  - s
  - release-storage
  - release-versions-max
  - t
  - tiller-ns
  - tiller-out-cluster
- name: move
  commands:
  - name: config
    flags:
    - dry-run
```

### 动态补全

Also starting with Helm 3.2, plugins can provide their own dynamic shell
auto-completion. Dynamic shell auto-completion is the completion of parameter
values or flag values that cannot be defined in advance.  For example,
completion of the names of helm releases currently available on the cluster.

For the plugin to support dynamic auto-completion, it must provide an
**executable** file called `plugin.complete` in its root directory. When the
Helm completion script requires dynamic completions for the plugin, it will
execute the `plugin.complete` file, passing it the command-line that needs to be
completed.  The `plugin.complete` executable will need to have the logic to
determine what the proper completion choices are and output them to standard
output to be consumed by the Helm completion script.

The `plugin.complete` file is entirely optional.  If it is not provided, Helm
will simply not provide dynamic auto-completion for the plugin.  Also, adding a
`plugin.complete` file is backwards-compatible and will not impact the behavior
of the plugin when using older helm versions.

The output of the `plugin.complete` script should be a new-line separated list
such as:

```console
rel1
rel2
rel3
```

When `plugin.complete` is called, the plugin environment is set just like when
the plugin's main script is called. Therefore, the variables `$HELM_NAMESPACE`,
`$HELM_KUBECONTEXT`, and all other plugin variables will already be set, and
their corresponding global flags will be removed.

The `plugin.complete` file can be in any executable form; it can be a shell
script, a Go program, or any other type of program that Helm can execute. The
`plugin.complete` file ***must*** have executable permissions for the user. The
`plugin.complete` file ***must*** exit with a success code (value 0).

In some cases, dynamic completion will require to obtain information from the
Kubernetes cluster.  For example, the `helm fullstatus` plugin requires a
release name as input. In the `fullstatus` plugin, for its `plugin.complete`
script to provide completion for current release names, it can simply run `helm
list -q` and output the result.

If it is desired to use the same executable for plugin execution and for plugin
completion, the `plugin.complete` script can be made to call the main plugin
executable with some special parameter or flag; when the main plugin executable
detects the special parameter or flag, it will know to run the completion. In
our example, `plugin.complete` could be implemented like this:

```sh
#!/usr/bin/env sh

# "$@" is the entire command-line that requires completion.
# It is important to double-quote the "$@" variable to preserve a possibly empty last parameter.
$HELM_PLUGIN_DIR/status.sh --complete "$@"
```

The `fullstatus` plugin's real script (`status.sh`) must then look for the
`--complete` flag and if found, printout the proper completions.

### 提示和技巧

1. The shell will automatically filter out completion choices that don't match
   user input. A plugin can therefore return all relevant completions without
   removing the ones that don't match the user input.  For example, if the
   command-line is `helm fullstatus ngin<TAB>`, the `plugin.complete` script can
   print *all* release names (of the `default` namespace), not just the ones
   starting with `ngin`; the shell will only retain the ones starting with
   `ngin`.
1. To simplify dynamic completion support, especially if you have a complex
   plugin, you can have your  `plugin.complete` script call your main plugin
   script and request completion choices.  See the [Dynamic
   Completion](#dynamic-completion) section above for an example.
1. To debug dynamic completion and the `plugin.complete` file, one can run the
   following to see the completion results :
    - `helm __complete <pluginName> <arguments to complete>`.  For example:
    - `helm __complete fullstatus --output js<ENTER>`,
    - `helm __complete fullstatus -o json ""<ENTER>`
