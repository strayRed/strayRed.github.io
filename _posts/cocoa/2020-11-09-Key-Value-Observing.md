---
title:Key Value Observing
author: strayRed
date: 2020-11-09 16:30:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa, objective-c]
---

在 Objective-C 和 Cocoa 中，有许多事件之间进行通信的方式，并且每个都有不同程度的形式和耦合。

- **`NSNotification`** & **`NSNotificationCenter`** 提供了一个中央枢纽，一个应用的任何部分都可能通知或者被通知应用的其他部分的变化。唯一需要做的是要知道在寻找什么，主要是通知的名字。例如，`UIApplicationDidReceiveMemoryWarningNotification` 是给应用发了一个内存不足的信号。

- **Key-Value Observing** 允许 ad-hoc，通过在特定对象之间监听一个特定的 keypath 的改变进行事件内省。例如：一个 ProgressView 可以观察 网络请求的 numberOfBytesRead 来更新它自己的 progress 属性。

- **Delegate** 是一个流行的传递事件的设计模式，通过定义一系列的方法来传递给指定的处理对象。例如：UIScrollView 每次它的 scroll offset 改变的时候都会发送 scrollViewDidScroll: 到它的代理

- **Callbacks** 像 NSOperation 里的 completionBlock（当 isFinished==YES 的时候会触发），或者说是 C 里边的函数指针，作为钩子传入函数，比如 `SCNetworkReachabilitySetCallback(3)`。

`<NSKeyValueObserving>` 或者 KVO，是一个非正式协议，它定义了对象之间观察和通知状态改变的通用机制的。作为一个非正式协议，你不会看到类的这种引以为豪的一致性(只是隐式的假定了使用者为 NSObject 的所有子类)。

KVO 的中心思想其实是相当引人注意的。任意一个对象都可以订阅以便被通知到其他对象状态的改变。这个过程大部分是内建的，自动的，透明的。

# Subscribing

对象可以让观察者添加一个特定的 keypath，oc的键路径是一个字符串，同样可以使用点语法来标记次级属性，用于观察对象的属性。不过对于KVO的一般使用而言，这些属性通常都是类的顶层属性。

> 当通过KVC调用对象时（KVO同样），比如：[self valueForKey:@”someKey”]时，程序会自动试图通过下面几种不同的方式解析这个调用。
>
> - 首先查找对象是否带有 someKey 这个方法，如果没找到，会继续查找对象是否带有someKey这个实例变量（iVar），如果还没有找到，程序会继续试图调用 -(id) valueForUndefinedKey:这个方法。如果这个方法还是没有被实现的话，程序会抛出一个NSUndefinedKeyException 异常错误。
> - 补充：KVC查找方法的时候，不仅仅会查找someKey这个方法，还会查找getsomeKey这个方法， 前面加一个get，或者_someKey以_getsomeKey这几种形式。同时，查找实例变量的时候也会不仅仅查找someKey这个变量，也会查找 _someKey这个变量是否存在。
> - 设计valueForUndefinedKey:方法的主要目的是当你使用-(id)valueForKey方法从对象中请求值时，对象能够在错误发生前，有最后的机会响应这个请求。

添加一个观察者的方法是 `–addObserver:forKeyPath:options:context:`:

```Objective-C
// 这个方法是被观察的对象调用
- (void)addObserver:(NSObject *)observer
         forKeyPath:(NSString *)keyPath
            options:(NSKeyValueObservingOptions)options
            context:(void *)context
```

- **observer**：KVO 通知的观察者。观察者必须实现 `key-value observing` 方法 `observeValueForKeyPath:ofObject:change:context:`。
- **keyPath**：被观察者的属性的 keypath，值不能是 nil。
- **options**: `NSKeyValueObservingOptions` 的组合，它指定了观察通知中包含了什么。
- **context**：在 `observeValueForKeyPath:ofObject:change:context:` 传给 `observer` 参数的上下文。

让这个API不堪入目的事实就是最后两个参数经常是 `0` 和 `NULL`。

## NSKeyValueObservingOptions

`options` 代表 `NSKeyValueObservingOptions` 的位掩码，需要注意 `NSKeyValueObservingOptionNew` & `NSKeyValueObservingOptionOld` ，因为这些是你经常要用到的，可以跳过 `NSKeyValueObservingOptionInitial` & `NSKeyValueObservingOptionPrior`:

- **NSKeyValueObservingOptionNew**: 表明变化的字典应该提供新的属性值，如何可以的话。

- **NSKeyValueObservingOptionOld**: 表明变化的字典应该包含旧的属性值，如何可以的话。

- **NSKeyValueObservingOptionInitial**: 如果被设置，一个通知会立即发送给观察者，甚至在观察者注册方法返回之前，如果 `NSKeyValueObservingOptionNew` 也被同时设置，则通知中的更改字典将始终包含 `NSKeyValueChangeNewKey`的值。如果有设置了`NSKeyValueChangeOldKey`，首次调用是不会包括在内 。（在一个 initial notification 里，被观察者的当前属性可能是旧的，但对观察者来说是新的），也就是说一旦设置了这个`Option`，在调用`addObserver`方法的同时，观察者的 `observeValueForKeyPath:ofObject:change:context:` 方法也会被调用（传入被观察属性的当前值）。类似于`BehaviorSubject`的特性。

- **NSKeyValueObservingOptionPrior**：在value发生变化之前发送一次通知，value变化之后正常发送一次通知。（也就是`willChange` 和 didChange都发送通知）。
   在变化前的通知（第一次通知）中，会包含`NSKeyValueChangeNotificationIsPriorKey` ,不包含`NewKey`（即使观察`OptionNew`，在第一次通知中，`newKey`也会被丢弃）

# Responding

KVO没有方法指定自定义的`selectors`来处理观察者，就像控件里使用的 Target-Action 模式那样，相反地，对于观察者，所有的改变都被聚集到一个单独的方法 `-observeValueForKeyPath:ofObject:change:context:`

```Objective-C
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context
```

这些参数跟我们指定的 `–addObserver:forKeyPath:options:context:` 是一样的，change 是个例外，它取决于哪个 `NSKeyValueObservingOptions` 选项被使用。

## NSKeyValueChangeKey

`change`是一个`NSDictionary<NSKeyValueChangeKey, id> *)`类型。`NSKeyValueChangeKey`是一个`NSString`的`typedef` ，它包括以下几个字符串。

- **NSKeyValueChangeKindKey**：值是一个 `NSKeyValueChange`枚举类型，会根据对属性操作的内容返回不同的值。

  ```Objective-C
  typedef NS_ENUM(NSUInteger, NSKeyValueChange) {
  NSKeyValueChangeSetting = 1, //调用set方法
  
  // 观测的值是一个可变集合/数组的情况
  NSKeyValueChangeInsertion = 2, //插入操作
  NSKeyValueChangeRemoval = 3, //移除操作
  NSKeyValueChangeReplacement = 4, // 替换操作
  };
  ```

  

**NSKeyValueChangeIndexesKey**：如果上述的值是`NSKeyValueChangeInsertion`, `NSKeyValueChangeRemoval`, 或者 `NSKeyValueChangeReplacement`。那么这个key会对应一个`NSIndexSet`类型的值，包含了 `inserted`, `removed`, 或者 `replaced`的对象的NSIndexSet。

**NSKeyValueChangeNewKey**：新值，当`NSKeyValueObservingOptionNew`被设置时才会存在，并且当`NSKeyValueChangeKindKey`为2/3/4的时候，对应的是变化的那个元素，而不是变化后的数组。

**NSKeyValueChangeOldKey**：同上，当`NSKeyValueObservingOptionOld`被设置时才会存在。

**NSKeyValueChangeNotificationIsPriorKey**：当`NSKeyValueObservingOptionPrior`被设置时，发送第一次（willChange）通知的时候有效，为`NSNumber`类型，封装了Bool值。

## Correct Context Declarations

因为观察者的回调方法只有一个，所以需要根据参数判断不同的被观察者。最安全的方法是做一个 `context` 等式检查。

声明一个`context`的最佳方式是这样

```C
static void * XXContext = &XXContext;
```

就是这么简单：一个静态变量存着它自己的指针。这意味着它自己什么也没有。

```Objective-C
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context
{
  if (context == XXContext) {
      if ([keyPath isEqualToString:NSStringFromSelector(@selector(isFinished))]) {

      }
  }
}
```

## Better Key Paths

比起直接通过 NSString 字面量这种不编译不安全的方式传递 keyPath，一个聪明的解决方案是使用 `NSStringFromSelector` 和一个 `@selector` 字面值:

```Objective-C
NSStringFromSelector(@selector(isFinished))
```

因为 `@selector` 检查目标中的所有可用 selector，这虽然不能阻止所有的错误，但是可以降低程序崩溃的风险。

```Objective-C
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context
{
    if ([object isKindOfClass:[NSOperation class]]) {
        if ([keyPath isEqualToString:NSStringFromSelector(@selector(isFinished))]) {

        }
    } else if (...) {
        // ...
    }
}
```

# Unsubscribing

当一个观察者完成了监听一个对象的改变，需要调用 `–removeObserver:forKeyPath:context:`。它经常在 `-observeValueForKeyPath:ofObject:change:context:`，或者 `-dealloc` 中被调用。

## Safe Unsubscribe with @try / @catch

如果你调用 `–removeObserver:forKeyPath:context:` 当这个对象没有被注册为观察者（因为它已经解注册了或者开始没有注册），抛出一个异常。有意思的是，没有一个内建的方式来检查对象是否注册。 这就会导致我们需要用一种相当不好的方式 `@try` 和一个没有处理的 `@catch`：

```Objective-C
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context
{
    if ([keyPath isEqualToString:NSStringFromSelector(@selector(isFinished))]) {
        if ([object isFinished]) {
          @try {
              [object removeObserver:self forKeyPath:NSStringFromSelector(@selector(isFinished))];
          }
          @catch (NSException * __unused exception) {}
        }
    }
}
```

# Automatic Property Notifications

Classes 可以选择自动退出 KVO 通过复写：`+automaticallyNotifiesObserversForKey:` 并且返回 `No`。

## DependentKey

有时候一个属性的值依赖于另一对象中的一个或多个属性，如果这些属性中任一属性的值发生变更，被依赖的属性值也应当为其变更进行标记。因此，objective-c 引入了依赖键。

实现依赖键首先需要重写对应键属性的`getter`和`setter`

```Objective-C
- (NSString *)information
{
    return [[[NSString alloc] initWithFormat:@"%d#%d", [_target grade], [_target age]] autorelease];
}

- (void)setInformation:(NSString *)theInformation
{
    NSArray * array = [theInformation componentsSeparatedByString:@"#"];
    [_target setGrade:[[array objectAtIndex:0] intValue]];
    [_target setAge:[[array objectAtIndex:1] intValue]];
}
```

然后需要在被观察者中实现以下方法

```Objective-C
//keyPathsForValuesAffecting后跟的是依赖键名
+ (NSSet *)keyPathsForValuesAffectingInformation
{
		//NSSet中是被依赖的键路径
    NSSet * keyPaths = [NSSet setWithObjects:@"target.age", @"target.grade", nil];
    return keyPaths;
}
```

也可以使用下面的方法实现

```Objective-C
+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key
{
  //需要获取父类的
    NSSet * keyPaths = [super keyPathsForValuesAffectingValueForKey:key];
    NSArray * moreKeyPaths = nil;
   
   if ([key isEqualToString:@"information"])
    {
        moreKeyPaths = [NSArray arrayWithObjects:@"target.age", @"target.grade", nil];
    }
    
    if (moreKeyPaths)
    {
        keyPaths = [keyPaths setByAddingObjectsFromArray:moreKeyPaths];
    }
    
    return keyPaths;
}
```

