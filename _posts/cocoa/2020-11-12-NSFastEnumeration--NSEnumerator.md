---
title: NSFastEnumeration / NSEnumerator
author: strayRed
date: 2020-11-12 14:51:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

在 Cocoa 当中，生成式方法（for-in）可以用在任何实现了 `NSFastEnumeration` 协议的类上，包括 `NSArray`, `NSSet`, 和 `NSDictionary`。

# NSFastEnumeration Protocol

`NSFastEnumeration` 只包含一个方法：

```Objective-C
- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state
                                  objects:(id *)stackbuf
                                    count:(NSUInteger)len
```

- `state`: 遍历中需要使用的上下文信息，确保在遍历过程中集合不被修改。
- `stackbuf`: 一个 C 数组，内容是将要被调用者遍历的对象们.
- `len`: stackbuf 中最多能返回的元素数量.

## NSFastEnumerationState

```Objective-C
typedef struct {
      unsigned long state;
      id *itemsPtr;
      unsigned long *mutationsPtr;
      unsigned long extra[5];
} NSFastEnumerationState;
```

- `state`: 遍历器使用的一个状态信息，通常在遍历开始时这个值被设置成 0
- `itemsPtr`: 一个 C 对象数组
- `mutationsPtr`: 用于检测集合是否被修改的状态信息
- `extra`: 一个 C 数组，可以用来保存返回值

对于 `NSFastEnumeration` 你需要了解的是它很快 。哪怕没有很明显的超过，也至少和你自己使用 `for` 循环是一样快的。高速背后的秘密是 `-countByEnumeratingWithState:objects:count:` 这个函数，它会缓存集合成员，并按需加载。和单线程的 `for` 循环实现不同的是，对象的加载是可以并发的，以最大程度利用系统资源。

# NSEnumerator

在 `NSFastEnumeration` 出现之前（大约是 OS X Leopard / iOS 2.0 时期），`NSEnumerator`被用于遍历。在外行人看来，`NSEnumerator` 是一个实现了下面两个方法的抽象类：

```Objective-C
- (id)nextObject
- (NSArray *)allObjects
```

`nextObject` 返回集合类型中的下一个元素，如果没有就返回 `nil`。`allObjects` 返回所有剩余的元素（如果有的话）。`NSEnumerator` 只能向一个方向遍历，而且只能进行单增。

要想遍历一个集合当中的所有元素，需要这样使用 `NSEnumerator`：

```Objective-C
id object = nil;
NSEnumerator *enumerator = ...;
while ((object = [enumerator nextObject])) {
    NSLog(@"%@", object);
}
```

`NSEnumerator` 本身也实现了 `<NSFastEnumeration>` 协议：

```Objective-C
for (id object in enumerator) {
    NSLog(@"%@", object);
}
```

如果你想给自己的非集合支持的自定义类添加快速遍历功能的话，使用 `NSEnumerator` 可能是更加方便而且易用的方法，相比深入 `NSFastEnumeration` 的实现细节而言。

关于 `NSEnumeration` 几个有趣的小知识：

- 使用一行代码实现数组反转：`array.reverseObjectEnumerator.allObjects`。
- 在 [`NSEnumeratorLinq`](https://github.com/k06a/NSEnumeratorLinq) 的帮助下进行 LINQ 风格的操作，这个库使用了链式的 `NSEnumerator` 子类。
- 使用 [`TTTRandomizedEnumerator`](https://github.com/mattt/TTTRandomizedEnumerator) 可以方便地随机取出集合当中的元素，又是一个第三方库，它支持使用随机顺序进行元素遍历。

# Enumerate With Blocks

最后，随着 OS X Snow Leopard / iOS 4 中 blocks 语法的引入，一种新的基于 block 的遍历集合的方法也被加入进来：

```Objective-C
[array enumerateObjectsUsingBlock:^(id object, NSUInteger idx, BOOL *stop) {
    NSLog(@"%@", object);
}];
```

诸如 `NSArray`, `NSSet`, `NSDictionary`，和 `NSIndexSet` 这些集合类型都包含了一系列类似的 block 遍历方法。

这种方法的一个优势是当前对象的索引 （`idx`）会跟随对象传递进来。`BOOL` 指针可以用于提前返回，相当于传统 C 循环当中的 `break` 语句。

除非你真的需要在遍历时使用数字索引，使用 `for/in` `NSFastEnumeration` 几乎总是更快的选择。

最后一个需要了解的是，这个系列方法还有带有 `options` 参数的扩展版本：

```Objective-C
- (void)enumerateObjectsWithOptions:(NSEnumerationOptions)opts
                         usingBlock:(void (^)(id obj, NSUInteger idx, BOOL *stop))block
```

## NSEnumerationOptions

```Objective-C
enum {
   NSEnumerationConcurrent = (1UL << 0),
   NSEnumerationReverse = (1UL << 1),
};
typedef NSUInteger NSEnumerationOptions;
```

> - `NSEnumerationConcurrent`: 指示 Block 遍历应当是并发的。遍历的顺序是不确定而且未定义的；这个标志位是一个提示，可能在某些情况下会被实现方忽略；Block 中的代码必须在并发调用的情况下是安全的。

> - `NSEnumerationReverse`: 指示遍历应该是反向进行的，这个选项在 `NSArray` 和 `NSIndexSet` 中可用；在 `NSDictionary` 和 `NSSet` 中，以及和 `NSEnumerationConcurrent` 同时使用的情况下，行为是未定义的。

再重申一次，快速遍历几乎可以肯定要比 block 遍历快很多，不过如果你被迫要使用 blocks 的话这些选项可能会有用。