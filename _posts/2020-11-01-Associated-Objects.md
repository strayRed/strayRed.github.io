---
title: Associated Objects
author: strayRed
date: 2020-11-01 15:24:00 +0800
categories: [Objective-C]
tags: [Objective-C]
---
对象关联（或称为关联引用）本来是 `Objective-C 2.0` 运行时的一个特性，起始于 `OS X Snow Leopard` 和 `iOS 4` 。相关参考可以查看  `<objc/runtime.h>`  中定义的以下三个允许你将任何键值在运行时关联到对象上的函数：

- `objc_setAssociatedObject`
- `objc_getAssociatedObject`
- `objc_removeAssociatedObjects`

这允许开发者对已经存在的类在扩展中添加自定义的属性。

# Associated Key

因为是使用唯一的 `key` 地址来检索关联对象，所以对 `key` 的要求应该是常量，唯一的。一般推荐的类型为 `static char`，或者说是指针类型。当然可以用更好的方式实现，也就是用 `selector`，使用 `_cmd` 可以访问当前方法的 `selector`。

# Associative Object Behaviors

|            **Behavior**             | `@property` **Equivalent**                            | **Description**                                              |
| :---------------------------------: | ----------------------------------------------------- | ------------------------------------------------------------ |
|       OBJC_ASSOCIATION_ASSIGN       | @property (assign)` or `@property (unsafe_unretained) | Specifies a weak(unsafe) reference to the associated object. |
| `OBJC_ASSOCIATION_RETAIN_NONATOMIC` | @property (nonatomic, strong)                         | Specifies a strong reference to the associated object, and that the association is not made atomically. |
|  `OBJC_ASSOCIATION_COPY_NONATOMIC`  | @property (nonatomic, copy)                           | Specifies that the associated object is copied, and that the association is not made atomically. |
|       OBJC_ASSOCIATION_RETAIN       | @property (atomic, strong)                            | Specifies a strong reference to the associated object, and that the association is made atomically. |
|        OBJC_ASSOCIATION_COPY        | @property (atomic, copy)                              | Specifies that the associated object is copied, and that the association is made atomically. |

使用 `OBJC_ASSOCIATION_ASSIGN` 在关联对象的引用计数为0时会出现野指针。
## 生命周期

被关联的对象在生命周期内要比对象本身释放的晚很多。它们会在被 `NSObject -dealloc` 调用的 `object_dispose()` 方法中释放。

# 移除关联对象

根据官方文档，不应该使用 `objc_removeAssociatedObjects()` 来移除关联对象。这会导致其他的关联对象被一并移除。正确的方法是调用 `objc_setAssociatedObject` 方法并传入一个 `nil` 值来清除一个关联对象。

# 优秀样例

- 在category中为对象添加私有属性用于更好地去实现public方法。
- 添加public的属性来增强category的功能。
- 创建一个用于KVO的关联观察者。当在一个category的实现中使用KVO时，建议用一个自定义的关联对象而不是该对象本身作观察者。

> 转载自[NSHipster](https://nshipster.com/associated-objects/)