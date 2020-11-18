---
title: Dark Mode on iOS 13
author: strayRed
date: 2020-11-18 15:06:00 +0800
categories: [ios, cocoa, uikit]
tags: [ios, cocoa, uikit]
---

# Cancel Color Literals

在 Xcode 中，我们可以使用 #colorLiteral 使用字面量来表示颜色。color literals 和 image literals 一样都是为了支持Xcode Playgrounds而引入的。如果你的应用程序需要支持夜间模式，请不要使用 color literals。

# Nix UIColor Hexadecimal Initializers

与上一个section类似，不推荐使用以下方式初始化颜色：

```Swift
import SwiftySwiftColorSwift
let orange = UIColor(hex: "#FB8C00") // 👎
```

# Find & Replace Fixed Colors

UIColor定义了几个类属性，这些类属性通过其通用名称返回颜色。 这些属性在iOS 13中存在问题，因为它们不会自动调整为浅色或深色外观。 例如，在默认的UITableViewCell背景下，将标签的颜色设置为.black看起来不错，但是当启用“夜间模式”时，如果背景变为黑色，则无法辨认。如果应用支持夜间模式，需要替换掉以下属性：`red`，`orange`，`yellow`，`brown`，`green`，`green`，`cyan`，`blue`，`purple`，`magenta`，`lightGray`，`gray`，`darkGray`.

在任何情况下，支持深色模式的最简单的更改就是将下面提到的任何固定颜色属性替换为系统对应的系统可调整的颜色。即是前缀包含了system的颜色，如`systemRed`。

> 不存在 systemWhite和systemBlack，因为黑白没有自适应颜色，因为它们的名称在夜间模式下将不再具有描述性； 如果存在.systemBlack，则几乎必须是.white才能在夜间模式中显示。

# Use Semantic Colors

确保在任何外观模式和在任何设备上呈现一致UI的最佳方法是使用 Semantic Colors，即根据其**功能**而不是外观来命名。与动态字体使用“headline”和“body”之类的语义标记来自动地为用户显示最合适的字体类似，Semantic Colors（或Apple所谓的UI Element Colors），为你的视图和控件提供了一个更加泛用的颜色设置方式。

|             Type              |                             Name                             |
| :---------------------------: | :----------------------------------------------------------: |
|       **Label Colors**        | `.label`，`.secondaryLabel`，`.tertiaryLabel`，`.quaternaryLabel` |
|        **Text Colors**        |                      `.placeholderText`                      |
|        **Link Colors**        |                           `.link`                            |
|     **Separator Colors**      |              `.separator`， `.opaqueSeparator`               |
|        **Fill Colors**        | `.systemFill`，`.secondarySystemFill`， `.tertiarySystemFill`， `.quaternarySystemFill` |
|     **Background Colors**     | `.systemBackground`，`.secondarySystemBackground`，`.tertiarySystemBackground` |
| **Grouped Background Colors** | `.systemGroupedBackground`，`.secondarySystemGroupedBackground`，`.tertiarySystemGroupedBackground` |

# Upgrade Homegrown Semantic Color Palettes

如果您对应用程序中的颜色管理有所考虑，那么您可能会采用以下某种形式的策略，即根据命名空间或UIColor扩展中的固定颜色定义语义颜色。例如，以下示例显示应用程序如何从Material UI颜色系统定义UIColor常量，并通过类方法使用它们：

```Swift
import UIKit

extension UIColor {
    static var customAccent: UIColor { return MaterialUI.red500 }
    …
}

fileprivate enum MaterialUI {
    static let orange600 = UIColor(red:   0xFB / 0xFF,
                                   green: 0x8C / 0xFF,
                                   blue:  0x00 / 0xFF,
                                   alpha: 1) // #FB8C00
    …
}
```

如果您的应用程序使用了这种模式，则可以使用iOS 13中新的`init(dynamicProvider :) `UIColor初始化方法使其与夜间模式兼容，如下所示：

```Swift
import UIKit

extension UIColor
    static var customAccent: UIColor {
        if #available(iOS 13, *) {
            return UIColor { (traitCollection: UITraitCollection) -> UIColor in
                if traitCollection.userInterfaceStyle == .dark {
                    return MaterialUI.orange300
                } else {
                    return MaterialUI.orange600
                }
            }
        } else {
            return MaterialUI.orange600
        }
    }
}
```

> 此外可以扩展此方法，以支持默认和高对比度模式下的夜间模式：
>
> ```Swift
> // iOS >= 13
> UIColor { (traitCollection: UITraitCollection) -> UIColor in
>     switch(traitCollection.userInterfaceStyle,
>            traitCollection.accessibilityContrast)
>     {
>         case (.dark, .high): return MaterialUI.orangeA200
>         case (.dark, _):     return MaterialUI.orange300
>         case (_, .high):     return MaterialUI.orangeA700
>         default:             return MaterialUI.orange600
>     }
> }
> ```

使用上面声明的颜色无法在 Interface Builder 中使用，如果我们需要在 Storyboards 或者 XIBs 中动态地使用颜色，最好的方式就是 color assets。

# Manage Colors with an Asset Catalog

使用 color assets （iOS 11）我们可以同一个颜色名在不同的 UITraitCollection 显示下设置不同的显示方案。

和使用 Image 和 Data 类是，在 Asset Catalog 中添加一个“New Color Set”，appearance 可以设置其适用于哪种模式，在 Input Method 中选择“8-bit Hexadecimal” 则可以以8位16进制数字输出颜色。

同样，使用  Asset Catalog 添加的 Color Set 也可以在代码中使用，只需要使用 Color Set 名来初始化。

```Swift
extension UIColor {
    @available(iOS 11, *)
    var customAccent: UIColor! {
        return UIColor(named: "Accent")
    }
}
```

> 一个应用可以有多个 Asset Catalogs，考虑使用一个单独的 Catalog 来存放各种 Color Sets。