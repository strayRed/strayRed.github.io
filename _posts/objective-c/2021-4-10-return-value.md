---
title: return value 
author: strayRed
date: 2021-4-10 17:47:00 +0800
categories: [ios, objective-c]
tags: [ios, objective-c]
---

在objc中，ARC会从两个方面决定函数返回值的retain, release或者autorelease。即函数的签名（signature）以及属性（attributes），比如以`init`、`alloc`、`copy`或者 `new`开头的函数，其会返回值会被`retain`。另外，直接在函数签名尾处添加`NS_RETURNS_RETAINED`，该`attributes`会使ARC无视函数签名，其返回值同样也会被`retain`。

```objc
- (NSString *)pcen {
    return (__bridge_transfer NSString *)CFURLCreateStringByAddingPercentEscapes(NULL, (__bridge CFStringRef) self, NULL, (CFStringRef) @"!*'();:@&=+$,/?%#[]", kCFStringEncodingUTF8);
}
```

```objc
- (void)applicationDidFinishLaunching:(NSNotification *)aNotification
{
   NSString *test = @"This & than > other";

   NSLog(@"pcen: %@", [test pcen]);
}
```

* 在编译`pcen`函数的返回值时，ARC会先查看函数签名，之后则会查看其`attributes`，发现其没有存在与`retain`相关的内容，所以其会将 return 表达式的值（`(__bridge_transfer NSString *)CFURLCr....`）设置为`autorelease`。之后当在 applicationDidFinishLaunching 编译`pcen`实例时，ARC会再次查看函数签名，因为这里存在一个赋值操作，ARC会将return表达式的值会被标记为`retain`。

> 是否为`autorelease`或者`retain`跟 return 的表达式无关，即 __bridge_transfer 并不会决定方法返回值引用的所有权。

* 而当我们为函数`pcen`添加`NS_RETURNS_RETAINED`属性时，return 表达式的值会在一开始就被标记为`retain`。

理论上来说，后者性能是优于前者的，因为ARC不需要插入autorelease/retain，避免了额外的操作，但是Runtime优化了这一过程，因此调用了`_objc_retainAutoreleasedReturnValue`，而不是像`_objc_retain`这样的函数，所以开销并没有看上去那么大。

但是实际上最优的做法是按照`Objective-C`的函数签名约定来设置函数名，即将`pcen`改为`copyPcen`。若不按照该命名方式，则可以考虑添加`NS_RETURNS_RETAINED`，如下。

```objc
+ (instancetype)absoluteLayoutSpecWithChildren:(NSArray<id<ASLayoutElement>> *)children NS_RETURNS_RETAINED AS_WARN_UNUSED_RESULT;
```

You can verify this by invoking "Product > Generate Output > Assembly File" in Xcode, in the resulting assembly you will see in the code for `pcen` something along the lines of:

```
callq   _CFURLCreateStringByAddingPercentEscapes
movq    %rax, %rdi
callq   _objc_autoreleaseReturnValue
addq    $16, %rsp
popq    %rbp
ret
```

(https://stackoverflow.com/questions/12135954/when-is-ns-returns-retained-needed#)