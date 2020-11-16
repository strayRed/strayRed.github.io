---
title: NS_ENUM & NS_OPTIONS
author: strayRed
date: 2020-11-16 15:09:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

`NS_ENUM` 和 `NS_OPTIONS` 都不算太古老的宏，在iOS 6 / OS X Mountain Lion才开始有，他们都是代替 `enum` 的更好的办法。

> 如果你想在更早的iOS或OS X系统中使用这两个宏，简单定义一下就好了：

```Objective-C
#ifndef NS_ENUM
#define NS_ENUM(_type, _name) enum _name : _type _name; enum _name : _type
#endif
```

`enum`，或者其他枚举类型（例如每周的星期几，或TableViewCell的类型等），都是通过C的方法去为预设值定义常量。在一个 `enum` 定义中，没有被赋予特别值的常量都会自动被赋为从0开始的连续值。

有几种合法的方式来定义 `enum`。容易产生困惑的地方是它们每种方法之间略有不同，但不必须想太多，任选一种即可。

例如：

```Objective-C
enum {
    UITableViewCellStyleDefault,
    UITableViewCellStyleValue1,
    UITableViewCellStyleValue2,
    UITableViewCellStyleSubtitle
};
```

…定义整型值，但不定义类型。

另一种方法:

```Objective-C
typedef enum {
    UITableViewCellStyleDefault,
    UITableViewCellStyleValue1,
    UITableViewCellStyleValue2,
    UITableViewCellStyleSubtitle
} UITableViewCellStyle;
```

…定义适合特性参数的 `UITableViewCellStyle` 类型。

然而，之前苹果自己的代码中都用这种方法来定义 `enum` ：

```Objective-C
typedef enum {
    UITableViewCellStyleDefault,
    UITableViewCellStyleValue1,
    UITableViewCellStyleValue2,
    UITableViewCellStyleSubtitle
};

typedef NSInteger UITableViewCellStyle;
```

…这种方法给出了 `UITableViewCellStyle` 确定的大小，但并没有告诉编译器这个类型和之前的 `enum` 有什么关系。

# NS_ENUM

从现在开始 `UITableViewCellStyle` 的定义已经变成这个样子了：

```Objective-C
typedef NS_ENUM(NSInteger, UITableViewCellStyle) {
    UITableViewCellStyleDefault,
    UITableViewCellStyleValue1,
    UITableViewCellStyleValue2,
    UITableViewCellStyleSubtitle
};
```

`NS_ENUM` 的第一个参数是用于存储的新类型的类型。在64位环境下，`UITableViewCellStyle` 和 `NSInteger` 一样有8bytes长。你要保证你给出的所有值能被该类型容纳，否则就会产生错误。第二个参数是新类型的名字。大括号里面和以前一样，是你要定义的各种值。

这种实现方法提取了之前各种不同实现的优点，甚至有提示编辑器在进行 `switch` 判断时检查类型匹配的功能。

# NS_OPTIONS

`enum` 也可以被定义为[按位掩码（bitmask）](https://en.wikipedia.org/wiki/Mask_(computing))。用简单的`OR` (`|`)和`AND` (`&`)数学运算即可实现对一个整型值的编码。每一个值不是自动被赋予从0开始依次累加1的值，而是手动被赋予一个带有一个bit偏移量的值：类似`1 << 0`、 `1 << 1`、 `1 << 2`等。如果你能够心算出每个数字的二进制表示法，例如：`10110` 代表 22，每一位都可以被认为是一个单独的布尔值。例如在UIKit中， `UIViewAutoresizing` 就是一个可以表示任何flexible top、bottom、 left 或 right margins、width、height组合的位掩码。

不像 `NS_ENUM` ，位掩码用 `NS_OPTIONS` 宏。

语法和 `NS_ENUM` 完全相同，但这个宏提示编译器值是如何通过位掩码 `|` 组合在一起的。同样的，注意值的区间不要超过所使用类型的最大容纳范围。