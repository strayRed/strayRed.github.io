---
title: NSDataDetector
author: strayRed
date: 2020-11-11 14:30:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

`NSDataDetector` 是 [`NSRegularExpression`](https://developer.apple.com/library/mac/#documentation/Foundation/Reference/NSRegularExpression_Class/Reference/Reference.html) 的子类，而不只是一个 ICU 的模式匹配，它可以检测半结构化的信息：日期，地址，链接，电话号码和交通信息。

```Objective-C
NSError *error = nil;
//检查地址和电话号码
NSDataDetector *detector = [NSDataDetector dataDetectorWithTypes:NSTextCheckingTypeAddress
                                                        | NSTextCheckingTypePhoneNumber
                                                           error:&error];

NSString *string = @"123 Main St. / (555) 555-5555";
[detector enumerateMatchesInString:string
                           options:kNilOptions
                             range:NSMakeRange(0, [string length])
                        usingBlock:
^(NSTextCheckingResult *result, NSMatchingFlags flags, BOOL *stop) {
  NSLog(@"Match: %@", result);
}];
```

> 当初始化 `NSDataDetector` 的时候，确保只指定你感兴趣的类型。每当增加一个需要检查的类型，随着而来的是不小的性能损失为代价。

返回的result为`NSTextCheckingResult`类型的实例，其有多个属性对应了不同 `NSTextCheckingType...`。

- **NSTextCheckingTypeDate**：`date`,`duration`,`timeZone`
- **NSTextCheckingTypeAddress**：`addressComponents`*`NSTextCheckingNameKey``NSTextCheckingJobTitleKey``NSTextCheckingOrganizationKey``NSTextCheckingStreetKey``NSTextCheckingCityKey``NSTextCheckingStateKey``NSTextCheckingZIPKey``NSTextCheckingCountryKey``NSTextCheckingPhoneKey`
- **NSTextCheckingTypeLink**：`url`
- **NSTextCheckingTypePhoneNumber**：`phoneNumber`
- **NSTextCheckingTypeTransitInformation**：`components`*`NSTextCheckingAirlineKey``NSTextCheckingFlightKey`
