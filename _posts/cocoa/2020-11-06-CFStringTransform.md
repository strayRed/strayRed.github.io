---
title: CFStringTransform
author: strayRed
date: 2020-11-06 15:03:00 +0800
categories: [ios, cocoa, core foundation]
tags: [ios, cocoa, core foundation]
---

`CFStringTransform` 是 `Core Foundation` 中的一部分。这个函数传入以下参数，并返回一个 `Boolean` 来表示转换是否成功：

- `string`: 需要转换的字符串。由于这个参数是 `CFMutableStringRef` 类型，一个 `NSMutableString` 类型也可以通过自由桥接的方式传入。
- `range`: 转换操作作用的范围。这个参数是 `CFRange`，而不是 `NSRange`。
- `transform`: 需要应用的变换。这个参数使用了包含下面将提到的字符串常量的 [ICU transform string](http://userguide.icu-project.org/transforms/general)。
- `reverse`: 如有需要，是否返回反转过的变换。

`CFStringTransform` 中的 `transform` 参数涉及的内容很多。这里有个它能做什么的概述：

# Strip Accents and Diacritics

Énġlišh långuãge lẳcks iñterêßţing diaçrïtičş. 如此类的字符串，把扩展的拉丁字符集正则化为 ASCII 的表示，它非常有用。用 `kCFStringTransformStripCombiningMarks` 变换来去掉任意字符串中弯弯扭扭的符号。

# Name Unicode Characters

`kCFStringTransformToUnicodeName` 让你可以找出特殊字符的 Unicode 标准名，包括 Emoji。例如：”🐑💨✨” 被转换成 “{SHEEP} {DASH SYMBOL} {SPARKLES}”，而 “🐷” 变成了 “{PIG FACE}”。

# Transliterate Between Orthographies

`CFStringTransform` 可以在拉丁语和阿拉伯语、西里尔语、希腊语、韩语（韩国）、希伯来语、日语（平假名和片假名）、普通话、泰语的文字和拼音之间转写。

|            Transformation            |     Input      |   Output   |
| :----------------------------------: | :------------: | :--------: |
|   `kCFStringTransformLatinArabic`    |     mrḥbạ      |   مرحبا    |
|  `kCFStringTransformLatinCyrillic`   |     privet     |   привет   |
|    `kCFStringTransformLatinGreek`    |    geiá sou    |  γειά σου  |
|   `kCFStringTransformLatinHangul`    | annyeonghaseyo | 안녕하세요 |
|   `kCFStringTransformLatinHebrew`    |      şlwm      |    שלום    |
|  `kCFStringTransformLatinHiragana`   |    hiragana    |  ひらがな  |
|  `kCFStringTransformLatinKatakana`   |    katakana    |  カタカナ  |
|    `kCFStringTransformLatinThai`     |     s̄wạs̄dī     |    สวัสดี    |
| `kCFStringTransformHiraganaKatakana` |    にほんご    |  ニホンゴ  |
|  `kCFStringTransformMandarinLatin`   |      中文      | zhōng wén  |

当然我们也可以直接使用`kCFStringTransformToLatin`来将所有非英文文本转换为拉丁字母。
>
> 并且这只是用了核心类库中常量定义！直接传入一个[ICU transform](http://userguide.icu-project.org/transforms/general#TOC-ICU-Transliterators)表达式，`CFStringTransform` 还可以在拉丁语和阿拉伯语、亚美尼亚语、注音、西里尔字母、格鲁吉亚语、希腊语、汉语、韩语、希伯来语、平假名、印度语（梵文，古吉拉特语，旁遮普文，卡纳达语，马拉雅拉姆语，奥里雅语，泰米尔语，特卢固）、朝鲜语、片假名、叙利亚语、塔纳文、泰语之间转写。

# Normalize User-Generated Content

字符串变换的一个更实际的应用是正则化（ASCII字母）不可预知的用户输入。即使你的应用并不单独处理其他语言，你也应当能智能地处理用户向你的应用输入的任何内容。

例如，你想在设备上建立一个可搜索的电影索引，它包含世界各地的人的问候：

- 首先，应用 `kCFStringTransformToLatin` 变换将所有非英文文本转换为拉丁字母表示。

>Hello! こんにちは! สวัสดี! مرحبا! 您好! → Hello! kon’nichiha! s̄wạs̄dī! mrḥbạ! nín hǎo!

- 然后，应用 `kCFStringTransformStripCombiningMarks` 变换来去除变音符和重音。

> Hello! kon’nichiha! s̄wạs̄dī! mrḥbạ! nín hǎo! → Hello! kon’nichiha! swasdi! mrhba! nin hao!

- 最后，用 `CFStringLowercase` 转为小写，并用[`CFStringTokenizer`](https://developer.apple.com/library/mac/#documentation/CoreFoundation/Reference/CFStringTokenizerRef/Reference/reference.html) 分词用作文本的索引。

> (hello, kon’nichiha, swasdi, mrhba, nin, hao)

通过对用户输入的文本使用同样的变换，你就可以实现一个通用的搜索，无论搜索文本或内容是什么语言！