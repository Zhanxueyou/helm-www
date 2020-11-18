---
title: "Helm Search"
---

## helm search

search for a keyword in charts

### 简介

Search provides the ability to search for Helm charts in the various places
they can be stored including the Helm Hub and repositories you have added. Use
search subcommands to search different locations for charts.

### 可选项

```shell
  -h, --help   help for search
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
* [helm search hub](helm_search_hub.md) - search for charts in the Helm Hub or an instance of Monocular
* [helm search repo](helm_search_repo.md) - search repositories for a keyword in charts
