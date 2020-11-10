---
title: KVC Collection Operators
author: strayRed
date: 2020-11-09 13:08:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa, objective-c]
---

[KVC 集合运算符](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/KeyValueCoding/Articles/CollectionOperators.html#//apple_ref/doc/uid/20002176-BAJEAIEE)允许在`valueForKeyPath:`方法中使用 key path 符号在一个集合中执行方法。无论什么时候你在 key path 中看见了`@`，它都代表了一个特定的集合方法，其结果可以被返回或者链接，就像其他的 key path 一样。

集合运算符会根据其返回值的不同分为以下三种类型：

- **简单的集合运算符** 返回的是 strings, number, 或者 dates
- **对象运算符** 返回的是一个数组
- **数组和集合运算符** 返回的是一个数组或者集合

> 键-值  编码会在必要的时候把基本数据类型的数据自动装箱和拆箱到`NSNumber`或者`NSValue`中来确保一切工作正常。

# 简单集合操作符

- `@count`: 返回一个值为集合中对象总数的`NSNumber`对象。
- `@sum`: 首先把集合中的每个对象都转换为`double`类型，然后计算其总，最后返回一个值为这个总和的`NSNumber`对象。
- `@avg`: 把集合中的每个对象都转换为`double`类型，返回一个值为平均值的`NSNumber`对象。
- `@max`: 使用`compare:`方法来确定最大值。所以为了让其正常工作，集合中所有的对象都必须支持和另一个对象的比较。
- `@min`: 和`@max`一样，但是返回的是集合中的最小值。

```Objective-C
@interface Product : NSObject
@property NSString *name;
@property double price;
@property NSDate *launchedOn;
@end
```



```Objective-C
[products valueForKeyPath:@"@count"];
[products valueForKeyPath:@"@sum.price"];
[products valueForKeyPath:@"@avg.price"]; 
[products valueForKeyPath:@"@max.price"]; 
[products valueForKeyPath:@"@min.launchedOn"]
```

>  可以简单的通过把 self 作为操作符后面的 key path 来获取一个由`NSNumber`组成的数组或者集合的总值，例如`[@[@(1), @(2), @(3)] valueForKeyPath:@"@max.self"]`。

# 对象操作符

- `@unionOfObjects` / `@distinctUnionOfObjects`: 返回一个由操作符右边的 key path 所指定的对象属性组成的数组。其中`@distinctUnionOfObjects` 会对数组去重, 而且 `@unionOfObjects` 不会。

  ```Objective-C
  NSArray<Product> *inventory = @[iPhone5, iPhone5, iPhone5, iPadMini, macBookPro, macBookPro];
  [inventory valueForKeyPath:@"@unionOfObjects.name"]; // "iPhone 5", "iPhone 5", "iPhone 5", "iPad Mini", "MacBook Pro", "MacBook Pro"
  [inventory valueForKeyPath:@"@distinctUnionOfObjects.name"]; // "iPhone 5", "iPad Mini", "MacBook Pro"
  ```

# 数组和集合操作符

- `@distinctUnionOfArrays` / `@unionOfArrays`: 返回了一个数组，其中包含这个集合中**每个数组**对于这个操作符右面指定的 key path 进行操作之后的值。`distinct`版本会移除重复的值。
- `@distinctUnionOfSets`: 和`@distinctUnionOfArrays`差不多, 但是它期望的是一个包含着`NSSet`对象的`NSSet`，并且会返回一个`NSSet`对象。因为集合不能包含重复的值，所以它只有`distinct`操作。

```Objective-C
//appleStoreInventory和verizonStoreInventory都是NSArray<Product> *类型的数组。
[@[appleStoreInventory, verizonStoreInventory] valueForKeyPath:@"@distinctUnionOfArrays.name"]; // "iPhone 5", "iPad Mini", "MacBook Pro"
```

