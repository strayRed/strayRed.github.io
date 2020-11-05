---
title: Bundles and Packages
author: strayRed
date: 2020-11-05 17:19:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

尽管是不同的概念，术语 `bundle` 和 `package` 经常互换使用。部分原因无疑是由于它们的名称相似，但可能造成混淆的主要原因是许多 `bundle` 恰好是包（反之亦然）。

因此，在进一步讨论之前，让我们定义一下术语：

- `bundle`是一个具有已知结构的目录文件，其中包含可执行代码和代码使用的资源。

- `package`是一个在Finder中看起来像是单个文件的目录文件。

总而言之，`bundle`像是个背包（🎒），每一个都有特殊的口袋和隔间，可以携带任何你需要的东西，并且根据它是用来上学、工作还是健身房的不同配置而来。而`package`就像一个盒子(📦)，看起来是密封的，是一个单一个个体。如果某样东西既是`package`又是`bundle`，那它就像一件行李(🧳)：像盒子一样密封，像背包一样被组织成隔间。

# Bundles

`Bundles`主要是用来为开发者提供便利的，开发者不仅可以使用`Bundles`来结构化地管理部分代码和资源文件，它还支持本地化这样的系统层面上的特性。

`Bundles`主要分为以下几类，每一类都有其独特的结构和要求。

- **App Bundles**：Info.plist、app icons、launch images，以及其他可用于程序使用的资源文件，interface files, strings files, and data files。
- **Framework Bundles**：通常为第三方库的资源文件。
- **Loadable Bundles**：和插件类似，它包含可执行代码和扩展应用程序功能的资源。

# Accessing Bundle Contents：

对于 `App Bundle`，我们可以直接使用`Bundle.main`访问。

对于`dynamic Framework Bundle`，其目录是与 `App Bundle`独立的，我们可以通过动态库中的类来访问到`Bundle`。

```Objective-C
+ (NSBundle *)getBundle{
    NSURL    *url=  [[NSBundle bundleForClass:[SomeClass class]] URLForResource:BUNDLE_NAME withExtension:@"bundle"];
    NSBundle *bundle = [NSBundle bundleWithURL:url];
    return bundle;
}
```

对于`static Framework Bundle`和 `Loadable Bundles`，其`Bundle`是在`App Bundle`的目录之下，直接使用`Bundle`名进行访问：

```Swift
fileprivate let moduleName = "Rx"

public final class UnkindledRxAPI {
    //静态库的bundle存储在mainBundle下
    public static var bundle: Bundle {
        Bundle
        guard let bundle = Bundle.main.url(forResource: moduleName, withExtension: "bundle").flatMap ({ Bundle(url: $0) }) else { return Bundle.main }
        return bundle
    }
}
```

# Getting App Information

app信息都包含在`info.plist`里面，我们可以用`Main Bundle`来获取相关的信息。

```Swift
import Foundation

let bundle = Bundle.main

bundle.bundleURL        // "/path/to/Example.app"
bundle.bundleIdentifier // "com.nshipster.example"
```

可以使用`InfoDictionary`获取app的其他信息，`localizedInfoDictionary`可用于获取本地化信息。

```Swift
bundle.infoDictionary["CFBundleName"] // "Example"
bundle.localizedInfoDictionary["CFBundleName"] // "Esempio" (`it_IT` locale)
```

# Getting Localized Strings

在项目的`Project`的`Info`标签，设置`Localization`选项，选择本地化支持的语言。然后添加`xxx.strings`文件，选中`strings`文件然后再点击右侧的`Localization`，会根据本地化语言派生出其他同名的`strings`文件。

我们需要在多个`strings`设置同一个`key`对应的本地化字符串。

```
//Localizable.strings(English)
"Projects" = "Projects";
"Discover" = "Discover";
"Mine" = "Mine";
"Featured" = "Featured";
"Popular"  = "Popular";
"Latest"   = "Latest";
"Events"   = "Events";

//Localizable.strings(Chinese, Simplified)
"Projects" = "项目";
"Discover" = "发现";
"Mine" = "我的";
"Featured" = "推荐";
"Popular"  = "热门";
"Latest"   = "最近更新";
"Events"   = "动态";
```

使用只需要调用`NSLocalizedString()`方法。

```Swift
extension String {
    var localized: String {
        return NSLocalizedString(self, comment: "")
    }
}

print("Projects".localized) // Projects or 项目
```

# Packages

`Package`主要用于通过将相关资源封装和整合到一个单元来改善用户体验。

如果一个目录文件有以下的特点则称之为 `Package`：

- 有一个特殊的扩展名，.app`, `.playground`, or `.plugin
- 有一个某一app已经事先注册为其文件类型的扩展名
- 有一个扩展属性，将其指定为package *(The directory has an extended attribute designating it as a package *)

# Accessing the Contents of a Package

我们同样可以在程序中打开一个`Package`。

- 如果一个`Package`有`Bundle`结构，通常最容易使用`NSBundle`来载入。
- 如果它是一个文档，则可以在macOS上使用NSDocument，在iOS上使用UIDocument来打开。
- 否则，可以使用`FileWrapper`来遍历目录，`FileHandler`来读写文件。

# Determining if a Directory is a Package

[`isPackageKey`](https://developer.apple.com/documentation/foundation/urlresourcekey/1413867-ispackagekey)可以用于判断某一个目录是否是`Package`或是其他app的注册扩展文件类型。

```Swift
let url: URL = …
let directoryIsPackage = (try? url.resourceValues(forKeys: [.isPackageKey]).isPackage) ?? false
```

如果没有确定的`url`，同时又希望检查特殊的扩展名，可以使用`Core Services`的`API`来完成这个需求：

```Swift
import Foundation
import CoreServices

func filenameExtensionIsPackage(_ filenameExtension: String) -> Bool {
    guard let uti = UTTypeCreatePreferredIdentifierForTag(
                        kUTTagClassFilenameExtension,
                        filenameExtension as NSString, nil
                    )?.takeRetainedValue()
    else {
        return false
    }

    return UTTypeConformsTo(uti, kUTTypePackage)
}

let xcode = URL(fileURLWithPath: "/Applications/Xcode.app")
let appExtension = xcode.pathExtension // "app"
filenameExtensionIsPackage(appExtension) // true
```

