---
title: Date Components
author: strayRed
date: 2020-11-17 14:31:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

DateComponents是有用的，但是模棱两可的类型。

在一种情况下，一个 DateComponents 实例可用于表示特定的日历日期。 但是在另一种情况下，同一对象可能会被用作一段时间。 例如，日期成分对象的年份设置为2018，月份设置为10，日期设置为10，则可以表示2018年，10个月和10天或2018年第十个月的第十天：

```Swift
import Foundation

let calendar = Calendar.current
let dateComponents = DateComponents(calendar: calendar,
                                    year: 2018,
                                    month: 10,
                                    day: 10)

// DateComponents as a date specifier
let date = calendar.date(from: dateComponents)! // 2018-10-10

// DateComponents as a duration of time
calendar.date(byAdding: dateComponents, to: date) // 4037-08-20
```

# Date Components as a Representation of a Calendar Date

## Extracting Components from a Date

可以使用Calendar方法`component(_：from :)`为特定日期创建 DateComponents 对象：

```Swift
let date = Date() // 2018-10-10T10:00:00+00:00
let calendar = Calendar.current
calendar.dateComponents([.year, .month, .day], from: date)
// (year: 2018, month: 0, day: 10)
```
>  NSCalendar提供了各种日历的实现。 它提供了几种不同日历的数据，包括佛教，公历，希伯来语，伊斯兰和日语。我们同样可以进一步设置 NSCalendar 的 `timeZone` 和 `locale` 属性来获取更详细的本地化信息。
>
>  NSCalendar与NSDateComponents类密切相关，NSDateComponents的实例描述了日历计算所需日期的组成元素。日历则由由NSLocale中的常量指定。 您可以使用NSCalendar方法currentCalendar最轻松地获取用户首选语言环境的日历。 您还可以使用键NSLocaleCalendar从任何NSLocale对象获取默认日历。
>
>  此外，如果使用 NSDateFormatter 本地化输出 NSDate，那么可以设置 NSDateFormatter的 locale 属性。

DateComponents 由以下元素组成：

|      Component      |        Value        |
| :-----------------: | :-----------------: |
|     `calendar`      |      gregorian      |
|     `timeZone`      | America/Los_Angeles |
|        `era`        |          1          |
|      `quarter`      |          0          |
|       `year`        |        2018         |
|       `month`       |         10          |
|        `day`        |          3          |
|       `hour`        |         10          |
|      `minute`       |          0          |
|      `second`       |          0          |
|    `nanosecond`     |      123456789      |
|      `weekday`      |          4          |
|  `weekdayOrdinal`   |          2          |
|    `weekOfMonth`    |          2          |
|    `weekOfYear`     |         41          |
| `yearForWeekOfYear` |        2018         |
|    `isLeapMonth`    |        false        |

# Era and Year

公历（Gregorian）有两个时代：公元前和公元（或者，公元前和公元前）。 它们各自`era`属性分别为0和1。无论时代是什么，年份成分始终为正数。

# Quarter

在学术界和商业界，一年被分为四个 Quarter，分别为Q1，Q2，Q3，Q4。

在 iOS 12 和 macOS Mojave 中，即使指定了年和月，`dateComponents(_：from :)`方法也不会填充返回值的 Quarter 属性。See [rdar://35247464](http://www.openradar.me/35247464)。

解决方法是，可以使用DateFormatter使用日期格式“ Q”生成字符串并解析其整数值：

```Swift
let formatter = DateFormatter()
formatter.dateFormat = "Q"
Int(formatter.string(from: Date())) // 4
```

# Weekday, Weekday Ordinal, and Week of Month

根据 Calendar的 locale 属性不同 会影响 `weekday`，`weekdayOrdinal` `weekOfMonth` 等 DateComponents 字段的值。

# Week of Year and Year for Week of Year

这两个可能是所有日期组件中最令人困惑的部分。 其中一部分与可笑的API名称yearForWeekOfYear有关，但主要归因于对ISO周日期缺乏普遍的了解。

weekOfYear组件返回有关日期的ISO周号。 例如，2018年10月10日发生在ISO第41周。

yearForWeekOfYear组件对跨两个日历年的几周很有帮助。 例如，今年的除夕-2018年12月31日-在星期一。 因为发生在2019年的第一周，所以它的weekOfYear值是1，其yearForWeekOfYear值是2019年，并且它的年值是2018年

# Creating a Date from Date Components

除了从日期中提取组件之外，我们还可以使用 Calendar 方法`date（from :)`从相反的方向从 Components 中创建日期。

```Swift
var date: Date?

// Bad
let timestamp = "2018-10-03"
let formatter = ISO8601DateFormatter()
formatter.formatOptions =
    [.withFullDate, .withDashSeparatorInDate]
date = formatter.date(from: timestamp)

// Good
// 比起使用时间戳，这种方法更加正确
let calendar = Calendar.current
let dateComponents =
    DateComponents(calendar: calendar,
                   year: 2018, month: 10, day: 3)
date = calendar.date(from: dateComponents)
```

当使用日期组件表示日期时，仍然存在一些歧义。 日期成分可能未指定（并且经常是未指定），因此可以从其他上下文中推断出时代（era）或小时（hour）等成分的值。 

当您使用`date（from :)`方法时，实际上是在告诉Calendar搜索满足您指定条件的下一个日期。有时这是不可能的，例如日期组成部分的值相互矛盾（例如weekOfYear = 1和weekOfMonth = 3），或者超出日历允许的值（例如hour = 127）。 在这些情况下，`date(from :)`返回nil。

# Getting the Range of a Calendar Unit

```Swift
let date = Date() // 2018-10-10T10:00:00+00:00
let calendar = Calendar.current

var beginningOfMonth: Date?

// OK
let dateComponents =
    calendar.dateComponents([.year, .month], from: date)
beginningOfMonth = calendar.date(from: dateComponents)

// Better
// iOS 10 available
beginningOfMonth =
    calendar.dateInterval(of: .month, for: date)?.start
```

# Date Components as a Representation of a Duration of Time

## Calculating the Distance Between Two Dates

使用 Calendar `dateComponents(_:from:to:)`可以计算一个 DateInterval 的间隔的 DateComponents。

```Swift
let date = Date() // 2018-10-10T10:00:00+00:00
let calendar = Calendar.current

let monthInterval =
    calendar.dateInterval(of: .month, for: date)!

calendar.dateComponents([.hour],
                        from: monthInterval.start,
                        to: monthInterval.end)
        .hour // 744
```

## Adding Components to Dates

```Swift
let date = Date() // 2018-10-10T10:00:00+00:00
let calendar = Calendar.current

var tomorrow: Date?

// Bad
tomorrow = date.addingTimeInterval(60 * 60 * 24)

// Good
tomorrow = calendar.date(byAdding: .day,
                         value: 1,
                         to: date)
```

```Swift
let date = Date()
let calendar = Calendar.current

// Adding a year
calendar.date(byAdding: .year, value: 1, to: date)

// Adding a year and a day
let dateComponents = DateComponents(year: 1, day: 1)
calendar.date(byAdding: dateComponents, to: date)
```

```Swift
let dateComponents =
    calendar.dateComponents([.hour,
                             .minute,
                             .second,
                             .nanosecond],
                            from: date)
// 获取下一个符合条件的Date
tomorrow = calendar.nextDate(after: date,
                             matching: dateComponents,
                             matchingPolicy: .nextTime,
                             repeatedTimePolicy: .first,
                             direction: .forward)
```