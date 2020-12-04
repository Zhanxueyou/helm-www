---
title: "依赖"
description: "涵盖chart依赖的最佳实践"
weight: 4
---

最佳实践的这部分阐述`Chart.yaml`中声明的`dependencies`。

## 版本

如果有可能的话，使用版本范围而不是某个固定的版本。建议的默认设置时使用补丁级别版本的匹配：

```yaml
version: ~1.2.3
```

这样会匹配 `1.2.3`以及该版本的任何补丁，也就是说，`~1.2.3`相当于`>= 1.2.3, < 1.3.0`

For the complete version matching syntax, please see the [semver
documentation](https://github.com/Masterminds/semver#checking-version-constraints).

### Repository URLs

Where possible, use `https://` repository URLs, followed by `http://` URLs.

If the repository has been added to the repository index file, the repository
name can be used as an alias of URL. Use `alias:` or `@` followed by repository
names.

File URLs (`file://...`) are considered a "special case" for charts that are
assembled by a fixed deployment pipeline. Charts that use `file://` are not
allowed in the official Helm repository.

## Conditions and Tags

Conditions or tags should be added to any dependencies that _are optional_.

The preferred form of a condition is:

```yaml
condition: somechart.enabled
```

Where `somechart` is the chart name of the dependency.

When multiple subcharts (dependencies) together provide an optional or swappable
feature, those charts should share the same tags.

For example, if both `nginx` and `memcached` together provide performance
optimizations for the main app in the chart, and are required to both be present
when that feature is enabled, then they should both have a tags section like
this:

```yaml
tags:
  - webaccelerator
```

This allows a user to turn that feature on and off with one tag.
