---
title: nil / Nil / NULL / NSNull
author: strayRed
date: 2020-11-04 14:58:00 +0800
categories: [ios, objective-c]
tags: [ios, objective-c]
---

|  Symbol  |      Value      |                  Meaning                   |
| :------: | :-------------: | :----------------------------------------: |
|  `NULL`  |   `(void *)0`   |     literal null value for C pointers      |
|  `nil`   |     `(id)0`     | literal null value for Objective-C objects |
|  `Nil`   |   `(Class)0`    | literal null value for Objective-C classes |
| `NSNull` | `[NSNull null]` |  singleton object used to represent null   |

# NULL

C 用 `0` 来作为 *不存在* 的原始值，而 `NULL` 作为指针，而指针环境中，`NULL`和`0`是相等的([faq](http://c-faq.com/null/nullor0.html))。在 Objective-C 中使用 `NULL`的情况只有是与底层`API`交互的时候，比如 `Core Foundation` 或者 `Core Graphics`。

# nil

Objective-C 在 C 的表达 **不存在** 的基础上增加了 `nil`。`nil` 是一个指向不存在的 **对象** 指针。虽然它在语义上与 `NULL` 不同，但它们在技术上是相等的。任何`NSObject`对象在`alloc`的阶段都会被赋予`nil`，所以没有必要为一个对象设置默认值。

`nil`还有一个好处就是可以对它发送任意消息（它本身就是一个`id`类型），在其他语言中，比如Java，这种行为会使你的程序崩溃。但是在Objective-C中，对`nil`调用一个方法会返回一个零值，也就是说，“`nil`产生`nil`”。这一事实本身就大大简化了Objective-C开发人员的工作，因为它避免了在执行任何操作之前检查`nil`的必要性。

```Objective-C
// For example, this expression...
if (name != nil && [name isEqualToString:@"Steve"]) { ... }

// ...can be simplified to:
if ([name isEqualToString:@"Steve"]) { ... }
```



# Nil

在 [Foundation/NSObjCRuntime.h](https://gist.github.com/4469665) 中，`Nil` 被定义为指向零的 **类** 指针。这个`nil`的鲜为人知的大写的表兄并不常常出现，但它至少值得注意。

# NSNull

在框架层面，Foundation 定义了 `NSNull`，即一个类方法 `+null`，它返回一个单独的 `NSNull`对象。`NSNull` 与 `nil` 以及 `NULL` 不同，因为它是一个实际的对象，而不是一个零值。

`NSNull` 在 Foundation 和其它框架中被广泛的使用，以解决如 `NSArray` 和 `NSDictionary` 之类的集合不能有 `nil` 值的缺陷。你可以将 `NSNull` 理解为有效的将 `NULL` 或者 `nil` 值封装[boxing](https://en.wikipedia.org/wiki/Object_type_(object-oriented_programming)#Boxing)，以达到在集合中使用它们的目的：

```Objective-C
NSMutableDictionary *mutableDictionary = [NSMutableDictionary dictionary];
mutableDictionary[@"someKey"] = [NSNull null]; // Sets value of NSNull singleton for `someKey`
NSLog(@"Keys: %@", [mutableDictionary allKeys]); // @[@"someKey"]
```



