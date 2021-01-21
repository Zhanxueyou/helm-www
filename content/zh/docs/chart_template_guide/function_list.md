---
title: "模板函数列表"
description: "Helm中模板函数变量的列表"
weight: 6
---

Helm 包含了很多可以在模板中利用的模板函数。以下列出了具体分类：

* [Cryptographic and Security](#cryptographic-and-security-functions)
* [Date](#date-functions)
* [Dictionaries](#dictionaries-and-dict-functions)
* [Encoding](#encoding-functions)
* [File Path](#file-path-functions)
* [Kubernetes and Chart](#kubernetes-and-chart-functions)
* [Logic and Flow Control](#logic-and-flow-control-functions)
* [Lists](#lists-and-list-functions)
* [Math](#math-functions)
* [Network](#network-functions)
* [Reflection](#reflection-functions)
* [Regular Expressions](#regular-expressions)
* [Semantic Versions](#semantic-version-functions)
* [String](#string-functions)
* [Type Conversion](#type-conversion-functions)
* [URL](#url-functions)
* [UUID](#uuid-functions)

## 逻辑和流控制函数

Helm 包括了需要逻辑和流控制函数，包括[and](#and),[coalesce](#coalesce), [default](#default),
[empty](#empty), [eq](#eq),[fail](#fail), [ge](#ge), [gt](#gt), [le](#le), [lt](#lt),
[ne](#ne), [not](#not), and [or](#or)。

### and

返回两个参数的and布尔值。

```yaml
and .Arg1 .Arg2
```

### or

返回两个参数的or布尔值。会返回第一个非空参数或最后一个参数。

```yaml
or .Arg1 .Arg2
```

### not

返回参数的布尔求反。

```yaml
not .Arg
```

### eq

返回参数的布尔等式（比如， Arg1 == Arg2）。

```yaml
eq .Arg1 .Arg2
```

### ne

返回参数的布尔非等式（比如 Arg1 != Arg2）。

```yaml
ne .Arg1 .Arg2
```

### lt

如果第一参数小于第二参数，返回布尔真。否则返回假（比如， Arg1 < Arg2）。

```yaml
lt .Arg1 .Arg2
```

### le

如果第一参数小于等于第二参数，返回布尔真，否则返回假（比如， Arg1 <= Arg2）。

```yaml
le .Arg1 .Arg2
```

### gt

如果第一参数大于第二参数，返回布尔真，否则返回假（比如， Arg1 > Arg2）。

```yaml
gt .Arg1 .Arg2
```

### ge

如果第一参数大于等于第二参数，返回布尔真，否则返回假。（比如， Arg1 >= Arg2）。

```yaml
ge .Arg1 .Arg2
```

### default

使用`default`设置一个简单的默认值。

```yaml
default "foo" .Bar
```

上述示例中，如果`.Bar`是非空值，则使用它，否则会返回`foo`。

"空"定义取决于以下类型：

* 整形: 0
* 字符串: ""
* 列表: `[]`
* 字典: `{}`
* 布尔: `false`
* 以及所有的`nil` (或 null)

对于结构体，没有空的定义，所以结构体从来不会返回默认值。

### empty

如果给定的值被认为是空的，则`empty`函数返回`true`，否则返回`false`。空值列举在`default`部分。

```yaml
empty .Foo
```

注意在Go模板条件中，空值是为你计算出来的。这样你很少需要 `if empty .Foo` ，仅使用 `if .Foo` 即可。

### fail

无条件地返回带有指定文本的空 `string` 或者 `error`。这在其他条件已经确定而模板渲染应该失败的情况下很有用。

```yaml
fail "Please accept the end user license agreement"
```

### coalesce

`coalesce`函数获取一个列表并返回第一个非空值。

```yaml
coalesce 0 1 2
```

上述会返回`1`。

此函数用于扫描多个变量或值：

```yaml
coalesce .name .parent.name "Matt"
```

上述示例会优先检查 `.name` 是否为空。如果不是，就返回值。如果 _是_ 空, 继续检查`.parent.name`。
最终，如果 `.name` 和 `.parent.name` 都是空，就会返回 `Matt`。

### ternary

`ternary`函数获取两个值和一个test值。如果test值是true，则返回第一个值。如果test值是空，则返回第二个值。
这和c的ternary运算符类似。

#### true test value

```yaml
ternary "foo" "bar" true
```

或者

```yaml
true | ternary "foo" "bar"
```

上述返回 `"foo"`。

#### false test value

```yaml
ternary "foo" "bar" false
```

或者

```yaml
false | ternary "foo" "bar"
```

上述返回 `"bar"`.

## 字符串函数

Helm 包含了一下字符串函数： [abbrev](#abbrev),
[abbrevboth](#abbrevboth), [camelcase](#camelcase), [cat](#cat),
[contains](#contains), [hasPrefix](#hasprefix-and-hassuffix),
[hasSuffix](#hasprefix-and-hassuffix), [indent](#indent), [initials](#initials),
[kebabcase](#kebabcase), [lower](#lower), [nindent](#nindent),
[nospace](#nospace), [plural](#plural), [print](#print), [printf](#printf),
[println](#println), [quote](#quote-and-squote),
[randAlpha](#randalphanum-randalpha-randnumeric-and-randascii),
[randAlphaNum](#randalphanum-randalpha-randnumeric-and-randascii),
[randAscii](#randalphanum-randalpha-randnumeric-and-randascii),
[randNumeric](#randalphanum-randalpha-randnumeric-and-randascii),
[repeat](#repeat), [replace](#replace), [shuffle](#shuffle),
[snakecase](#snakecase), [squote](#quote-and-squote), [substr](#substr),
[swapcase](#swapcase), [title](#title), [trim](#trim), [trimAll](#trimall),
[trimPrefix](#trimprefix), [trimSuffix](#trimsuffix), [trunc](#trunc),
[untitle](#untitle), [upper](#upper), [wrap](#wrap), 和 [wrapWith](#wrapwith)

### print

返回各部分组合的字符串。

```yaml
print "Matt has " .Dogs " dogs"
```

如果可能，非字符串类型会被转换成字符串。

注意，当相邻两个参数不是字符串时会在它们之间添加一个空格。

### println

和[print](#print)效果一样，但会在末尾新添加一行。

### printf

返回参数按顺序传递的格式化字符串。

```yaml
printf "%s has %d dogs." .Name .NumberDogs
```

占位符取决于传入的参数类型。这包括：

一般用途：

* `%v` 默认格式的值
  * 当打印字典时，加号参数(`%+v`)可以添加字段名称
* `%%` 字符百分号，不使用值

布尔值：

* `%t` true或false

整形：

* `%b` 二进制
* `%c` the character represented by the corresponding Unicode code point
* `%d` 十进制
* `%o` 8进制
* `%O` 带0o前缀的8进制
* `%q` 安全转义的单引号字符
* `%x` 16进制，使用小写字符a-f
* `%X` 16进制，使用小写字符A-F
* `%U` Unicode格式： U+1234; 和"U+%04X"相同

浮点数和复杂成分

* `%b` 指数二次幂的无小数科学计数法，比如 -123456p-78
* `%e` 科学计数法，比如： -1.234456e+78
* `%E` 科学计数法，比如： -1.234456E+78
* `%f` 无指数的小数，比如： 123.456
* `%F` 与%f同义
* `%g` %e的大指数，否则是%f
* `%G` %E的大指数，否则是%F
* `%x` 十六进制计数法(和两个指数的十进制幂)，比如： -0x1.23abcp+20
* `%X` 大写的十六进制计数法，比如： -0X1.23ABCP+20

字符串和字节切片：

* `%s` 未解析的二进制字符串或切片
* `%q` 安全转义的双引号字符串
* `%x` 十六进制，小写，每个字节两个字符
* `%X` 十六进制，大写，每个字节两个字符

切片：

* `%p` 16进制的第0个元素的地址，以0x开头

### trim

`trim`行数移除字符串两边的空格：

```yaml
trim "   hello    "
```

上述结果为： `hello`

### trimAll

从字符串中移除给定的字符：

```yaml
trimAll "$" "$5.00"
```

上述结果为：`5.00` (作为一个字符串)。

### trimPrefix

从字符串中移除前缀：

```yaml
trimPrefix "-" "-hello"
```

上述结果为：`hello`

### trimSuffix

从字符串中移除后缀：

```yaml
trimSuffix "-" "hello-"
```

上述结果为： `hello`

### lower

将整个字符串转换成小写：

```yaml
lower "HELLO"
```

上述结果为： `hello`

### upper

将整个字符串转换成大写：

```yaml
upper "hello"
```

上述结果为： `HELLO`

### title

首字母转换成大写：

```yaml
title "hello world"
```

上述结果为： `Hello World`

### untitle

移除首字母大写：`untitle "Hello World"` 会得到 `hello world`.

### repeat

重复字符串多次：

```yaml
repeat 3 "hello"
```

上述结果为： `hellohellohello`

### substr

获取字符串的子串，有三个参数：

* start (int)
* end (int)
* string (string)

```yaml
substr 0 5 "hello world"
```

上述结果为： `hello`

### nospace

去掉字符串中的所有空格：

```yaml
nospace "hello w o r l d"
```

上述结果为： `helloworld`

### trunc

截断字符串。

```yaml
trunc 5 "hello world"
```

上述结果为： `hello`.

```yaml
trunc -5 "hello world"
```

上述结果为： `world`.

### abbrev

用省略号截断字符串 (`...`)

参数：

* 最大长度
* 字符串

```yaml
abbrev 5 "hello world"
```

上述结果为： `he...`， 因为将省略号算进了长度中。

### abbrevboth

两边都省略

```yaml
abbrevboth 5 10 "1234 5678 9123"
```

上述结果为： `...5678...`

It takes:

* 左侧偏移值
* 最大长度
* 字符串

### initials

截取给定字符串每个单词的首字母，并组合在一起。

```yaml
initials "First Try"
```

上述结果为： `FT`

### randAlphaNum, randAlpha, randNumeric, and randAscii

这四个字符串生成加密安全的(使用 ```crypto/rand```)的随机字符串，但是字符集合不同：

* `randAlphaNum` 使用 `0-9a-zA-Z`
* `randAlpha` 使用 `a-zA-Z`
* `randNumeric` 使用 `0-9`
* `randAscii` 使用所有的可打印ASCII字符

每个函数都需要一个参数：字符串的整形长度

```yaml
randNumeric 3
```

上述会生成三个数字的字符串。

### wrap

以给定列数给文字换行。

```yaml
wrap 80 $someText
```

上述结果会以`$someText`在80列处换行。

### wrapWith

`wrapWith` 和 `wrap` 类似，但可以以指定字符串换行。(`wrap` 使用的是 `\n`)

```yaml
wrapWith 5 "\t" "Hello World"
```

上述结果为： `hello world` (其中空格是ASCII tab字符)

### contains

测试字符串是否包含在另一个字符串中：

```yaml
contains "cat" "catch"
```

上述结果为： `true` 因为 `catch` 包含了 `cat`.

### hasPrefix and hasSuffix

`hasPrefix` 和 `hasSuffix` 函数测试字符串是否有给定的前缀或后缀：

```yaml
hasPrefix "cat" "catch"
```

上述结果为： `true` 因为 `catch` 有 `cat`.

### quote and squote

该函数将字符串用双引号(`quote`) 或者单引号(`squote`)括起来。

### cat

`cat` 函数将多个字符串合并成一个，用空格分隔：

```yaml
cat "hello" "beautiful" "world"
```

上述结果为： `hello beautiful world`

### indent

`indent` 以指定长度缩进给定字符串所在行，在对齐多行字符串时很有用：

```yaml
indent 4 $lots_of_text
```

上述结果会将每行缩进4个空格。

### nindent

`nindent` 函数和indent函数一样，但可以在字符串开头添加新行。

```yaml
nindent 4 $lots_of_text
```

上述结果会在字符串所在行缩进4个字符，并且在开头新添加一行。

### replace

执行简单的字符串替换。

需要三个参数

* 待替换字符串
* 要替换字符串
* 源字符串

```yaml
"I Am Henry VIII" | replace " " "-"
```

上述结果为： `I-Am-Henry-VIII`

### plural

字符串复数化。

```yaml
len $fish | plural "one anchovy" "many anchovies"
```

如上，如果字符串长度为1，则第一个参数会被打印(`one anchovy`）。否则，会打印第二个参数(`many anchovies`）。

参数包括：

* 单数字符串
* 复数字符串
* 整形长度

注意： Helm 现在不支持多语言复杂的复数规则。`0`被认为是复数的因为英文中作为(`zero anchovies`) 对待。

### snakecase

将驼峰写法转换成蛇形写法。

```yaml
snakecase "FirstName"
```

上述结果为： `first_name`.

### camelcase

将字符串从蛇形写法转换成驼峰写法。

```yaml
camelcase "http_server"
```

上述结果为： `HttpServer`。

### kebabcase

将驼峰写法转换成烤串写法。

```yaml
kebabcase "FirstName"
```

上述结果为： `first-name`.

### swapcase

基于单词的算法切换字符串的大小写。

转换算法：

* 大写字符变成小写字母
* 首字母变成小写字母
* 空格后或开头的小写字母转换成大写字母
* 其他小写字母转换成大写字母
* 通过unicode.IsSpace(char)定义空格

```yaml
swapcase "This Is A.Test"
```

上述结果为： `tHIS iS a.tEST`.

### shuffle

对字符串进行洗牌。

```yaml
shuffle "hello"
```

上述`hello`的随机字符串可能会是`oelhl`。

## Type Conversion Functions

Helm提供了以下类型转换函数：

* `atoi`: 字符串转换成整形。
* `float64`: 转换成 `float64`。
* `int`: 按系统整形宽度转换成`int`。
* `int64`: 转换成 `int64`。
* `toDecimal`: 将unix八进制转换成`int64`。
* `toString`: 转换成字符串。
* `toStrings`: 将列表、切片或数组转换成字符串列表。
* `toJson` (`mustToJson`): 将列表、切片、数组、字典或对象转换成JSON。
* `toPrettyJson` (`mustToPrettyJson`): 将列表、切片、数组、字典或对象转换成格式化JSON。
* `toRawJson` (`mustToRawJson`): 将列表、切片、数组、字典或对象转换成HTML字符未转义的JSON。

只有`atoi`需要输入一个特定的类型。其他的会尝试将任何类型转换成目标类型。比如，`int64`可以把浮点数转换成整形，也可以把字符串转换成整形。

### toStrings

给定一个类列表集合，输出字符串切片。

```yaml
list 1 2 3 | toStrings
```

上述会将`1`转成`"1"`，`2`转成`"2"`，等等，然后将其作为列表返回。

### toDecimal

给定一个unix八进制权限，转换成十进制。

```yaml
"0777" | toDecimal
```

上述回将 `0777` 转换成 `511` 并返回int64的值。

### toJson, mustToJson

`toJson`函数将内容编码为JSON字符串。如果内容无法被转换成JSON会返回空字符串。`mustToJson`会返回错误以防无法编码成JSON。

```yaml
toJson .Item
```

上述结果为： `.Item`的JSON字符串表示。

### toPrettyJson, mustToPrettyJson

`toPrettyJson`函数将内容编码为好看的（缩进的）JSON字符串。

```yaml
toPrettyJson .Item
```

上述结果为： `.Item`的已缩进的JSON字符串表示。

### toRawJson, mustToRawJson

`toRawJson` 函数将内容编码成包含非转义HTML字符的JSON字符串。

```yaml
toRawJson .Item
```

上述结果为： `.Item`的非转义的JSON字符串表示。

## Regular Expressions

Helm 包含以下正则表达式函数 [regexFind(mustRegexFind)](#regexfindall-mustregexfindall),
[regexFindAll(mustRegexFindAll)](#regexfind-mustregexfind), [regexMatch
(mustRegexMatch)](#regexmatch-mustregexmatch), [regexReplaceAll
(mustRegexReplaceAll)](#regexreplaceall-mustregexreplaceall),
[regexReplaceAllLiteral(mustRegexReplaceAllLiteral)](#regexreplaceallliteral-mustregexreplaceallliteral),
[regexSplit (mustRegexSplit)](#regexsplit-mustregexsplit)。

### regexMatch, mustRegexMatch

如果输入字符串包含可匹配正则表达式任意字符串，则返回true。

```yaml
regexMatch "^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$" "test@acme.com"
```

上述结果为： `true`

`regexMatch`有问题时会出错， `mustRegexMatch`有问题时会向模板引擎返回错误。

### regexFindAll, mustRegexFindAll

返回输入字符串匹配正则表达式的所有切片。最后一个参数表示要返回的子字符串的数量，-1表示返回所有。

```yaml
regexFindAll "[2,4,6,8]" "123456789" -1
```

上述结果为： `[2 4 6 8]`

`regexFindAll` 有问题时会出错， `mustRegexFindAll` 有问题时会向模板引擎返回错误。

### regexFind, mustRegexFind

返回输入字符串的第一个 (最左边的) 正则匹配。

```yaml
regexFind "[a-zA-Z][1-9]" "abcd1234"
```

上述结果为： `d1`

`regexFind` 有问题时会出错， `mustRegexFind` 有问题时会向模板引擎返回错误。

### regexReplaceAll, mustRegexReplaceAll

返回输入字符串的拷贝，用替换字符串替换Regexp的匹配项。在替换字符串里面， $ 标志被解释为扩展，因此对于实例来说 $1 表示第一个子匹配的文本

```yaml
regexReplaceAll "a(x*)b" "-ab-axxb-" "${1}W"
```

上述结果为： `-W-xxW-`

`regexReplaceAll` 有问题时会出错， `mustRegexReplaceAll` 有问题时会向模板引擎返回错误。

### regexReplaceAllLiteral, mustRegexReplaceAllLiteral

返回输入字符串的拷贝，用替换字符串替换Regexp的匹配项。匹配字符串直接替换而不是扩展。

```yaml
regexReplaceAllLiteral "a(x*)b" "-ab-axxb-" "${1}"
```

上述结果为： `-${1}-${1}-`

`regexReplaceAllLiteral` 有问题时会出错，`mustRegexReplaceAllLiteral` 有问题时会向模板引擎返回错误。

### regexSplit, mustRegexSplit

将输入字符串切成有表达式分隔的子字符串，并返回表达式匹配项之间的切片。最后一个参数`n`确定要返回的子字符串数量，`-1`表示返回所有匹配。

```yaml
regexSplit "z+" "pizza" -1
```

上述结果为： `[pi a]`

`regexSplit` 有问题时会出错，`mustRegexSplit` 有问题时会向模板引擎返回错误。

## Cryptographic and Security Functions

Helm提供了一些高级的加密函数。包括了 [adler32sum](#adler32sum), [buildCustomCert](#buildcustomcert),
[decryptAES](#decryptaes), [derivePassword](#derivepassword),[encryptAES](#encryptaes),
[genCA](#genca), [genPrivateKey](#genprivatekey), [genSelfSignedCert](#genselfsignedcert),
[genSignedCert](#gensignedcert), [htpasswd](#htpasswd), [sha1sum](#sha1sum)， 以及 [sha256sum](#sha256sum)。

### sha1sum

`sha1sum`函数接收一个字符串，并计算它的SHA1摘要。

```yaml
sha1sum "Hello world!"
```

### sha256sum

`sha256sum` 函数接收一个字符串，并计算它的SHA256摘要。

```yaml
sha256sum "Hello world!"
```

上述语句会以“ASCII包装”格式计算SHA 256 校验和，并安全打印出来。

### adler32sum

`adler32sum`函数接收一个字符串，并计算它的Adler-32校验和。 

```yaml
adler32sum "Hello world!"
```

### htpasswd

`htpasswd` 函数使用`username` 和 `password` 生成一个密码的`bcrypt`哈希值。该结果可用于[Apache HTTP 
Server](https://httpd.apache.org/docs/2.4/misc/password_encryptions.html#basic) 的基础认证。

```yaml
htpasswd "myUser" "myPassword"
```

注意，将密码直接存储在模板中并不安全。

### derivePassword

`derivePassword` 函数可用于基于某些共享的“主密码”约束得到特定密码。这方面的算法有[详细说明](https://masterpassword.app/masterpassword-algorithm.pdf)。

```yaml
derivePassword 1 "long" "password" "user" "example.com"
```

注意，将这部分直接存储在模板中并不安全。

### genPrivateKey

`genPrivateKey` 函数生成一个编码成PEM块的新私钥。

第一个参数会采用以下某个值：

* `ecdsa`: 生成椭圆曲线 DSA key (P256)
* `dsa`: 生成 DSA key (L2048N256)
* `rsa`: 生成 RSA 4096 key

### buildCustomCert

`buildCustomCert` 函数允许自定义证书。

会采用以下字符串参数：

* base64 编码PEM格式证书
* base64 编码PEM格式私钥

返回包含以下属性的整数对象：

* `Cert`:PEM编码证书
* `Key`: PEM编码私钥

示例：

```yaml
$ca := buildCustomCert "base64-encoded-ca-crt" "base64-encoded-ca-key"
```

注意返回的对象可以使用这个CA传递给`genSignedCert`函数进行签名。

### genCA

`genCA` 函数生成一个新的，自签名的x509 证书机构。

会采用以下参数：

* 主体通用名 (cn)
* 证书有效期（天）

会返回一个包含以下属性的对象：

* `Cert`: PEM编码证书
* `Key`: PEM编码私钥

示例：

```yaml
$ca := genCA "foo-ca" 365
```

注意返回的对象可以使用这个CA传递给`genSignedCert`函数进行签名。

### genSelfSignedCert

The `genSelfSignedCert` 函数生成一个新的，自签名的x509 证书。

会采用下列参数：

* 主体通用名 (cn)
* 可选IP列表；可以为空
* 可选备用DNS名称列表；可以为空
* 证书有效期（天）

会返回一个包含以下属性的对象：

* `Cert`: PEM编码证书
* `Key`: PEM编码私钥

示例：

```yaml
$cert := genSelfSignedCert "foo.com" (list "10.0.0.1" "10.0.0.2") (list "bar.com" "bat.com") 365
```

### genSignedCert

`genSignedCert` 通过指定的CA签名生成一个新的， x509证书

会采用以下参数：

* 主体通用名 (cn)
* 可选IP列表；可以为空
* 可选备用DNS名称列表；可以为空
* 证书有效期（天）
* CA (查看 `genCA`)

示例：

```yaml
$ca := genCA "foo-ca" 365
$cert := genSignedCert "foo.com" (list "10.0.0.1" "10.0.0.2") (list "bar.com" "bat.com") 365 $ca
```

### encryptAES

`encryptAES` 函数使用AES-256 CBC 加密文本并返回一个base64编码字符串。

```yaml
encryptAES "secretkey" "plaintext"
```

### decryptAES

`decryptAES`函数接收一个AES-256 CBC编码的字符串并返回解密文本。

```yaml
"30tEfhuJSVRhpG97XCuWgz2okj7L8vQ1s6V9zVUPeDQ=" | decryptAES "secretkey"
```

## 日期 Functions

Helm 包含以下可以在模板中使用的函数：[ago](#ago), [date](#date), [dateInZone](#dateinzone),
[dateModify(mustDateModify)](#datemodify-mustdatemodify), [duration](#duration),
[durationRound](#durationround), [htmlDate](#htmldate),
[htmlDateInZone](#htmldateinzone), [now](#now),
[toDate(mustToDate)](#todate-musttodate), and [unixEpoch](#unixepoch)。

### now

当前日期/时间。和其他日期函数一起使用。

### ago

`ago` 函数返回距time.Now的以秒为单位的间隔时间。

```yaml
ago .CreatedAt"
```

返回`time.Duration`的字符串格式

```yaml
2h34m7s
```

### date

`date`函数格式化日期

日期格式化为YEAR-MONTH-DAY：

```yaml
now | date "2006-01-02"
```

日期格式化在Go中有[一些不同](https://pauladamsmith.com/blog/2011/05/go_time.html)。

简言之，以此为基准日期：

```yaml
Mon Jan 2 15:04:05 MST 2006
```

将其写成你想要的格式，上面的例子中，`2006-01-02` 是同一个日期，却是我们需要的格式。

### dateInZone

和 `date` 一样，但是和时区一起。

```yaml
dateInZone "2006-01-02" (now) "UTC"
```

### duration

将给定的秒数格式化为`time.Duration`。

这会返回 1m35s。

```yaml
duration 95
```

### durationRound

将给定时间舍入到最重要的单位。当`time.Time`计算为一个自某个时刻以来的时间，字符串和`time.Duration`被解析为一个时间段。

这会返回2h

```yaml
durationRound "2h10m5s"
```

这会返回3mo

```yaml
durationRound "2400h10m5s"
```

### unixEpoch

返回`time.Time`的unix时间戳。

```yaml
now | unixEpoch
```

### dateModify, mustDateModify

`dateModify` 给定一个修改日期并返回时间戳。

从当前时间减去一个小时三十分钟：

```yaml
now | date_modify "-1.5h"
```

如果修改格式错误， `dateModify`会返回日期未定义。而`mustDateModify`会返回错误。

### htmlDate

`htmlDate`函数用于格式化插入到HTML日期选择器输入字段的日期。

```yaml
now | htmlDate
```

### htmlDateInZone

和htmlDate一样，但多了个时区。

```yaml
htmlDateInZone (now) "UTC"
```

### toDate, mustToDate

`toDate` 将字符串转换成日期。第一个参数是日期格式，第二个参数是日期字符串。
如果字符串无法转换就会返回0值。`mustToDate`以防无法转换会返回错误。

这在你将日期字符串转换成其他格式时很有用（使用pipe）。下面的例子会将"2017-12-31" 转换成 "31/12/2017"。

```yaml
toDate "2006-01-02" "2017-12-31" | date "02/01/2006"
```

## Dictionaries and Dict Functions

Helm 提供了一个key/value存储类型称为`dict`（"dictionary"的简称，Python中也有）。`dict`是无序类型。

字典的key **必须是字符串**。但值可以是任意类型，甚至是另一个`dict` 或 `list`。

不像`list`， `dict`不是不可变的。`set`和`unset`函数会修改字典的内容。

Helm 提供了以下函数支持使用字典：[deepCopy(mustDeepCopy)](#deepcopy-mustdeepcopy),
[dict](#dict), [get](#get),[hasKey](#haskey), [keys](#keys), [merge (mustMerge)](#merge-mustmerge),
[mergeOverwrite (mustMergeOverwrite)](#mergeoverwrite-mustmergeoverwrite),
[omit](#omit), [pick](#pick), [pluck](#pluck), [set](#set), [unset](#unset),和[values](#values)。

### dict

通过调用`dict`函数并传递一个键值对列表创建字典。

下面是创建三个键值对的字典：

```yaml
$myDict := dict "name1" "value1" "name2" "value2" "name3" "value 3"
```

### get

给定一个映射和一个键，从映射中获取值。

```yaml
get $myDict "key1"
```

上述结果为： `"value1"`

注意如果没有找到，会简单返回`""`。不会生成error。

### set

使用`set`给字典添加一个键值对。

```yaml
$_ := set $myDict "name4" "value4"
```

注意`set` _返回字典_ (Go模板函数的一个要求)，因此你可能需要像上面那样使用使用`$_`赋值来获取值。

### unset

给定一个映射和key，从映射中删除这个key。

```yaml
$_ := unset $myDict "name4"
```

和`set`一样，需要返回字典。

注意，如果key没有找到，这个操作会简单返回，不会生成错误。

### hasKey

`hasKey`函数会在给定字典中包含了给定key时返回`true`。

```yaml
hasKey $myDict "name1"
```

如果key没找到，会返回`false`。

### pluck

`pluck` 函数给定一个键和多个映射，并获得所有匹配项的列表：

```yaml
pluck "name1" $myDict $myOtherDict
```

上述会返回的`list`包含了每个找到的值(`[value1 otherValue1]`)。

如果可以在映射中 _没有找到_ ，列表中的映射就不会有内容，返回列表的长度也会小于调用`pluck`的字典。

如果key是 _存在的_， 但是值是空值，会插入一个值。

Helm模板中的一个常见用法是`pluck... | first` 从字典集合中获取第一个匹配的键。

### merge, mustMerge

将两个或多个字典合并为一个， 目标字典优先：

```yaml
$newdict := merge $dest $source1 $source2
```

这是个深度合并操作，但不是深度拷贝操作。合并的嵌套对象是两个字典上的同一实例。如果想深度合并的同时进行深度拷贝，
合并的时候同时使用`deepCopy`函数，比如：

```yaml
deepCopy $source | merge $dest
```

`mustMerge` 会返回错误，以防出现不成功的合并。

### mergeOverwrite, mustMergeOverwrite

合并两个或多个字典，优先按照 **从右到左**，在目标字典中有效地覆盖值：

给定的：

```yaml
dst:
  default: default
  overwrite: me
  key: true

src:
  overwrite: overwritten
  key: false
```

会生成：

```yaml
newdict:
  default: default
  overwrite: overwritten
  key: false
```

```yaml
$newdict := mergeOverwrite $dest $source1 $source2
```

这是一个深度合并操作但不是深度拷贝操作。两个字典上嵌入的对象被合并到了同一个实例中。如果你想在合并的同时进行深度拷贝，
使用`deepCopy`函数，比如：

```yaml
deepCopy $source | mergeOverwrite $dest
```

`mustMergeOverwrite` 会返回错误，以防出现不成功的合并。

### keys

`keys`函数会返回一个或多个`dict`类型中所有的key的`list`。由于字典是 _无序的_，key不会有可预料的顺序。
可以使用`sortAlpha`存储。

```yaml
keys $myDict | sortAlpha
```

当提供了多个词典时，key会被串联起来。使用`uniq`函数和`sortAlpha`获取一个唯一有序的键列表。

```yaml
keys $myDict $myOtherDict | uniq | sortAlpha
```

### pick

`pick`函数只从字典中选择给定的键，并创建一个新的`dict`。

```yaml
$new := pick $myDict "name1" "name2"
```

上述结果为： `{name1: value1, name2: value2}`

### omit

`omit`函数类似于`pick`，除它之外返回一个新的`dict`，所有的key _不_ 匹配给定的key。

```yaml
$new := omit $myDict "name1" "name3"
```

上述结果为： `{name2: value2}`

### values

`values`函数类似于`keys`，返回一个新的`list`包含源字典中所有的value(只支持一个字典)。

```yaml
$vals := values $myDict
```

上述结果为： `list["value1", "value2", "value 3"]`。注意 `values`不能保证结果的顺序-如果你需要顺序，
请使用`sortAlpha`。

### deepCopy, mustDeepCopy

`deepCopy` 和 `mustDeepCopy` 函数给定一个值并深度拷贝这个值。包括字典和其他结构体。 `deepCopy`有问题时会出错，
而`mustDeepCopy`会返回一个错误给模板系统。

```yaml
dict "a" 1 "b" 2 | deepCopy
```

### 字典的内部说明

`dict` 在Go里是作为`map[string]interface{}`执行的。Go开发者可以传`map[string]interface{}`值给上下文，
将其作为 `dict` 提供给模板。

## 编码函数

Helm有以下编码和解码函数：

* `b64enc`/`b64dec`: 编码或解码 Base64
* `b32enc`/`b32dec`: 编码或解码 Base32

## 列表和列表函数

Helm 提供了一个简单的`list`类型，包含任意顺序的列表。类似于数组或切片，但列表是被设计用于不可变数据类型。

创建一个整形列表：

```yaml
$myList := list 1 2 3 4 5
```

上述会生成一个列表 `[1 2 3 4 5]`。

Helm 提供了以下列表函数： [append(mustAppend)](#append-mustappend), [compact
(mustCompact)](#compact-mustcompact), [concat](#concat),
[first(mustFirst)](#first-mustfirst), [has (mustHas)](#has-musthas), [initial
(mustInitial)](#initial-mustinitial), [last (mustLast)](#last-mustlast),
[prepend (mustPrepend)](#prepend-mustprepend), [rest
(mustRest)](#rest-mustrest), [reverse (mustReverse)](#reverse-mustreverse),
[seq](#seq), [slice (mustSlice)](#slice-mustslice), [uniq
(mustUniq)](#uniq-mustuniq), [until](#until), [untilStep](#untilstep), 和
[without (mustWithout)](#without-mustwithout)。

### first, mustFirst

获取列表中的第一项，使用 `first`。

`first $myList` 返回 `1`

`first` 有问题时会出错，`mustFirst` 有问题时会向模板引擎返回错误。

### rest, mustRest

获取列表的尾部内容(除了第一项外的所有内容)，使用`rest`。

`rest $myList` returns `[2 3 4 5]`

`rest`有问题时会出错，`mustRest` 有问题时会向模板引擎返回错误。

### last, mustLast

使用`last`获取列表的最后一项：

`last $myList` 返回 `5`。这大致类似于反转列表然后调用`first`。

### initial, mustInitial

通过返回所有元素 _但_ 除了最后一个元素来赞赏`last`。 `initial $myList` 返回 `[1 2 3 4]`。

`initial`有问题时会出错，但是 `mustInitial` 有问题时会向模板引擎返回错误。

### append, mustAppend

在已有列表中追加一项，创建一个新的列表。

```yaml
$new = append $myList 6
```

上述语句会设置 `$new` 为 `[1 2 3 4 5 6]`。 `$myList`会保持不变。

`append` 有问题时会出错，但 `mustAppend` 有问题时会向模板引擎返回错误。

### prepend, mustPrepend

将元素添加到列表的前面，生成一个新的列表。

```yaml
prepend $myList 0
```

上述语句会生成 `[0 1 2 3 4 5]`。 `$myList`会保持不变。

`prepend` 有问题时会出错，但 `mustPrepend` 有问题时会向模板引擎返回错误。

### concat

将任意数量的列表串联成一个。

```yaml
concat $myList ( list 6 7 ) ( list 8 )
```

上述语句会生成 `[1 2 3 4 5 6 7 8]`。 `$myList` 会保持不变。

### reverse, mustReverse

反转给定的列表生成一个新列表。

```yaml
reverse $myList
```

上述语句会生成一个列表： `[5 4 3 2 1]`。

`reverse` 有问题时会出错，但 `mustReverse` 有问题时会向模板引擎返回错误。

### uniq, mustUniq

生成一个移除重复项的列表。

```yaml
list 1 1 1 2 | uniq
```

上述语句会生成 `[1 2]`

`uniq` 有问题时会出错，但 `mustUniq` 有问题时会向模板引擎返回错误。

### without, mustWithout

`without` 函数从列表中过滤内容。

```yaml
without $myList 3
```

上述语句会生成 `[1 2 4 5]`

一个过滤器可以过滤多个元素：

```yaml
without $myList 1 3 5
```

这样会得到： `[2 4]`

`without` 有问题时会出错，但 `mustWithout` 有问题时会向模板引擎返回错误。

### has, mustHas

验证列表是否有特定元素。

```yaml
has 4 $myList
```

上述语句会返回 `true`, 但 `has "hello" $myList` 就会返回false。

`has` 有问题时会出错，但 `mustHas` 有问题时会向模板引擎返回错误。

### compact, mustCompact

接收一个列表并删除空值项。

```yaml
$list := list 1 "a" "foo" ""
$copy := compact $list
```

`compact` 会返回一个移除了空值(比如， "")的新列表。

`compact` 有问题时会出错，但 `mustCompact` 有问题时会向模板引擎返回错误。

### slice, mustSlice

从列表中获取部分元素，使用 `slice list [n] [m]`。等同于 `list[n:m]`.

* `slice $myList` 返回 `[1 2 3 4 5]`。 等同于 `myList[:]`。
* `slice $myList 3` 返回 `[4 5]`等同于 `myList[3:]`。
* `slice $myList 1 3` 返回 `[2 3]`等同于 `myList[1:3]`。
* `slice $myList 0 3` 返回 `[1 2 3]`等同于 `myList[:3]`。

`slice` 有问题时会出错，但 `mustSlice` 有问题时会向模板引擎返回错误。

### until

`until` 函数构建一个整数范围。

```yaml
until 5
```

上述语句会生成一个列表： `[0, 1, 2, 3, 4]`。

对循环语句很有用： `range $i, $e := until 5`。

### untilStep

类似`until`， `untilStep` 生成一个可计数的整形列表。但允许你定义开始，结束和步长：

```yaml
untilStep 3 6 2
```

上述语句会生成 `[3 5]`，从3开始，每次加2，直到大于等于6。类似于Python的 `range` 函数。

### seq

原理和bash的 `seq` 命令类似。

* 1 单个参数  (结束位置) - 会生成所有从1到包含 `end` 的整数。
* 2 多个参数 (开始， 结束) - 会生成所有包含`start` 和 `end` 的整数，递增或者递减。
* 3 多个参数 (开始， 步长， 结束) - 会生成所有包含 `start` 和 `end` 按 `step`递增或递减的整数。

```yaml
seq 5       => 1 2 3 4 5
seq -3      => 1 0 -1 -2 -3
seq 0 2     => 0 1 2
seq 2 -2    => 2 1 0 -1 -2
seq 0 2 10  => 0 2 4 6 8 10
seq 0 -2 -5 => 0 -2 -4
```

## Math 函数

除非另外指定，否则所有的math函数都是操作 `int64` 的值。

有以下math函数可用： [add](#add), [add1](#add1),
[ceil](#ceil), [div](#div), [floor](#floor), [len](#len), [max](#max),
[min](#min), [mod](#mod), [mul](#mul), [round](#round), and [sub](#sub）。

### add

使用`add`求和。接受两个或多个输入。

```yaml
add 1 2 3
```

### add1

自增加1，使用 `add1`

### sub

相减使用 `sub`

### div

整除使用 `div`

### mod

取模使用`mod`

### mul

相乘使用`mul`。接受两个或多个输入。

```yaml
mul 1 2 3
```

### max

返回一组整数中最大的整数。

下列会返回`3`:

```yaml
max 1 2 3
```

### min

返回一组数中最小的数。

`min 1 2 3` 会返回 `1`

### floor

Returns the greatest float value less than or equal to input value

`floor 123.9999` will return `123.0`

### ceil

Returns the greatest float value greater than or equal to input value

`ceil 123.001` will return `124.0`

### round

Returns a float value with the remainder rounded to the given number to digits
after the decimal point.

`round 123.555555 3` will return `123.556`

### len

Returns the length of the argument as an integer.

```yaml
len .Arg
```

## Network Functions

Helm has a single network function, `getHostByName`.

The `getHostByName` receives a domain name and returns the ip address.

```yaml
getHostByName "www.google.com" would return the corresponding ip address of www.google.com
```

## File Path Functions

While Helm template functions do not grant access to the filesystem, they do
provide functions for working with strings that follow file path conventions.
Those include [base](#base), [clean](#clean), [dir](#dir), [ext](#ext), and
[isAbs](#isabs）。

### base

Return the last element of a path.

```yaml
base "foo/bar/baz"
```

The above prints "baz"

### dir

Return the directory, stripping the last part of the path. So `dir
"foo/bar/baz"` returns `foo/bar`

### clean

Clean up a path.

```yaml
clean "foo/bar/../baz"
```

The above resolves the `..` and returns `foo/baz`

### ext

Return the file extension.

```yaml
ext "foo.bar"
```

上述结果为： `.bar`.

### isAbs

To check whether a file path is absolute, use `isAbs`.

## Reflection Functions

Helm provides rudimentary reflection tools. These help advanced template
developers understand the underlying Go type information for a particular value.
Helm is written in Go and is strongly typed. The type system applies within
templates.

Go has several primitive _kinds_, like `string`, `slice`, `int64`, and `bool`.

Go has an open _type_ system that allows developers to create their own types.

Helm provides a set of functions for each via [kind functions](#kind-functions)
and [type functions](#type-functions）。 A [deepEqual](#deepequal) function is
also provided to compare to values.

### Kind Functions

There are two Kind functions: `kindOf` returns the kind of an object.

```yaml
kindOf "hello"
```

The above would return `string`. For simple tests (like in `if` blocks), the
`isKind` function will let you verify that a value is a particular kind:

```yaml
kindIs "int" 123
```

The above will return `true`

### Type Functions

Types are slightly harder to work with, so there are three different functions:

* `typeOf` returns the underlying type of a value: `typeOf $foo`
* `typeIs` is like `kindIs`, but for types: `typeIs "*io.Buffer" $myVal`
* `typeIsLike` works as `typeIs`, except that it also dereferences pointers.

**Note:** None of these can test whether or not something implements a given
interface, since doing so would require compiling the interface in ahead of
time.

### deepEqual

`deepEqual` returns true if two values are ["deeply
equal"](https://golang.org/pkg/reflect/#DeepEqual)

Works for non-primitive types as well (compared to the built-in `eq`）。

```yaml
deepEqual (list 1 2 3) (list 1 2 3)
```

The above will return `true`

## Semantic Version Functions

Some version schemes are easily parseable and comparable. Helm provides
functions for working with [SemVer 2](http://semver.org) versions. These include
[semver](#semver) and [semverCompare](#semvercompare）。 Below you will also find
details on using ranges for comparisons.

### semver

The `semver` function parses a string into a Semantic Version:

```yaml
$version := semver "1.2.3-alpha.1+123"
```

_If the parser fails, it will cause template execution to halt with an error._

At this point, `$version` is a pointer to a `Version` object with the following
properties:

* `$version.Major`: The major number (`1` above)
* `$version.Minor`: The minor number (`2` above)
* `$version.Patch`: The patch number (`3` above)
* `$version.Prerelease`: The prerelease (`alpha.1` above)
* `$version.Metadata`: The build metadata (`123` above)
* `$version.Original`: The original version as a string

Additionally, you can compare a `Version` to another `version` using the
`Compare` function:

```yaml
semver "1.4.3" | (semver "1.2.3"）。Compare
```

The above will return `-1`.

The return values are:

* `-1` if the given semver is greater than the semver whose `Compare` method was
 called
* `1` if the version who's `Compare` function was called is greater.
* `0` if they are the same version

(Note that in SemVer, the `Metadata` field is not compared during version
comparison operations.)

### semverCompare

A more robust comparison function is provided as `semverCompare`. This version
supports version ranges:

* `semverCompare "1.2.3" "1.2.3"` checks for an exact match
* `semverCompare "~1.2.0" "1.2.3"` checks that the major and minor versions
  match, and that the patch number of the second version is _greater than or
  equal to_ the first parameter.

The SemVer functions use the [Masterminds semver
library](https://github.com/Masterminds/semver), from the creators of Sprig.

### Basic Comparisons

There are two elements to the comparisons. First, a comparison string is a list
of space or comma separated AND comparisons. These are then separated by || (OR)
comparisons. For example, `">= 1.2 < 3.0.0 || >= 4.2.3"` is looking for a
comparison that's greater than or equal to 1.2 and less than 3.0.0 or is greater
than or equal to 4.2.3.

The basic comparisons are:

* `=`: equal (aliased to no operator)
* `!=`: not equal
* `>`: greater than
* `<`: less than
* `>=`: greater than or equal to
* `<=`: less than or equal to

_Note, according to the Semantic Version specification pre-releases may not be
API compliant with their release counterpart. It says,_

### Working With Prerelease Versions

Pre-releases, for those not familiar with them, are used for software releases
prior to stable or generally available releases. Examples of prereleases include
development, alpha, beta, and release candidate releases. A prerelease may be a
version such as `1.2.3-beta.1` while the stable release would be `1.2.3`. In the
order of precedence, prereleases come before their associated releases. In this
example `1.2.3-beta.1 < 1.2.3`.

According to the Semantic Version specification prereleases may not be API
compliant with their release counterpart. It says,

> A pre-release version indicates that the version is unstable and might not
> satisfy the intended compatibility requirements as denoted by its associated
> normal version.

SemVer comparisons using constraints without a prerelease comparator will skip
prerelease versions. For example, `>=1.2.3` will skip prereleases when looking
at a list of releases while `>=1.2.3-0` will evaluate and find prereleases.

The reason for the `0` as a pre-release version in the example comparison is
because pre-releases can only contain ASCII alphanumerics and hyphens (along
with `.` separators), per the spec. Sorting happens in ASCII sort order, again
per the spec. The lowest character is a `0` in ASCII sort order (see an [ASCII
Table](http://www.asciitable.com/))

Understanding ASCII sort ordering is important because A-Z comes before a-z.
That means `>=1.2.3-BETA` will return `1.2.3-alpha`. What you might expect from
case sensitivity doesn't apply here. This is due to ASCII sort ordering which is
what the spec specifies.

### Hyphen Range Comparisons

There are multiple methods to handle ranges and the first is hyphens ranges.
These look like:

* `1.2 - 1.4.5` which is equivalent to `>= 1.2 <= 1.4.5`
* `2.3.4 - 4.5` which is equivalent to `>= 2.3.4 <= 4.5`

### Wildcards In Comparisons

The `x`, `X`, and `*` characters can be used as a wildcard character. This works
for all comparison operators. When used on the `=` operator it falls back to the
patch level comparison (see tilde below）。 For example,

* `1.2.x` is equivalent to `>= 1.2.0, < 1.3.0`
* `>= 1.2.x` is equivalent to `>= 1.2.0`
* `<= 2.x` is equivalent to `< 3`
* `*` is equivalent to `>= 0.0.0`

### Tilde Range Comparisons (Patch)

The tilde (`~`) comparison operator is for patch level ranges when a minor
version is specified and major level changes when the minor number is missing.
For example,

* `~1.2.3` is equivalent to `>= 1.2.3, < 1.3.0`
* `~1` is equivalent to `>= 1, < 2`
* `~2.3` is equivalent to `>= 2.3, < 2.4`
* `~1.2.x` is equivalent to `>= 1.2.0, < 1.3.0`
* `~1.x` is equivalent to `>= 1, < 2`

### Caret Range Comparisons (Major)

The caret (`^`) comparison operator is for major level changes once a stable
(1.0.0) release has occurred. Prior to a 1.0.0 release the minor versions acts
as the API stability level. This is useful when comparisons of API versions as a
major change is API breaking. For example,

* `^1.2.3` is equivalent to `>= 1.2.3, < 2.0.0`
* `^1.2.x` is equivalent to `>= 1.2.0, < 2.0.0`
* `^2.3` is equivalent to `>= 2.3, < 3`
* `^2.x` is equivalent to `>= 2.0.0, < 3`
* `^0.2.3` is equivalent to `>=0.2.3 <0.3.0`
* `^0.2` is equivalent to `>=0.2.0 <0.3.0`
* `^0.0.3` is equivalent to `>=0.0.3 <0.0.4`
* `^0.0` is equivalent to `>=0.0.0 <0.1.0`
* `^0` is equivalent to `>=0.0.0 <1.0.0`

## URL Functions

Helm includes the [urlParse](#urlparse), [urlJoin](#urljoin), and
[urlquery](#urlquery) functions enabling you to work with URL parts.

### urlParse

Parses string for URL and produces dict with URL parts

```yaml
urlParse "http://admin:secret@server.com:8080/api?list=false#anchor"
```

上述结果为： a dict, containing URL object:

```yaml
scheme:   'http'
host:     'server.com:8080'
path:     '/api'
query:    'list=false'
opaque:   nil
fragment: 'anchor'
userinfo: 'admin:secret'
```

This is implemented used the URL packages from the Go standard library. For more
info, check https://golang.org/pkg/net/url/#URL

### urlJoin

Joins map (produced by `urlParse`) to produce URL string

```yaml
urlJoin (dict "fragment" "fragment" "host" "host:80" "path" "/path" "query" "query" "scheme" "http")
```

上述结果为： the following string:

```yaml
proto://host:80/path?query#fragment
```

### urlquery

Returns the escaped version of the value passed in as an argument so that it is
suitable for embedding in the query portion of a URL.

```yaml
$var := urlquery "string for query"
```

## UUID Functions

Helm can generate UUID v4 universally unique IDs.

```yaml
uuidv4
```

上述结果为： a new UUID of the v4 (randomly generated) type.

## Kubernetes and Chart Functions

Helm includes functions for working with Kubernetes including
[.Capabilities.APIVersions.Has](#capabilitiesapiversionshas),
[Files](#file-functions), and [lookup](#lookup）。

### lookup

`lookup` is used to look up resource in a running cluster. When used with the
`helm template` command it always returns an empty response.

You can find more detail in the [documentation on the lookup
function](functions_and_pipelines.md/#using-the-lookup-function）。

### .Capabilities.APIVersions.Has

Returns if an API version or resource is available in a cluster.

```yaml
.Capabilities.APIVersions.Has "apps/v1"
.Capabilities.APIVersions.Has "apps/v1/Deployment"
```

More information is available on the [built-in object
documentation](builtin_objects.md）。

### File Functions

There are several functions that enable you to get to non-special files within a
chart. For example, to access application configuration files. These are
documented in [Accessing Files Inside Templates](accessing_files.md）。

_Note, the documentation for many of these functions come from
[Sprig](https://github.com/Masterminds/sprig）。 Sprig is a template function
library available to Go applications._
