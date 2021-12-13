---
title: 深入Go：Internationalization-国际化
tags: Go
key: go-internationalization
---

当服务需要应对多语言场景时，我们应该如何组织代码？
<!--more-->

###### 说明

本文不探讨诸如单复数变换等复杂情况，如有需要，请参见[这里](https://phrase.com/blog/posts/internationalization-i18n-go/)。

###### （太长不看版）

1. 读取语言标签与相应翻译，进行翻译字符串的注册
2. 获取时根据语言解析标签并获得相应语言的Printer，根据Key进行翻译字符串的查找与生成

示例代码如下：
```go
package main

import (
  "golang.org/x/text/language"
  "golang.org/x/text/message"
)

// 此处`und`为undefined，用于语言无对应的翻译时的显示
// 请根据翻译文件准备好相应的key与翻译字符串
var msg = map[string]string{"zh": "你好，%s", "en": "Hello, %s", "und": "Hello, %s"}
var key = "HelloString" // key是字典用于查找翻译字符串的键；当然这里也可以使用"Hello, %s"

func register(k string, d map[string]string) {
  for lang, translation := range d {
    // 根据语言字符串解析语言标签
    tag, err := language.Parse(lang)
    if err != nil {
      // 语言注册建议在程序初始化阶段完成，此时出错可能直接panic较好
      panic(err)
    }
    // 根据语言tag、key与翻译字符串进行设置
    err = message.SetString(tag, key, translation)
    if err != nil {
      panic(err)
    }
  }
}

func getTranslation(k, lang string, content []interface{}) string {
  tag, err := language.Parse(lang)
  if err != nil {
    // 指定解析出错时希望返回的语言
    tag = language.Und
  }
  // 根据语言获取Printer
  p := message.NewPrinter(tag)
  return p.Sprintf(k, content...)
}

func main() {
  register(key, msg)
  println(getTranslation(key, "en", []interface{}{"en"}))
  println(getTranslation(key, "en-US", []interface{}{"en-US"}))
  println(getTranslation(key, "zh", []interface{}{"zh"}))
  // zh-CN 和 zh-SG 的 parent 都是 zh，因此会根据 zh 进行返回
  println(getTranslation(key, "zh-CN", []interface{}{"zh-CN"}))
  println(getTranslation(key, "zh-SG", []interface{}{"zh-SG"}))
  // zh-TW 的 parent 并不是 zh，因此会根据 und tag 返回相应的翻译
  println(getTranslation(key, "zh-TW", []interface{}{"zh-TW"}))
  // 解析语言出错时的处理
  println(getTranslation(key, "???", []interface{}{"???"}))
}
/* 输出：
Hello, en
Hello, en-US
你好，zh
你好，zh-CN
你好，zh-SG
Hello, zh-TW
Hello, ???
*/
```

###### （太长不看版结束）

## Prerequisites: 原理

### 语言标签

我们通常可以在HTTP header里看见类似于`Accept-Language: zh-CN`或是`Accept-Language: en`之类的值，这里header里对应的值就是**语言标签**。类似地，`zh-cmn-Hans-CN`也是语言标签。

#### 语言标签的语法

我们需要关注的语言标签的语法：

`主语言子标签`-`扩展语言子标签`-`文字子标签`-`地区子标签`

`zh-cmn-Hans-CN`

+ 除了主语言子标签是必填，其他都是可选；
+ 扩展语言子标签为3字母，最多可有三个；
+ 文字子标签为4字母（首字母大写），最多一个；
+ 地区子标签为2字母（通常大写）

因此，`zh-cmn-Hans-CN`被解读为：

`汉语(zhongwen)-普通话(simplified mandarin)-简体(Han Simplified)-中国大陆`

另外，该语言标签在2009年后就不再被推荐使用了，因为扩展语言标签`cmn`蕴含该语言是`zh`（汉语）。当然，目前**为了兼容，建议使用`zh`而非`cmn`**。

### Go存储翻译字符串

Go通过调用`message.SetString(tag language.Tag, key string, msg string)`来存储翻译字符串；其中，`tag`为语言标签，`key`为该字符串的键，`msg`为该字符串在该语言下的值。例如

```go
// tag: en
message.SetString(language.English, "HelloString", "Hello, %s")
// tag: zh-Hans
message.SetString(language.SimplifiedChinese, "HelloString", "你好，%s") 
// tag: und
message.SetString(language.Und, "HelloString", "Hi, %s")
```

### Go获取翻译字符串

Go通过调用`message.NewPrinter(tag language.Tag)`来获取翻译字符串字典，通过调用`printer.Sprintf(key string, content ...interface{})`（或其他类似于`fmt`中的方法）来生成（或打印）翻译字符串。下文我们统一使用`Sprintf`作为示例。

打印的时候，使用key并根据语言标签查找相应的字典，如果在该语言标签中找不到该key，则依次在其祖先节点中继续查找；如果找到根节点（`und`）仍未找到，则效果同直接调用`fmt.Sprintf`相同。因此，在简单场景下，**建议直接在`en`、`zh`和`und`下增加翻译语句**；`und`用于处理无法解析的语言标签（例如，`???`）或意料之外的语言标签（例如，`zh-TW`的祖先节点依次为`zh-Hant`、`und`）。

例如

```go
message.NewPrinter(language.English, "en") // Hello, en
message.NewPrinter(language.AmericanEnglish, "en-US") //* Hello, en-US
message.NewPrinter(language.Chinese, "zh") //** Hi, zh
message.NewPrinter(language.SimplifiedChinese, "zh-Hans") // 你好，zh-Hans
/*
*   这里，因为en-US的父节点为en，因此可以找到对应翻译字符串
**  这里，因为zh的父节点是und，因此找到了und内的翻译字符串（zh-Hans是zh的子节点）
*/
```

## 实践

### Step 1: 准备字典

建议将各语言的翻译字符串与语言标签准备在单独的文本文件里，通过读取文件、解析标签与字符串，调用`message.SetString`设置。

### Step 2: 根据传入的语言标签获取翻译字符串

根据语言标签调用`language.Parse`获取tag，使用该tag获取`message.Printer`，使用该printer根据key调用`printer.Sprintf()`生成翻译字符串。

#### gRPC Gateway获取请求中的语言标签

gRPC Gateway会将HTTP headers存储在context中，调用`metadata.FromIncomingContext(ctx context.Context)`可获取到。

注意，获取到的是`map[string][]string`的结构，map的键为HTTP headers的各名称，因此可能是全小写（`accept-language`）也可能是首字母大写（`Accept-Language`），因此需要使用`strings.EqualFold(k, "accept-language")`来忽略大小写地比较。
