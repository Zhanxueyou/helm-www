---
title: "Helm Get Notes"
---

## helm get notes

download the notes for a named release

### 简介


This command shows notes provided by the chart of a named release.


```
helm get notes RELEASE_NAME [flags]
```

### 可选项

```
  -h, --help           help for notes
      --revision int   get the named release with revision
```

### 从父命令继承的命令

```
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

* [helm get](helm_get.md)	 - download extended information of a named release


