---
title: Temporary Files
author: strayRed
date: 2020-11-16 15:49:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

临时文件用于将数据写入磁盘，然后再将其移动到永久存储的位置或将其丢弃。 例如，当电影编辑器应用程序导出项目时，它可能会将每个帧写入一个临时文件，直到到达结尾为止，然后将完成的文件移动到〜/ Movies目录。 在这种情况下使用临时文件可确保完成任务过程的原子性（要么获得成品，要么一无所获），并且不会在系统上造成过多的内存压力（在大多数计算机上，磁盘空间是丰富的而内存是有限的）。

使用临时文件有四个不同的步骤：

1. 在文件系统中创建一个临时目录
2. 在该目录中创建一个具有唯一文件名的临时文件
3. 将数据写入临时文件
4. 完成后，移动或删除临时文件

# Creating a Temporary Directory

创建临时文件的第一步是找到一个合理的，偏僻的位置，您可以将其写入—在不起眼的地方，不会妨碍用户或不会被诸如 Spotlight索引，Time Machine备份或iCloud同步。

在Unix系统上，/ tmp目录是实际的暂存空间。 但是，当今的macOS和iOS应用程序在沙盒中运行，无法访问系统目录； 这样的硬编码路径不能被适用。

如果不打算保留临时文件，则可以使用 NSTemporaryDirectory() 函数来获取当前用户的临时目录的路径。

```Objective-C
NSURL *temporaryDirectoryURL = [NSURL fileURLWithPath: NSTemporaryDirectory() isDirectory: YES];
```


另外，如果最终您打算将临时文件移动到目标URL，则首选的方法（尽管更为复杂）是调用FileManager方法`url（for：in：appropriateFor：create :)`。

```Objective-C
NSURL *destinationURL = <#/path/to/destination#>;
NSFileManager *fileManager = [NSFileManager defaultManager];
NSError *error = nil;
NSURL *temporaryDirectoryURL =
    [fileManager URLForDirectory:NSItemReplacementDirectory
                        inDomain:NSUserDomainMask
               appropriateForURL:destinationURL
                          create:YES
                           error:&error];
```

- **URLForDirectory**：NSItemReplacementDirectory，与`NSDocumentDirectory` 一样是 `NSSearchPathDirectory` 枚举，表示用于创建一个临时目录。
- **inDomain**：使用`NSUserDomainMask` 表示当前用户的沙盒域名下，Domain一般都传入这个参数。
- **appropriateForURL**：指定的目标URL（最终保存文件的URL），传入这个参数可以让得到的临时目录能够在之后快速地跳转到这个URL。
- **create**：最终传入YES来创建这个临时目录。

# Creating a Temporary File

下一步就是弄清楚该怎么称呼我们的临时文件。 只要它是唯一的，我们对它的名字并不挑剔，并且不会干扰目录中的任何其他临时文件。生成唯一标识符的最佳方法是`ProcessInfo`的实例属性`globalUniqueString`。

```Objective-C
[[NSProcessInfo processInfo] globallyUniqueString];
//42BC63F7-E79E-4E41-8E0D-B72B049E9254-25121-000144AB9F08C9C1
```

使用`UUID`同样也不失为一个好方法。

```Objective-C
[[NSUUID UUID] UUIDString];
//B49C292E-573D-4F5B-A362-3F2291A786E7
```

有了唯一的文件名之后，就可以创建临时文件了。

```Objective-C
NSURL *destinationURL = <#/path/to/destination#>;

NSFileManager *fileManager = [NSFileManager defaultManager];
NSError *error = nil;
NSURL *temporaryDirectoryURL =
    [fileManager URLForDirectory:NSItemReplacementDirectory
                        inDomain:NSUserDomainMask
               appropriateForURL:destinationURL
                          create:YES
                           error:&error];

NSString *temporaryFilename =
    [[NSProcessInfo processInfo] globallyUniqueString];
NSURL *temporaryFileURL =
    [temporaryDirectoryURL
        URLByAppendingPathComponent:temporaryFilename];
```

# Writing to a Temporary File

只是创建文件URL的这一行为与文件系统无关。 真正的文件只有被写入的时候才会被创建。 

## Writing Data to a URL
将数据写入文件的最简单方法是调用Data的`write(to：options)`方法：

```Objective-C
NSData *data = <#some data#>;
NSError *error = nil;
[data writeToURL:temporaryFileURL
         options:NSDataWritingAtomic
           error:&error];
```

通过传递`NSDataWritingAtomic`选项，我们确保要么写入所有数据，要么该方法返回错误。

## Writing Data to a File Handle

如果要做的事情比将单个Data对象写入文件还要复杂，考虑创建FileHandle，如下所示：

```Objective-C
NSError *error = nil;
NSFileHandle *fileHandle = [NSFileHandle fileHandleForWritingToURL:temporaryFileURL error:&error];
[fileHandle writeData:data];
[fileHandle closeFile];
```

## Writing Data to an Output Stream

对于更高级的API，使用OutputStream定向数据流并不少见。 创建到临时文件的输出流与任何其他类型的文件都没有什么不同：

```Objective-C
[NSOutputStream outputStreamWithURL:temporaryFileURL
                                 append:YES];

[outputStream write:data.bytes
          maxLength:data.length];

[outputStream close];
```

# Moving or Deleting the Temporary File

操作系统会定期删除系统指定的临时目录中的文件。 因此，如果您打算保留已写入的文件，则需要将其移动到临时目录之外的某个位置。如果您已经知道文件的存放位置，则可以使用FileManager将其移至永久位置：

```Objective-C
NSFileManager *fileManager = [NSFileManager defaultManager];

NSURL *fileURL = <#/path/to/file#>;
NSError *error = nil;
[fileManager moveItemAtURL:temporaryFileURL
                     toURL:fileURL
                     error:&error];
```

删除临时文件同样也使用 FileManager

```Objective-C
NSFileManager *fileManager = [NSFileManager defaultManager];

NSError *error = nil;
[fileManager removeItemAtURL:temporaryFileURL
                       error:&error];
```

