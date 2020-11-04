---
title: C Storage Classes
author: strayRed
date: 2020-11-01 18:50:00 +0800
categories: [ios, objective-c]
tags: [ios, objective-c]
---

在 C 语言中，程序内变量或函数的 *作用域（scope）* 和 *寿命（lifetime）* 是由其 *存储类（storage class）* 确定的。每个变量都具有生存周期，或存储其值的上下文。方法，同变量一样，也存在或可见于一个特殊的范围里，这就决定了哪一部分程序知道且能够访问它们。

C中有四种存储类：

- `auto`
- `register`
- `static`
- `extern`

# auto

`auto` 是默认的  `storage classes`，当进入对应的作用域（代码块）中时，声明为 `auto` 的变量的内存空间会被自动地分配，并且在离开该作用域（代码块）的时候被释放，只有在对应的代码块，或者嵌套的代码块中才能访问这个变量。

# register

大多数 Objective-C 程序员可能也不熟悉 `register`，因为它没有被广泛的使用在 `NS` 世界里。

`register` 行为就像 `auto`，但不同的是它们不是被分配到堆栈中，它们被存储在一个[寄存器](https://zh.wikipedia.org/wiki/寄存器)里。

寄存器能比内存提供更快的访问速度，但由于内存管理的复杂性，把变量放在寄存器中并不能保证程序变得更快。事实上，很可能由于在寄存器上占用了不必要的空间而最终使代码执行得更慢。使用寄存器实际上只是一个给编译器存储变量的建议，开发者实现时可以选择是否遵从这一点。

# static

当涉及到`storage classes`，`static` 通常指两个方面。

- 在方法或者函数中的一个 `static` 变量，并且保留了上次调用的值。
- 一个全局的方法或者变量，可以被当前文件中的任何方法或者函数调用。
## 静态单例

OC中的一个常见的模式，使用类方法这一形式，暴露出函数中的`static` 变量供外部访问。`dispatch once` 用于保证变量初始化在一个线程安全的方式下只发生一次：
```ObjectiveC
+ (instancetype)sharedInstance {
  static id _sharedInstance = nil;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
      _sharedInstance = [[self alloc] init];
  });
  return _sharedInstance;
}
```

# extern

`static` 使特定文件中的方法或者变量在该文件中可见，那么`extern`则可以使它们被所有文件可见。

一般来说，全局变量并不是一个好主意。由于没有如何及何时改变值的任何限制，常常导致无法调试的错误。在 Objective-C，对 `extern` 有两个常见和实际的用途。

## 全局字符串常量

任何时候，如果你的应用程序要在一个公共头文件申明一个非自然语言的字符串常量，都应该将其声明为外部字符串常量。尤其是在声明诸如 `userInfo` 字典，`NSNotification` 名称和 `NSError` 域的时候。

该模式是在公共头文件里申明一个 `extern` 的 `NSString * const`，并在实现文件里定义该 `NSString * const`：

AppDelegate.h

```objective-c
//在h文件中声明
extern NSString * const kAppErrorDomain;
```

AppDelegate.m
```objective-c
//在m文件中实现
NSString * const kAppErrorDomain = @"com.example.yourapp.error";
```
## 公共方法

一些 API 可能会想要公开曝光一些辅助方法。出于仅提供辅助而与具体状态无关的考虑，用方法来封装这些行为是一个很好的方式，而且如果特别有用，还可能值得使其全局可用。

 TransactionStateMachine.h
```objective-c
//定义枚举 TransactionState
typedef NS_ENUM(NSUInteger, TransactionState) {
    TransactionOpened,
    TransactionPending,
    TransactionClosed,
};

// 声明了 TransactionState 转换为 NSString * 的方法
extern NSString * NSStringFromTransactionState(TransactionState state);
```

 TransactionStateMachine.m

```objective-c
// 在 .m 文件中实现这个方法
NSString * NSStringFromTransactionState(TransactionState state) {
  switch (state) {
    case TransactionOpened:
      return @"Opened";
    case TransactionPending:
      return @"Pending";
    case TransactionClosed:
      return @"Closed";
    default:
      return nil;
  }
}
```

> 转载自[NSHipster](https://nshipster.com/c-storage-classes/)