---
title: TimeInterval, Date, and DateInterval
author: strayRed
date: 2020-11-16 15:49:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

# Date and Time

Foundation 库的时间类型被称为 `Date` 人们通常把“日期”与“时间”区分开来，说前者与日历日有关，后者与一天中的时间有关。但是实际上`Date`与日历是无关的，它表示的是一个**绝对**的时间点。

Date的另一个混淆之处在于，尽管代表了绝对时间点，但是它是由一个 reference data 的 time interval 得到的相对值([Date](https://github.com/apple/swift-corelibs-foundation/blob/master/Foundation/Date.swift#L17-L20))。这个reference data（参考日期）是格林威治标准时间（GMT）2001年1月1日的第一时刻。

# Date Intervals and Time Intervals

`DateInterval`是 Foundation 的新增类型。 在iOS 10 和 macOS Sierra 中引入的这种类型表示两个绝对时间点之间的闭合间隔（a closed interval）这再次与 TimeInterval 相反，TimeInterval 以秒为单位表示持续时间。

## Getting the Date Interval of a Calendar Unit

为了知道某个时间点的一天中的某个时间点，或者首先是哪一天，你需要查阅日历。在此基础上，可以确定特定日历单位的范围，例如日、月或年。Calendar 的方法 `dateInterval(of:for:)`使此操作非常简单：

```Swift
let calendar = Calendar.current
let date = Date()
// 使用这个函数获取指定日期的 年 月 日 的 DateInterval
let dateInterval = calendar.dateInterval(of: .month, for: date)
 
 let dstComponents = DateComponents(year: 2018,
                                   month: 11,
                                   day: 4)
  // 这里获取了 2018年11月4日 这天的持续时间 TimeInterval
calendar.dateInterval(of: .day,
                      for: calendar.date(from: dstComponents)!)?.duration
```

# Calculating Intersections of Date Intervals

```Swift
import Foundation

let calendar = Calendar.current

// Simon Vouet
// 9 January 1590 – 30 June 1649
let vouet =
    DateInterval(start: calendar.date(from:
        DateComponents(year: 1590, month: 1, day: 9))!,
                 end: calendar.date(from:
                    DateComponents(year: 1649, month: 6, day: 30))!)

// Peter Paul Rubens
// 28 June 1577 – 30 May 1640
let rubens =
    DateInterval(start: calendar.date(from:
                            DateComponents(year: 1577, month: 6, day: 28))!,
                 end: calendar.date(from:
                            DateComponents(year: 1640, month: 5, day: 30))!)

// 取Date Intervals的交集
let overlap = rubens.intersection(with: vouet)!
// 使用日历获取到dateComponents
calendar.dateComponents([.year],
                        from: overlap.start,
                        to: overlap.end) // 50 years
```