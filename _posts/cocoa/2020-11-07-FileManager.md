---
title: FileManager
author: strayRed
date: 2020-11-07 13:37:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

`FileManager`提供了一系列便利的方法来创建、读取、移动、复制和删除文件和目录，不论是本地还是 iClond，我们可以通过传入 URL 或者文件路径(Path)来进行这些操作。

# Paths and File URLs

文件系统上的对象可以用几种不同的方法标识。例如，以下每一项都表示同一文本文档的位置：

- Path: `/Users/NSHipster/Documents/article.md`
- File URL: `file:///Users/NSHipster/Documents/article.md`
- File Reference URL: `file:///.file/id=1234567.7654321/`

路径是斜杠（/）分割字符串，用于指定目录层次结构中的位置。文件URL是除了文件路径外，还具有`File://` scheme 的 URL。`File Reference URL`使用独立于任何目录结构的唯一标识符标识文件的位置。

其中，我们主要使用前种路径方式，它们都使用关系路径标示文件或者目录，该路径可能是绝对路径，提供了根目录中资源的完整位置，也可能是相对路径，显示了如何从给定的起点获取资源。绝对URL以`/`，而相对URL以`./`（当前目录）、`../`（父目录）或`~`（当前用户的主目录）开头。

`FileManager`有接受者两类参数的方法（URL或者String），一般来说，使用URL比使用路径更为方便，因为使用URL更灵活。（从URL到路径的转换也比从URL到路径更容易）。

# Locating Files and Directories

不同平台的标准位置各不相同，因此不需要手动构建/System或~/Documents之类的路径，而是使用FileManager方法`url(for:in:appropriateFor:create:)`(`NSSearchPathForDirectoriesInDomains(...)`)找到适合需要的位置。

第一个参数接受由指定的值之一 [`FileManager.SearchPathDirectory`](https://developer.apple.com/documentation/foundation/filemanager/searchpathdirectory)（OC为常量`NSDocumentDirectory`，`NSCacheDirectory`）. 它们决定了您要查找的标准目录类型，如“Directory”或“Cache”。

第二个参数传递[`FileManager.SearchPathDomainMask`](https://developer.apple.com/documentation/foundation/filemanager/searchpathdomainmask)决定了你要寻找的范围。例如，`.applicationDirectory`可能主目录下的的/Applications，或者是用户目录下的~/Applications。

```Swift
let documentsDirectoryURL =
    try FileManager.default.url(for: .documentDirectory,
                            in: .userDomainMask,
                            appropriateFor: nil,
                            create: false)
```

```Objective-C
NSFileManager *fileManager = [NSFileManager defaultManager];
NSString *documentsPath =
    [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
NSString *filePath = [documentsPath stringByAppendingPathComponent:@"file.txt"];
```

> 文件和目录的名称可以与路径编码的名称不同。
>
> 例如，大多数macOS系统目录都是本地化的；虽然用户的照片位于~/photos，但是~/photos/.localized文件可以更改文件夹在Finder和Open/Save面板中的命名方式。应用程序还可以为其自身及其创建的目录提供特定于区域设置的名称。
>
> 总的来说，当向用户显示文件或目录的名称时，不要简单地使用最后一个路径component。而是调用方法displayName（atPath:）：

```Swift
let directoryURL: URL = /path/to/directory

// Bad
let filename = directoryURL.pathComponents.last

// Good
let filename = FileManager.default.displayName(atPath: url.path)
```

# Determining Whether a File Exists

使用`fileExists(atPath:)`检查文件是否存在

```Objective-C
NSURL *fileURL = <#/path/to/file#>;
NSFileManager *fileManager = [NSFileManager defaultManager];
BOOL fileExists = [fileManager fileExistsAtPath:[fileURL path]];
```

# Getting Information About a File

使用`attributesOfItem(atPath:)`获取文件属性，结果是一个字典类型，使用`FileAttributeKey`来获取对应的属性。

[see everything that’s available](https://developer.apple.com/documentation/foundation/fileattributekey).

```Objective-C
NSURL *fileURL = <#/path/to/file#>;
NSFileManager *fileManager = [NSFileManager defaultManager];

NSError *error = nil;
NSDictionary *attributes = [fileManager attributesOfItemAtPath:[fileURL path]
                                                         error:&error];
NSDate *creationDate = attributes[NSFileCreationDate];
```

# Listing Files in a Directory

使用``contentsOfDirectory(at:includingPropertiesForKeys:options:)`获取目录下的所有文件，options可以跳过隐藏文件（SkipsHiddenFiles，也是其唯一的选项），keys则是上部分提到的文件属性`key`，需要获取的属性可以在这里传入对应的`key`，查询结果会包含在返回的URL中。

```Objective-C
NSFileManager *fileManager = [NSFileManager defaultManager];
NSURL *bundleURL = [[NSBundle mainBundle] bundleURL];
NSArray *contents = [fileManager contentsOfDirectoryAtURL:bundleURL
                               includingPropertiesForKeys:@[]
                                                  options:NSDirectoryEnumerationSkipsHiddenFiles
                                                    error:nil];

NSPredicate *predicate = [NSPredicate predicateWithFormat:@"pathExtension == 'png'"];
for (NSURL *fileURL in [contents filteredArrayUsingPredicate:predicate]) {
    // Enumerate each .png file in directory
}
```

# Recursively Enumerating Files In A Directory

如果需要获取一个目录下包括其子目录的所有文件，Swift可以使用`FileManager.DirectoryEnumerator`类型，使用`enumerator(atPath:)`创建。

```Swift
let directoryURL: URL = /path/to/directory

if let enumerator =
    FileManager.default.enumerator(atPath: directoryURL.path)
{
    for case let path as String in enumerator {
        // Skip entries with '_' prefix, for example
        if path.hasPrefix("_") {
            enumerator.skipDescendants()
        }
    }
}
```

```Objective-C
NSFileManager *fileManager = [NSFileManager defaultManager];
NSURL *bundleURL = [[NSBundle mainBundle] bundleURL];
//OC则是获取一个NSDirectoryEnumerator实例
NSDirectoryEnumerator *enumerator = [fileManager enumeratorAtURL:bundleURL
                                      includingPropertiesForKeys:@[NSURLNameKey, NSURLIsDirectoryKey]
                                                         options:NSDirectoryEnumerationSkipsHiddenFiles
                                                    errorHandler:^BOOL(NSURL *url, NSError *error)
{
    if (error) {
        NSLog(@"[Error] %@ (%@)", error, url);
        return NO;
    }

    return YES;
}];
//新建一个数组获取结果
NSMutableArray *mutableFileURLs = [NSMutableArray array];
for (NSURL *fileURL in enumerator) {
    NSString *filename;
  //使用指针获取值
    [fileURL getResourceValue:&filename forKey:NSURLNameKey error:nil];
    NSNumber *isDirectory;
  //使用指针获取值
    [fileURL getResourceValue:&isDirectory forKey:NSURLIsDirectoryKey error:nil];
    // Skip directories with '_' prefix, for example
    if ([filename hasPrefix:@"_"] && [isDirectory boolValue]) {
        [enumerator skipDescendants];
        continue;
    }
    if (![isDirectory boolValue]) {
        [mutableFileURLs addObject:fileURL];
    }
}
```

# Deleting a File or Directory

```Objective-C
NSFileManager *fileManager = [NSFileManager defaultManager];
NSString *documentsPath =
    [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,
                                         NSUserDomainMask,
                                         YES) firstObject];
NSString *filePath = [documentsPath stringByAppendingPathComponent:@"image.png"];
NSError *error = nil;

if (![fileManager removeItemAtPath:filePath error:&error]) {
    NSLog(@"[Error] %@ (%@)", error, filePath);
}
```

# FileManagerDelegate

FileManager可以选择设置一个代理来验证它是否应该执行特定的文件操作。这是审核应用程序中所有文件操作的方便方法，也是分离和集中业务逻辑（例如要保护哪些文件不被删除）的好地方。

`FileManagerDelegate`可以允许和禁止4种类操作，moving, copying, removing, and linking，每一种操作都有用于处理路径和URL的不同方法，以及在发生错误后如何继续操作的方法。

如果我们需要使用 `FileManagerDelegate`，那么我们就需要创建自己的`FileManager`而不是使用系统给出的单例。

```Objective-C
NSFileManager *fileManager = [[NSFileManager alloc] init];
fileManager.delegate = delegate;

NSURL *bundleURL = [[NSBundle mainBundle] bundleURL];
NSArray *contents = [fileManager contentsOfDirectoryAtURL:bundleURL
                               includingPropertiesForKeys:@[]
                                                  options:NSDirectoryEnumerationSkipsHiddenFiles
                                                    error:nil];

for (NSString *filePath in contents) {
    [fileManager removeItemAtPath:filePath error:nil];
}

// CustomFileManagerDelegate.m

#pragma mark - NSFileManagerDelegate

- (BOOL)fileManager:(NSFileManager *)fileManager
shouldRemoveItemAtURL:(NSURL *)URL
{
    return ![[[URL lastPathComponent] pathExtension] isEqualToString:@"pdf"];
}
```