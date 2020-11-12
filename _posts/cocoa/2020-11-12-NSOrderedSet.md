---
title: NSOrderedSet
author: strayRed
date: 2020-11-12 16:57:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

`NSOrderedSet`是一个有序版本的`NSSet`。

实际上，`NSOrderedSet`**并不是**`NSSet`的子类。

`NSOrderedSet`如果是一个 `NSSet` 的*子类*这个逻辑看起来是很完美的。它跟 `NSSet` 具有相同的方法，还添加了一些 `NSArray`风格的方法，像 `objectAtIndex:`。据大家所说，它似乎完全满足了[里氏替换原则](https://zh.wikipedia.org/zh-cn/里氏替换原则)的要求，其大致含义为：

> 在一段程序中， 如果 `S` 是 `T`的子类型，那么`T`类型的对象就有可能在没有任何警告的情况下被替换为`S`类型的对象。

# Mutable / Immutable Class Clusters

[类簇](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CocoaFundamentals/CocoaObjects/CocoaObjects.html%23//apple_ref/doc/uid/TP40002974-CH4-SW34)是 Foundation framework 核心所使用的一种设计模式；也是日常使用 Objective-C 简洁性的本质。

当类簇提供简单的可扩展性的同时，他也把有些事情变得很棘手，尤其是对于一对可变/不可变类像 `NSSet` / `NSMutableSet`来说。

正如 [Tom Dalling](http://tomdalling.com/) 在这个 [Stack Overflow](http://stackoverflow.com/questions/11278995/why-doesnt-nsorderedset-inherit-from-nsset) 回答里面巧妙的演示，`-mutableCopy` 这个方法的创建是跟 Objective-C 对单继承的约束矛盾的。

开始，让我们来看下 `-mutableCopy` 在类簇中是如果工作的：

```Objective-C
NSSet* immutable = [NSSet set];
NSMutableSet* mutable = [immutable mutableCopy];

[mutable isKindOfClass:[NSSet class]]; // YES
[mutable isKindOfClass:[NSMutableSet class]]; // YES
```

现在让我们假设下`NSOrderedSet`事实上是`NSSet`的子类：

```Objective-C
// @interface NSOrderedSet : NSSet

NSOrderedSet* immutable = [NSOrderedSet orderedSet];
NSMutableOrderedSet* mutable = [immutable mutableCopy];

[mutable isKindOfClass:[NSSet class]]; // YES
[mutable isKindOfClass:[NSMutableSet class]]; // NO (!)
```

因为了满足类簇的定义，假设  `NSOrderedSet`是`NSSet`的子类，那么 `NSMutableOrderedSet` 是  `NSOrderedSet` 的子类的同时，也应该为`NSMutableSet`的子类，这显然是不符合单继承的原则的。

这也是一个菱形缺陷问题，也是单继承面向对象的语言的通病，即不同父类（`NSOrderedSet`和`NSSet`）的子类对象（`NSMutableOrderedSet`和`NSMutableSet`）不能共享接口（这里表现为`NSMutableOrderedSet`实例不能被当作 `NSMutableSet` 使用）。



