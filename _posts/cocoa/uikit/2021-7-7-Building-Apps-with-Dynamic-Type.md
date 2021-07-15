---
title: Building Apps with Dynamic Type
author: strayRed
date: 2021-7-7 17:21:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

一般而言，使用动态字体最直接的方法就是使用`UITextStyle`。

```Swift
label.font = UIFont.preferredFont(forTextStyle: .body)
//自动调节size保证所有的内容都是可见的
label.adjustsFontForContentSizeCategory = true
```

在iOS11之后，所有的`UITextStyle`都可以根据系统的辅助功能的设置来动态地改变字体大小（iOS 11之前只有 body 可以）

对于自定义的字体，可以根据系统预设的 TextStyle 的放大倍率来调整大小。

```Swift
//默认为body
label.font = UIFontMetrics.default.scaledFont(for: customFont)
titleLabel.font = UIFontMetrics(forTextStyle: .title1).scaledFont(for: customFont)
```

为避免字体放大后 label 之间的重叠，可以使用自动布局来调整 UILabel 的 baseline 的位置。

```Swift
secondLabel.firstBaselineAnchor.constraintEqualToSystemSpacingBelow(
firstLabel.lastBaselineAnchor, multiplier: 1.0)
```

根据 `preferredContentSizeCategory`属性，我们可以根据当前的字体大小来做一些自定义的配置或者布局。

```Swift
if traitCollection.preferredContentSizeCategory.isAccessibilityCategory {
 // Vertically stack
} else {
 // Lay out side by side
}

if traitCollection.preferredContentSizeCategory > .extraExtraLarge {
 // Vertically stack
} else {
 // Lay out side by side
}
```



对于 Image，在 asset 中对pdf格式的矢量图勾选`Preserve Vector Data`以此来提供矢量图优化（系统只会保存一份图片拷贝）

```Swift
//同时设置该属性为ture可以让图片根据 Accessibility 的字体大小按比例调整图片大小
imageView.adjustsImageSizeForAccessibilityContentSizeCategory = true
```

