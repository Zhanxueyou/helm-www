---
title: "Helm Show"
---

## helm show

show information of a chart

### 简介

This command consists of multiple subcommands to display information about a chart

### 可选项

```shell
  -h, --help   help for show
```

### 从父命令继承的命令

```shell
      --debug                       enable verbose output
      --kube-apiserver string       the address and the port for the Kubernetes API server
      --kube-as-group stringArray   Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --kube-as-user string         Username to impersonate for the operation
      --kube-context string         name of the kubeconfig context to use
      --kube-token string           bearer token used for authentication
      --kubeconfig string           path to the kubeconfig file
  -n, --namespace string            namespace scope for this request
      --registry-config string      path to the registry config file (default "~/.config/helm/registry.json")
      --repository-cache string     path to the file containing cached repository indexes (default "~/.cache/helm/repository")
      --repository-config string    path to the file containing repository names and URLs (default "~/.config/helm/repositories.yaml")
```

### 请参阅

* [helm](helm.md) - 针对Kubernetes的Helm包管理器
* [helm show all](helm_show_all.md) - show all information of the chart
* [helm show chart](helm_show_chart.md) - show the chart's definition
* [helm show readme](helm_show_readme.md) - show the chart's README
* [helm show values](helm_show_values.md) - show the chart's values
