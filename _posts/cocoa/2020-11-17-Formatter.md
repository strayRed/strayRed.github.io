---
title: Formatter
author: strayRed
date: 2020-11-17 17:59:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

# Formatting Numbers and Quantities

|         Class          | Example Output  |      Availability      |
| :--------------------: | :-------------: | :--------------------: |
|   `NumberFormatter`    |   “1,234.56”    |  iOS 2.0 macOS 10.0+   |
| `MeasurementFormatter` | “-9.80665 m/s²” | iOS 10.0+ macOS 10.12+ |
|  `ByteCountFormatter`  |    “756 KB”     |  iOS 6.0+ macOS 10.8+  |
|   `EnergyFormatter`    |    “80 kcal”    | iOS 8.0+ macOS 10.10+  |
|    `MassFormatter`     |    “175 lb”     | iOS 8.0+ macOS 10.10+  |
|   `LengthFormatter`    |  “5 ft, 11 in”  | iOS 8.0+ macOS 10.10+  |
| `MKDistanceFormatter`  |   “500 miles”   |  iOS 7.0+ macOS 10.9+  |

ByteCountFormatter，EnergyFormatter，MassFormatter，LengthFormatter和MKDistanceFormatter被MeasurementFormatter取代了。

|   Legacy Formatter    | Measurement Formatter Unit |
| :-------------------: | :------------------------: |
| `ByteCountFormatter`  |  `UnitInformationStorage`  |
|   `EnergyFormatter`   |        `UnitEnergy`        |
|    `MassFormatter`    |         `UnitMass`         |
|   `LengthFormatter`   |        `UnitLength`        |
| `MKDistanceFormatter` |        `UnitLength`        |

使用HealthKit框架时，您唯一仍可能使用EnergyFormatter，MassFormatter或LengthFormatter的情况是。 这些formatters 提供了与 HKUnit 的转换和交互。

## NumberFormatter

NumberFormatter涵盖了可以想象的数字格式的各个方面。 不管是好是坏（大多数情况下是更好），这个多合一的API可以处理各种的数字格式，百分比以及货币金额。 它甚至可以用几种不同的语言写出数字！

因此，每当使用 NumberFormatter 时，首要任务就是确定数字类型并相应地设置`numberStyle`属性。

### Number Styles

|     Number Style     |      Example Output      |
| :------------------: | :----------------------: |
|        `none`        |           123            |
|      `decimal`       |         123.456          |
|      `percent`       |           12%            |
|     `scientific`     |       1.23456789E4       |
|      `spellOut`      | one hundred twenty-three |
|      `ordinal`       |           3rd            |
|      `currency`      |         $1234.57         |
| `currencyAccounting` |        ($1234.57)        |
|  `currencyISOCode`   |       USD1,234.57        |
|   `currencyPlural`   |   1,234.57 US dollars    |

### Rounding & Significant Digits

`NumberFormatter`的四舍五入行为两个选择：

- 设置 `usesSignificantDigits` 为 true，根据 [significant figures](https://en.wikipedia.org/wiki/Significant_figures) 来进行格式化。

```Swift
var formatter = NumberFormatter()
formatter.usesSignificantDigits = true
formatter.minimumSignificantDigits = 1 // default
formatter.maximumSignificantDigits = 6 // default

formatter.string(from: 1234567) // 1234570
formatter.string(from: 1234.567) // 1234.57
formatter.string(from: 100.234567) // 100.235
formatter.string(from: 1.23000) // 1.23
formatter.string(from: 0.0000123) // 0.0000123
```