---
title: "本地化Helm文档"
description: "本地化Helm文档的说明。"
weight: 5
---

本指南介绍如何本地化Helm文档。

## 入门

翻译工作使用与文档稿件相同的过程。翻译通过[pull
requests](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests)
提供给 [helm-www](https://github.com/helm/helm-www) 仓库并由管理站点的团队审查。

### 双字母语言代码

语言代码文档由[ISO 639-1标准](https://www.loc.gov/standards/iso639-2/php/code_list.php)组织。
比如，韩国的双字母代码是 `ko`。

在内容和配置中可以找到使用的语音码。三个例子如下：

- `content` 目录中的子目录以语言码命名且本地化内容在对应的目录中。 主要在每个语言码目录的 `docs`子目录中。
- `i18n`目录包含一个网站使用的每种语言的配置文件。文件按照 `[LANG].toml` 形式命名，`[LANG]` 是双字母语言码。
- 顶层`config.toml`文件中，有按语言码组织的导航和其他详细信息的配置。

英文使用语言码 `en`，是翻译的默认语言和资源。

### Fork, Branch, Change, Pull Request

贡献翻译从[helm-www仓库](https://github.com/helm/helm-www)
[创建fork](https://help.github.com/en/github/getting-started-with-github/fork-a-repo)开始。在你的fork中提交更改。

默认情况下，fork将在名为master的默认分支上工作。请使用分支提交更改并创建pull requests。如果你不熟悉分支，可以
[阅读GitHub文档](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-branches)。

一旦有了分支，就可以添加翻译，将内容本地化为一种语言。

注意，Helm使用一个[Developers Certificate of Origin](https://developercertificate.org/)。
所有的提交需要signoff。提交时可以使用 `-s` 或 `--signoff` 参数使用你Git配置的用户和邮箱签署这个提交。
更多细节请查看 [CONTRIBUTING.md](https://github.com/helm/helm-www/blob/master/CONTRIBUTING.md#sign-your-work)
文件。

准备好之后，创建一个 [pull request](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests)
将翻译提交到helm-www仓库。

一旦创建了pull request，维护者会进行审查。过程细节查看 [CONTRIBUTING.md](https://github.com/helm/helm-www/blob/master/CONTRIBUTING.md)。

## 翻译内容

本地化所有的Helm内容是一项巨大的任务。开始很小的改动是可以的。翻译会随着时间扩展。

### 开始一个新语言

When starting a new language there is a minimum needed. This includes:

- Adding a `content/[LANG]/docs` directory containing an `_index.md` file. This
  is the top level documentation landing page.
- Creating a `[LANG].toml` file in the `i18n` directory. Initially you can copy
  the `en.toml` file as a starting point.
- Adding a section for the language to the `config.toml` file to expose the new
  language. An existing language section can serve as a starting point.

### Translating

Translated content needs to reside in the `content/[LANG]/docs` directory. It
should have the same URL as the English source. For example, to translate the
intro into Korean it can be useful to copy the english source like:

```sh
mkdir -p content/ko/docs/intro
cp content/en/docs/intro/install.md content/ko/docs/intro/install.md
```

The content in the new file can then be translated into the other language.

Do not add an untranslated copy of an English file to `content/[LANG]/`.
Once a language exists on the site, any untranslated pages will redirect to
English automatically. Translation takes time, and you always want to be
translating the most current version of the docs, not an outdated fork.

Make sure you remove any `aliases` lines from the header section. A line like
`aliases: ["/docs/using_helm/"]` does not belong in the translations. Those
are redirections for old links which don't exist for new pages.

Note, translation tools can help with the process. This includes machine
generated translations. Machine generated translations should be edited or
otherwise reviewing for grammar and meaning by a native language speaker before
publishing.

## Navigating Between Languages

![Screen Shot 2020-05-11 at 11 24 22
AM](https://user-images.githubusercontent.com/686194/81597103-035de600-937a-11ea-9834-cd9dcef4e914.png)

The site global
[config.toml](https://github.com/helm/helm-www/blob/master/config.toml#L83L89)
file is where language navigation is configured.

To add a new language, add a new set of parameters using the [two-letter
language code](./localization/#two-letter-language-code) defined above. Example:

```toml
# Korean
[languages.ko]
title = "Helm"
description = "Helm - The Kubernetes Package Manager."
contentDir = "content/ko"
languageName = "한국어 Korean"
weight = 1
```

## Resolving Internal Links

Translated content will sometimes include links to pages that only exist in
another language. This will result in site [build
errors](https://app.netlify.com/sites/helm-merge/deploys). Example:

```shell
12:45:31 PM: htmltest started at 12:45:30 on app
12:45:31 PM: ========================================================================
12:45:31 PM: ko/docs/chart_template_guide/accessing_files/index.html
12:45:31 PM:   hash does not exist --- ko/docs/chart_template_guide/accessing_files/index.html --> #basic-example
12:45:31 PM: ✘✘✘ failed in 197.566561ms
12:45:31 PM: 1 error in 212 documents
```

To resolve this, you need to check your content for internal links.

- anchor links need to reflect the translated `id` value
- internal page links need to be fixed

For internal pages that do not exist _(or have not been translated yet)_, the
site will not build until a correction is made. As a fallback, the url can point
to another language where that content _does_ exist as follows:

`< relref path="/docs/topics/library_charts.md" lang="en" >`

See the [Hugo Docs on cross references between
languages](https://gohugo.io/content-management/cross-references/#link-to-another-language-version)
for more info.
