---
title: iOS Sand Box
author: strayRed
date: 2020-11-16 15:49:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

沙盒也叫沙箱，英文sandbox，其原理是通过重定向技术，把程序生成和修改的文件定向到自身文件夹中。应用程序中所有的非代码文件都保存在沙盒中，比如图片、声音、属性列表，sqlite数据库和文本文件等。安全第一，在沙盒机制下，iOS程序之间的文件夹禁止互相访问。

# Documents

此目录一般保存应用程序本身产生的文件数据，例如游戏进度，绘图软件的绘图等， iTunes备份和恢复的时候，会包括此目录。

注意：此目录下不要保存网络上下载的文件，否则app无法上架！

# Library

## Library/Caches

此目录用来保存应用程序运行时生成的需要持久化的数据，这些数据一般存储体积比较大，又不是十分重要，比如网络请求数据等。这些数据需要用户负责删除。iTunes同步设备时不会备份该目录。

## Library/Preferences

此目录保存应用程序的所有偏好设置，iOS的Settings(设置)应用会在该目录中查找应用的设置信息。iTunes同步设备时会备份该目录。

注意：该文件夹下不能直接创建偏好设置文件，而是应该使用NSUserDefaults类来取得和设置应用程序的偏好。

## tmp

此目录保存应用程序运行时所需的临时数据，使用完毕后再将相应的文件从该目录删除。应用没有运行时，系统也可能会清除该目录下的文件。iTunes同步设备时不会备份该目录。

# 获取沙盒路径

`NSSearchPathForDirectoriesInDomains(NSSearchPathDirectory directory, NSSearchPathDomainMask domainMask, BOOL expandTilde)`  用来获取沙盒路径，此函数需要三个参数：

- **directory**： 表示需要查找的文件夹类型，枚举值
- **domainMask**：表示在用户的主目录中查找，枚举值，一般传 NSUserDomainMask
- **expandTilde**：表示返回的沙盒路径是否展开，布尔值，一般传 YES

## 获取沙盒根路径

`NSHomeDirectory()`

## Documents

```Objective-C
NSString *path = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).lastObject;
```

## Library

```Objective-C
NSString *path = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES).lastObject;
```

### Library/Caches

```Objective-C
NSString *path = NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES).lastObject;
```

### Library/Preferences

```Objective-C
NSString *path = NSSearchPathForDirectoriesInDomains(NSPreferencePanesDirectory, NSUserDomainMask, YES).lastObject;
//此方法获取路径是沙盒/Library/PreferencePanes，但并不存在这样的路径，想要访问Preferences文件夹，需要拼接路径
NSString *path = [NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES).lastObject stringByAppendingPathComponent:@"Preferences"];
```

### tmp

```Objective-C
NSString *path = NSTemporaryDirectory();
```

