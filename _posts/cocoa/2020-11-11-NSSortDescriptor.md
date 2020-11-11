---
title: NSSortDescriptor
author: strayRed
date: 2020-11-11 18:17:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

`NSSortDescriptor`由下述参数组成 ：

- `keyPath`：对于一个给定的集合，对应值的键位将对集合中的每个对象进行排序。
- `ascending`：指定一个集合是否按照升序（`YES`）还是降序（`NO`）进行排序的布尔值。
- `NSComparator`：这是一个可选参数，其类型为 `(AnyObject, AnyObject) -> NSComparisonResult`，我们可以传入一个selecor，其中`NSComparisonResult`为一个枚举类型，它用于返回两个对象的比较结果，小于（左相较于右）返回 `NSOrderedAscending`，大于返回 `NSOrderedDescending`，相等返回`NSOrderedSame`。

> 任何时候当你在为面向用户的字符串排序时，一定要传入`localizedStandardCompare:`选择器，它将根据当前语言环境的语言规则进行排序（语言环境可能会根据大小写，变音符号等等的顺序而发生改变）。

我们可以使用 `NSSortDescriptor` 来设置如何排序数组，再使用NSArray的 `sortedArray` 方法传入 `NSSortDescriptor` `sortedArray`为了确定两个元素的顺序，它会先使用第一个描述符，并检查其结果。如果两个元素在第一个描述符下相同，那么它将使用第二个描述符，以此类推。

```Swift
//声明其属性和方法为objc的成员
@objcMembers
final class Person: NSObject {
  let first: String
  let last: String
  let yearOfBirth: Int
  init(first: String, last: String, yearOfBirth: Int) {
  self.first = first
  self.last = last
  self.yearOfBirth = yearOfBirth
  }
}

//定义排序用的数组
let people = [
Person(first: "Emily", last: "Young", yearOfBirth: 2002),
Person(first: "David", last: "Gray", yearOfBirth: 1991),
Person(first: "Robert", last: "Barnes", yearOfBirth: 1985),
Person(first: "Ava", last: "Barnes", yearOfBirth: 2000),
Person(first: "Joanne", last: "Miller", yearOfBirth: 1994),
Person(first: "Ava", last: "Barnes", yearOfBirth: 1998),
]
//定义排序描述符号
let lastDescriptor = NSSortDescriptor(key: #keyPath(Person.last),
ascending: true,
selector: #selector(NSString.localizedStandardCompare(_:)))
let firstDescriptor = NSSortDescriptor(key: #keyPath(Person.first),
ascending: true,
selector: #selector(NSString.localizedStandardCompare(_:)))
let yearDescriptor = NSSortDescriptor(key: #keyPath(Person.yearOfBirth),
ascending: true)
// 排序
let result = people.sortedArray(using: [lastDescriptor, firstDescriptor, yearDescriptor])
```

`localizedStandardCompare` 用于比较字符串，返回值为一个枚举类型，它分别比较每一个字符的ASC码值，当前的字符串小于给定的字符串，就返回 `orderedAscending`，大于则返回 `orderedDescending`, 等于返回 `orderedSame`。
Swift标准库中 `Sequence` 的`lexicographicallyPrecedes` 方法，它相当于 `localizedStandardCompare` 的范用类型，`lexicographicallyPrecedes` 接受一个同类型元素的 `Sequence`，和一个闭包（可选），返回一个 `Bool`，它会逐个比较 `Sequence` 中的各个元素（相等就比较下一个），最后返回比较结果，默认的比较方式为 `<`，不过我们可以通过传入闭包来自己定义。
此外，这里的 `#keyPath` 写法，`key` 是 `Objective-C` 的键路径，它其实是一个包含属性名字的链表。它并不是 Swift 4 引入的原生的键路径（在OC环境下直接传NSString就可以了）。