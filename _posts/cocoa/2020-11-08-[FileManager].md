title: FileManager
author: strayRed
date: 2020-11-08 13:37:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]

`FileManager`提供了一系列便利的方法来创建、读取、移动、复制和删除文件和目录，不论是本地还是 iClond，我们可以通过传入 URL 或者文件路径(Path)来进行这些操作。

# Paths and File URLs

文件系统上的对象可以用几种不同的方法标识。例如，以下每一项都表示同一文本文档的位置：

- Path: `/Users/NSHipster/Documents/article.md`
- File URL: `file:///Users/NSHipster/Documents/article.md`
- File Reference URL: `file:///.file/id=1234567.7654321/`

路径是斜杠（/）分割字符串，用于指定目录层次结构中的位置。文件URL是除了文件路径外，还具有`File://` scheme 的 URL。`File Reference URL`使用独立于任何目录结构的唯一标识符标识文件的位置。

其中，我们主要使用前种路径方式，它们都使用关系路径标示文件或者目录，