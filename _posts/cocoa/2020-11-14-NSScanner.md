---
title: NSScanner
author: strayRed
date: 2020-11-14 15:23:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

Cocoa提供了一组强大的工具来处理字符串，主要为以下 几种：

- **`string.componentsSeparatedByCharactersInSet`** / **`string.componentsSeparatedByString`**: 根据某一字符将字符串分段。
- **`NSRegularExpression`**：可从预期格式验证和提取字符串数据。
- **`NSDataDetector`**: 完美的检测和提取日期，地址，链接，等等。不过仅限于其预定义类型。
- **`NSScanner`**：可高层度自定义，用于扫描字符串，且不限制边界。

在Cocoa中，NSScanner充当字符串的包装器，扫描字符串的内容以有效地检索子字符串和数值。它提供了几个属性（可读可写）来修改NSScanner实例的行为：

- `caseSensitive` *`Bool`*: 在扫描时是否应该大小写敏感。不过这个属性只适用于`scanString:intoString:` 和`scanUpToString:intoString:`这两个方法，character sets 扫描永远是大小写敏感的。
- `charactersToBeSkipped` *`NSCharacterSet`*: 在查找给定的值时需要略过的字符集。这个属性会在**每次**扫描之前适用，扫描之前，会跳过`NSCharacterSet`中的元素，直到第一个元素不属于给定的`NSCharacterSet`，才会开始扫描，**一但扫描开始，则不会跳过任何元素**。
- `scanLocation` *`Int`*: Scanner在字符串中的当前位置。通过设置此属性，可以重绕或重新启动扫描。
- `locale` *`NSLocale`*: Scanner在扫描数值数组时应使用的区域设置。

NSScanner实例还有有两个附加的只读属性：`string`，它返回scanner正在扫描的字符串；`atEnd`，如果scanLocation在字符串的末尾，则为true。

> NSScanner其实是类簇的抽象类，即使在NSScanner上调用alloc和init，实际上也会收到其子类之一的实例，比如NSConcreteScanner。

# Extracting Substrings and Numeric Values

NSScanner存在的理由是从更大的字符串中提取子字符串和数值。它有15种方法来实现这一点，所有这些方法都遵循相同的基本模式。每个方法都将对输出变量的引用作为参数，并返回指示扫描成功或失败的布尔值：

```Objective-C
NSMutableCharacterSet *whitespaceAndPunctuationSet = [NSMutableCharacterSet punctuationCharacterSet];
[whitespaceAndPunctuationSet formUnionWithCharacterSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]];

NSScanner *stringScanner = [[NSScanner alloc] initWithString:@"John & Paul & Ringo & George."];
//这里的设置是很有必要的，因为使用scanUpTo方法每次扫描后，下一个scanLocation必定是scanUpTo方法参数中的字符串或者Unicode字符，所以要多次扫描字符串，则需要在下次扫描的时候略过这些内容。
//每次查找都会略过前面的标点和空白。
stringScanner.charactersToBeSkipped = whitespaceAndPunctuationSet;

NSString *name;
while ([stringScanner scanUpToCharactersFromSet:whitespaceAndPunctuationSet intoString:&name]) {
    NSLog(@"%@", name);
}
// John
// Paul
// Ringo
// George
```

NSScanner API有两种类扫描用的用例的方法：扫描字符串和数字类型。

##  String Scanners

- `scanString:intoString:` / `scanCharactersFromSet:intoString:`扫描以分别匹配 NSString 或 NSCharacterSet 参数中的字符。如果找到，intoString参数将返回包含扫描的字符串。这些方法通常用于推进 scanners 的`scanLocation`-- `intoString`传入nil可以忽略输出。
- `scanUpToString:intoString:` / `scanUpToCharactersFromSet:intoString:`：扫描NSScanner中的字符串，查找输出字符串的**边界**Unicode字符（字符串也会被当成一个元素处理）。这个边界是与参数的NSString 或者 CharacterSet 中的元素匹配的Unicode标量。如果找到，那么 intoString 就会输出这次查找开始时的`scanLocation`到查找到指定字符串的`scanLocation`这之间（对应原字符串的NSRange）的字符串（不包含这个边界Unicode字符）。如果查找到边界Unicode字符以外的字符，会返回YES（整个字符串如果都不包含边界Unicode字符，也会返回YES），如果这次查找开始的`scanLocation`对应的字符就为边界Unicode字符，就会返回NO（所以多次扫描的时候需要设置 `charactersToBeSkipped ` 属性，来略过上次查找的边界Unicode字符）。

## Numeric Scanners

- `scanDouble:` / `scanFloat:` / `scanDecimal:`扫描  NSScanner 中的字符串中的浮点值，并返回引用的Double、Float或NSDecimal实例中的值（如果找到）。

- `scanInteger:` / `scanInt:` / `scanLongLong:` / `scanUnsignedLongLong:` 同上，Int、Int32、Int64或UInt64实例。

- `scanHexDouble:` / `scanHexFloat:`十六进制浮点值，并返回引用的Double或Float实例中的值（如果找到）。要正确扫描，浮点值必须具有0x或0X前缀。

- `scanHexInt:` / `scanHexLongLong:`十六进制整数值，并返回引用的UInt32或UInt64实例。

## localizedScannerWithString / locale

NSScanner有内置的本地化支持。NSScanner实例可以在通过 `+localizedScannerWithString:`创建时使用用户的区域设置，也可以在设置其`locale`属性设置。特别是，浮点值的分隔符将根据给定的区域设置来被正确地解释。

```Objective-C
double price;
NSScanner *gasPriceScanner = [[NSScanner alloc] initWithString:@"2.09 per gallon"];
[gasPriceScanner scanDouble:&price];
// 2.09

// use a german locale instead of the default
NSScanner *benzinPriceScanner = [[NSScanner alloc] initWithString:@"1,38 pro Liter"];
[benzinPriceScanner setLocale:[NSLocale localeWithLocaleIdentifier:@"de-DE"]];
[benzinPriceScanner scanDouble:&price];
// 1.38
```

## Example: Parsing SVG Path Data

为了使用NSScanner，我们将从SVG路径解析路径数据。SVG路径数据存储为绘制路径的一系列指令，其中“M”表示“移动到”步骤，“L”表示“线到”，“C”表示曲线。大写指令后面是绝对坐标中的点；小写指令后面是相对于路径中最后一个点的坐标。

```Objective-C
static NSString *const svgPathData = @"M28.2,971.4c-10,0.5-19.1,13.3-28.2,2.1c0,15.1,23.7,30.5,39.8,16.3c16,14.1,39.8-1.3,39.8-16.3c-12.5,15.4-25-14.4-39.8,4.5C35.8,972.7,31.9,971.2,28.2,971.4z";

CGPoint offsetPoint(CGPoint p1, CGPoint p2) {
    return CGPointMake(p1.x + p2.x, p1.y + p2.y);
}
```

请注意，点数据相当不规则。有时一个点的x值和y值用逗号分隔，有时不用逗号分隔，同样地，点本身也是这样。用正则表达式解析这些数据可能会很快变得一团糟，但是使用NSScanner，代码是清晰明了的。

我们将定义一个bezierPathFromSVGPath函数，该函数将把路径数据字符串转换为UIBezierPath。我们的扫描仪设置为在扫描值时跳过逗号和空格。

```Objective-C
- (UIBezierPath *)bezierPathFromSVGPath:(NSString *)str {
    NSScanner *scanner = [NSScanner scannerWithString:str];
    
    // skip commas and whitespace
  	// 这里只需要跳过逗号，因为我们重复扫描的内容为坐标，会多次扫描，并用逗号隔开。
    NSMutableCharacterSet *skipChars = [NSMutableCharacterSet characterSetWithCharactersInString:@","];
    [skipChars formUnionWithCharacterSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]];
    scanner.charactersToBeSkipped = skipChars;
    
    // the resulting bezier path
    UIBezierPath *path = [UIBezierPath bezierPath];

```

设置好了，该开始扫描了。我们首先扫描这些关键的字符串。

```Objective-C
    // instructions codes can be upper- or lower-case
    NSCharacterSet *instructionSet = [NSCharacterSet characterSetWithCharactersInString:@"MCSQTAmcsqta"];
    NSString *instruction;
    
    // scan for an instruction code
   //一旦找到 “MCSQTAmcsqta” 中的任意一个，就会调用block，并让 scanner 向前移位
    while ([scanner scanCharactersFromSet:instructionSet intoString:&instruction]) {
```

下一节将扫描一行中的两个double值，将它们转换为CGPoint，然后最终将正确的步骤添加到bezier路径：

```Objective-C
/*接上面的block*/
double x, y;
        NSMutableArray *points = [NSMutableArray array];
        
        // scan for pairs of Double, adding them as CGPoints to the points array
			  // 这个操作同样会让并让 scanner 向前移位
				// 因为已经跳过了逗号，所以能查找到两个坐标点
        while ([scanner scanDouble:&x] && [scanner scanDouble:&y]) {
            [points addObject:[NSValue valueWithCGPoint:CGPointMake(x, y)]];
        }
        // new point in path
				//根据坐标和之前获取到的关键字，绘制
        if ([instruction isEqualToString:@"M"]) {
            [path moveToPoint:[points[0] CGPointValue]];
        } else if ([instruction isEqualToString:@"C"]) {
            [path addCurveToPoint:[points[2] CGPointValue]
                    controlPoint1:[points[0] CGPointValue]
                    controlPoint2:[points[1] CGPointValue]];
        } else if ([instruction isEqualToString:@"c"]) {
            CGPoint newPoint = offsetPoint(path.currentPoint, [points[2] CGPointValue]);
            CGPoint control1 = offsetPoint(path.currentPoint, [points[0] CGPointValue]);
            CGPoint control2 = offsetPoint(path.currentPoint, [points[1] CGPointValue]);
            
            [path addCurveToPoint:newPoint
                    controlPoint1:control1
                    controlPoint2:control2];
        }
    }
    [path applyTransform:CGAffineTransformMakeScale(1, -1)];
    return path;
}
```

## Swift-Friendly Scanning

iOS 13 为Swift的Scanner的扫描方法进行了重写，使其不依赖指针，更加符合Swift的语言风格，原来的方法也在iOS 13和之后被废弃。