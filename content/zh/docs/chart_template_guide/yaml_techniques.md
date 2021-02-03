---
title: "附录： YAML技术"
description: "详细描述了YAML规范以及如何应用于Helm。"
weight: 15
---

本指南大部分都聚焦于编写模板语言。这里，我们要看看YAML格式。作为模板作者，YAML有一些有用的特性
使我们的模板不易出错，更易阅读。

## 标量和集合

根据 [YAML 规范](https://yaml.org/spec/1.2/spec.html)，有两种集合类型和很多标量类型。

两种集合类型是map和sequence：

```yaml
map:
  one: 1
  two: 2
  three: 3

sequence:
  - one
  - two
  - three
```

标量值是单个值，（与集合相反）

### YAML中的标量类型

在Helm内部的YAML语言中，一个标量值的数据类型是由一组复杂的规则决定的，包含了资源定义的Kubernetes模式。
但在推断类型时，以下规则往往是正确的。

如果整形或浮点型数字没有引号，通常被视为数字类型：

```yaml
count: 1
size: 2.34
```

但是如果被引号引起来，会被当做字符串：

```yaml
count: "1" # <-- string, not int
size: '2.34' # <-- string, not float
```

布尔函数也是如此：

```yaml
isGood: true   # bool
answer: "true" # string
```

空字符串是 `null` （不是 `nil`）。

注意 `port: "80"`是合法的YAML，可以通过模板引擎和YAML解释器传值，但是如果Kubernetes希望`port`是整形，就会失败。

在一些场景中，可以使用YAML节点标签强制推断特定类型：

```yaml
coffee: "yes, please"
age: !!str 21
port: !!int "80"
```

如上所示，`!!str`告诉解释器`age`是一个字符串，即使它看起来像是整形。即使`port`被引号括起来，也会被视为int。

## YAML中的字符串

YAML中的大多数数据都是字符串。YAML有多种表示字符串的方法。本节解释这些方法并演示如何使用其中一些方法。

有三种单行方式声明一个字符串：

```yaml
way1: bare words
way2: "double-quoted strings"
way3: 'single-quoted strings'
```

单行样式必须在一行：

- 裸字没有引号，也没有转义，因此，必须小心使用字符。
- 双引号字符串可以使用`\`转义指定字符。比如，`"\"Hello\", she said"`。可以使用`\n`转义换行。
- 单引号字符串是字面意义的字符串，不用`\`转义，只有单引号`''`需要转义，转成单个`'`。

除了单行字符串，可以声明多行字符串：

```yaml
coffee: |
  Latte
  Cappuccino
  Espresso
```

上述会被当作`coffee`的字符串值，等同于`Latte\nCappuccino\nEspresso\n`。

注意在第一行`|`后面必须正确缩进。可以这样破坏上述示例：

```yaml
coffee: |
         Latte
  Cappuccino
  Espresso

```

由于`Latte`没有正确缩进，会遇到这样的错误：

```shell
Error parsing file: error converting YAML to JSON: yaml: line 7: did not find expected key
```

模板中，有时候为了避免上述错误，在多行文本中添加一个假的“第一行”会更加安全：

```yaml
coffee: |
  # Commented first line
         Latte
  Cappuccino
  Espresso

```

Note that whatever that first line is, it will be preserved in the output of the
string. So if you are, for example, using this technique to inject a file's
contents into a ConfigMap, the comment should be of the type expected by
whatever is reading that entry.

### Controlling Spaces in Multi-line Strings

In the example above, we used `|` to indicate a multi-line string. But notice
that the content of our string was followed with a trailing `\n`. If we want the
YAML processor to strip off the trailing newline, we can add a `-` after the
`|`:

```yaml
coffee: |-
  Latte
  Cappuccino
  Espresso
```

Now the `coffee` value will be: `Latte\nCappuccino\nEspresso` (with no trailing
`\n`).

Other times, we might want all trailing whitespace to be preserved. We can do
this with the `|+` notation:

```yaml
coffee: |+
  Latte
  Cappuccino
  Espresso


another: value
```

Now the value of `coffee` will be `Latte\nCappuccino\nEspresso\n\n\n`.

Indentation inside of a text block is preserved, and results in the preservation
of line breaks, too:

```yaml
coffee: |-
  Latte
    12 oz
    16 oz
  Cappuccino
  Espresso
```

In the above case, `coffee` will be `Latte\n  12 oz\n  16
oz\nCappuccino\nEspresso`.

### Indenting and Templates

When writing templates, you may find yourself wanting to inject the contents of
a file into the template. As we saw in previous chapters, there are two ways of
doing this:

- Use `{{ .Files.Get "FILENAME" }}` to get the contents of a file in the chart.
- Use `{{ include "TEMPLATE" . }}` to render a template and then place its
  contents into the chart.

When inserting files into YAML, it's good to understand the multi-line rules
above. Often times, the easiest way to insert a static file is to do something
like this:

```yaml
myfile: |
{{ .Files.Get "myfile.txt" | indent 2 }}
```

Note how we do the indentation above: `indent 2` tells the template engine to
indent every line in "myfile.txt" with two spaces. Note that we do not indent
that template line. That's because if we did, the file content of the first line
would be indented twice.

### Folded Multi-line Strings

Sometimes you want to represent a string in your YAML with multiple lines, but
want it to be treated as one long line when it is interpreted. This is called
"folding". To declare a folded block, use `>` instead of `|`:

```yaml
coffee: >
  Latte
  Cappuccino
  Espresso


```

The value of `coffee` above will be `Latte Cappuccino Espresso\n`. Note that all
but the last line feed will be converted to spaces. You can combine the
whitespace controls with the folded text marker, so `>-` will replace or trim
all newlines.

Note that in the folded syntax, indenting text will cause lines to be preserved.

```yaml
coffee: >-
  Latte
    12 oz
    16 oz
  Cappuccino
  Espresso
```

The above will produce `Latte\n  12 oz\n  16 oz\nCappuccino Espresso`. Note that
both the spacing and the newlines are still there.

## Embedding Multiple Documents in One File

It is possible to place more than one YAML documents into a single file. This is
done by prefixing a new document with `---` and ending the document with
`...`

```yaml

---
document:1
...
---
document: 2
...
```

In many cases, either the `---` or the `...` may be omitted.

Some files in Helm cannot contain more than one doc. If, for example, more than
one document is provided inside of a `values.yaml` file, only the first will be
used.

Template files, however, may have more than one document. When this happens, the
file (and all of its documents) is treated as one object during template
rendering. But then the resulting YAML is split into multiple documents before
it is fed to Kubernetes.

We recommend only using multiple documents per file when it is absolutely
necessary. Having multiple documents in a file can be difficult to debug.

## YAML is a Superset of JSON

Because YAML is a superset of JSON, any valid JSON document _should_ be valid
YAML.

```json
{
  "coffee": "yes, please",
  "coffees": [
    "Latte", "Cappuccino", "Espresso"
  ]
}
```

The above is another way of representing this:

```yaml
coffee: yes, please
coffees:
- Latte
- Cappuccino
- Espresso
```

And the two can be mixed (with care):

```yaml
coffee: "yes, please"
coffees: [ "Latte", "Cappuccino", "Espresso"]
```

All three of these should parse into the same internal representation.

While this means that files such as `values.yaml` may contain JSON data, Helm
does not treat the file extension `.json` as a valid suffix.

## YAML Anchors

The YAML spec provides a way to store a reference to a value, and later refer to
that value by reference. YAML refers to this as "anchoring":

```yaml
coffee: "yes, please"
favorite: &favoriteCoffee "Cappucino"
coffees:
  - Latte
  - *favoriteCoffee
  - Espresso
```

In the above, `&favoriteCoffee` sets a reference to `Cappuccino`. Later, that
reference is used as `*favoriteCoffee`. So `coffees` becomes `Latte, Cappuccino,
Espresso`.

While there are a few cases where anchors are useful, there is one aspect of
them that can cause subtle bugs: The first time the YAML is consumed, the
reference is expanded and then discarded.

So if we were to decode and then re-encode the example above, the resulting YAML
would be:

```yaml
coffee: yes, please
favorite: Cappucino
coffees:
- Latte
- Cappucino
- Espresso
```

因为Helm和Kubernetes经常读取，修改和重写YAML文件，锚点会丢失。
