---
title: Search Kit
author: strayRed
date: 2020-11-16 13:03:00 +0800
categories: [mac os, cocoa]
tags: [mac os, cocoa]
---

Search Kit是用于以人类语言搜索和索引内容的C框架。 它支持短语或部分词的匹配，包括逻辑（AND，OR）和通配符（*）运算符，并且可以按相关性对结果进行排名。 Search Kit也提供了文档总结功能，在生成有代表性的摘要时很有用。 最重要的是：它是线程安全的。

取得极大成功的即打即搜的特性底层都是 Search Kit，从 OS X 的邮件，Xcode 到系统偏好和 Spotlight。

>  [Apple’s Search Kit Programming Guide](https://developer.apple.com/library/mac/#documentation/UserExperience/Conceptual/SearchKitConcepts/searchKit_intro/searchKit_intro.html)

# Search 101

## Extract

首先，内容必须从 [语料库(corpus)](https://zh.wikipedia.org/wiki/语料库) 中提取出来。对一个文本文档而言，这会涉及到移除样式，格式或其他元信息。对于数据记录（例如NSManagedObject），这意味着将所有主要的字段合并为一种表示形式。一旦抽取完毕，内容将被 [符号化(tokenized)](https://en.wikipedia.org/wiki/Tokenization) 为后续过程作准备。

## Filter

为了得到最相关的匹配，过滤掉常见的，对整体意思表达没有帮助的“停止”词汇是十分重要的，如冠词，代词和助动词。

## Reduce

在同一行中，表达同一事物的词语应当简化为一个共同的形式。词素组，如语法上的动词变化，举个例子像 “computer”，”computers”，”computed”，”computing” 都可以用 [词干提取(stemmer)](https://zh.wikipedia.org/wiki/词干提取) 简化为 “compute”。同样地，同义词可以用同义词表归并为统一的条目。

## Index

将上述过程后得到的结果为一个 arrray of normalized tokens，用其生成一个 [inverted index](https://en.wikipedia.org/wiki/Inverted_index)，这样每一个 token 在这个索引中都会对应其来源位置。

对语料库中的每一份文档或记录重复这一过程之后，每个 token 可以指向许多不同的文章。在搜索过程中，一个查询映射到一个或多个 token ，检索出这些 token 对应的文章的并集。

# Using Search Kit

## Creating an Index

`SKIndexRef` 是 Search Kit 的核心数据类型，它包含所有所需要的信息，用来处理及实现搜索，以及为新文档添加信息。索引可以是持久的 / 基于文件，或暂时的 / 内存中。索引既可以从头开始创建，也可以从已有的文件或数据对象加载，一旦索引用完了，就像许多其它 C 接口一样，索引就关闭了。


创建一个新的存储在内存中索引时，使用空的NSMutableData实例作为数据存储：

```Objective-C
NSMutableData *mutableData = [NSMutableData data];
SKIndexRef index = SKIndexCreateWithMutableData((__bridge CFMutableDataRef)mutableData, NULL, kSKIndexInverted, NULL);
```

如需要从本地磁盘创建索引，则使用下面方法：

```C
SKIndexRef SKIndexCreateWithURL(CFURLRef inURL, CFStringRef inIndexName, SKIndexType inIndexType, CFDictionaryRef inAnalysisProperties);
```



## Adding Documents to an Index

`SKDocumentRef` 是与索引中的条目相对应的数据类型。当一次搜索执行完毕，documents（包括它们的内容和相关度）就是结果。

每个 `SKDocumentRef` 都关联着一个 URI。

对于文件系统上的文档，URI 就是文件在磁盘上的路径：

```Objective-C
NSURL *fileURL = [NSURL fileURLWithPath:@"/path/to/document"];
SKDocumentRef document = SKDocumentCreateWithURL((__bridge CFURLRef)fileURL);
```

对于 Core Data 管理的对象，可以用 `NSManagedObjectID -URIRepresentation`：

```Objective-C
NSURL *objectURL = [objectID URIRepresentation];
SKDocumentRef document = SKDocumentCreateWithURL((__bridge CFURLRef)objectURL);
```

> 对于任何其它数据，由开发者自己定义 URI 表示。

当向 `SKIndexRef` 添加 `SKDocumentRef` 的内容时，文本既可以手动指定：

```Objective-C
NSString *string = @"Lorem ipsum dolar sit amet"
SKIndexAddDocumentWithText(index, document, (__bridge CFStringRef)string, true);
```

…也可以从文件自动采集：

```Objective-C
NSString *mimeTypeHint = @"text/rtf"
SKIndexAddDocument(index, document, (__bridge CFStringRef)mimeTypeHint, true);
```

为了改变基于文件的文档内容的处理方式，在创建索引时可以定义一些属性：

```Objective-C
NSSet *stopwords = [NSSet setWithObjects:@"all", @"and", @"its", @"it's", @"the", nil];

NSDictionary *properties = @{
  @"kSKStartTermChars": @"", // additional starting-characters for terms
  @"kSKTermChars": @"-_@.'", // additional characters within terms
  @"kSKEndTermChars": @"",   // additional ending-characters for terms
  @"kSKMinTermLength": @(3),
  @"kSKStopWords":stopwords
};

SKIndexRef index = SKIndexCreateWithURL((CFURLRef)url, NULL, kSKIndexInverted, (CFDictionaryRef)properties);
```

## Searching

`SKSearchRef` 是在 `SKIndexRef` 上执行搜索时构建的数据类型。它包含索引的引用，查询字符串，和一些选项：

```Objective-C
NSString *query = @"kind of blue";
SKSearchOptions options = kSKSearchOptionDefault;
SKSearchRef search = SKSearchCreate(index, (CFStringRef)query, options);
```

`SKSearchOptions` 是有以下可能值的位掩码：

>
> kSKSearchOptionDefault：默认搜索选项包括：
>
> - 计算相关度的值
> - 查询中的空格解释为逻辑**与**操作
> - 不使用相似搜索

这些选项也可以单独指定：

> - `kSKSearchOptionNoRelevanceScores`：这个选项不计算相关度值可以节省搜索时间。
> - `kSKSearchOptionSpaceMeansOR`：这个选项将查询语句中的空格改为逻辑**或**操作。
> - `kSKSearchOptionFindSimilar`：这个选项使 Search Kit 返回与示例文本相似的文档引用。当这个选项指定时，Search Kit 忽略所有查询操作符。

将所有这些放到一起的是 `SKIndexCopyDocumentURLsForDocumentIDs`，它执行搜索然后用结果填充数组。在匹配的范围中遍历可以访问文档的 URL 和相关度值（如果计算了的话）：

```Objective-C
NSUInteger limit = ...; // Maximum number of results
NSTimeInterval time = ...; // Maximum time to get results, in seconds
SKDocumentID documentIDs[limit];
CFURLRef urls[limit];
float scores[limit];
CFIndex count;
Boolean hasResult = SKSearchFindMatches(search, limit, documentIDs, scores, time, &count);

SKIndexCopyDocumentURLsForDocumentIDs(index, foundCount, documentIDs, urls);

NSMutableArray *mutableResults = [NSMutableArray array];
[[NSIndexSet indexSetWithIndexesInRange:NSMakeRange(0, count)] enumerateIndexesUsingBlock:^(NSUInteger idx, BOOL *stop) {
    CFURLRef url = urls[idx];
    float relevance = scores[idx];

    NSLog(@"- %@: %f", url, relevance);

    if (objectID) {
      [mutableResults addObject:(NSURL *)url];
    }

    CFRelease(url);
}];
```