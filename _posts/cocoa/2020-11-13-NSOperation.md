---
title: NSOperation
author: strayRed
date: 2020-11-13 14:48:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

`NSOperation`表示了一个独立的计算单元。作为一个抽象类，它给了它的子类一个十分有用而且线程安全的方式来建立状态、优先级、依赖性和取消等的模型。如果开发者不愿意自定义`NSOperation`子类，Foundation提供了`NSBlockOperation`和`NSInvocationOperation`两个实体类。

- **NSBlockOperation**：抽象类`NSOperation`的子类，使用Block的形式添加执行的内容。
- **NSInvocationOperation**：同为`NSOperation`的子类，使用 Target-Action的模式，将任务执行的内容转发给 Target 对应的Seletor，因为使用了运行时机制，在Swift被删除了。

# NSOperationQueue

`NSOperationQueue`控制操作的并发执行。它充当一个优先级队列，因此操作以大致先进先出的方式执行，但是具有更高的优先级(`NSOperation.queuePriority`)的任务要比低优先级的任务先执行。`NSOperationQueue`还可以使用`maxConcurrentOperationCount`属性限制在任何给定时刻执行的最大并发操作数。（`NSOperationQueue`类似于GCD queue，尽管这是一个私有实现。）

`NSOperation`实例一旦被加入到队列，就会被`NSOperationQueue`强引用。直到任务结束。

## KVO

可以通过KVO来观测以下属性。

- **operations** - read-only 
- **operationCount** - read-only 
- **maxConcurrentOperationCount** - readable and writable 
- **suspended** - readable and writable 
- **name** - readable and writable

## Initialization

使用`NSOperationQueue`的类方法进行初始化。`+(NSOperationQueue *)mainQueue`获取主线程队列。`+(NSOperationQueue *)currentQueue`创建一个并发队列。

## Managing

- **qualityOfService**：为整个队列设置QoS。
- **maxConcurrentOperationCount**：最大并发的任务数。设置为1就为一个串行队列。

# State

`NSOperation`包含了一个十分优雅的状态机来描述每一个操作的执行。

> isReady` → `isExecuting` → `isFinished

为了替代不那么清晰的`state`属性，状态直接由上面那些keypath的KVO通知决定，也就是说，当一个操作在准备好被执行的时候，它发送了一个KVO通知给`isReady`的keypath，让这个keypath对应的属性`isReady`在被访问的时候返回`YES`。

每一个属性对于其他的属性必须是互相独立不同的，也就是同时只可能有一个属性返回`YES`，从而才能维护一个连续的状态：

- `isReady`: 返回 `YES` 表示操作已经准备好被执行, 如果返回`NO`则说明还有其他没有先前的相关步骤没有完成。
- `isExecuting`: 返回`YES`表示操作正在执行，反之则没在执行。
- `isFinished` : 返回`YES`表示操作执行成功或者被取消了，`NSOperationQueue`只有当它管理的所有操作的`isFinished`属性全标为`YES`以后操作才停止出列，也就是队列停止运行，所以正确实现这个方法对于避免死锁很关键。

# Cancellation

早些取消那些没必要的操作是十分有用的。取消的原因可能包括用户的明确操作或者某个相关的操作失败。

与之前的执行状态类似，当`NSOperation`的`-cancel`状态调用的时候会通过KVO通知`isCancelled`的keypath来修改`isCancelled`属性的返回值，`NSOperation`需要尽快地清理一些内部细节，而后到达一个合适的最终状态。特别的，这个时候`isCancelled`和`isFinished`的值将是YES，而`isExecuting`的值则为NO。

# Priority

不可能所有的操作都是一样重要，通过以下的顺序设置`queuePriority`属性可以加快或者推迟操作的执行：

- `NSOperationQueuePriorityVeryHigh`
- `NSOperationQueuePriorityHigh`
- `NSOperationQueuePriorityNormal`
- `NSOperationQueuePriorityLow`
- `NSOperationQueuePriorityVeryLow`

此外，有些操作还可以指定`threadPriority`的值，它的取值范围可以从`0.0`到`1.0`，`1.0`代表最高的优先级。鉴于`queuePriority`属性决定了操作执行的顺序，`threadPriority`则指定了当操作开始执行以后的CPU计算能力的分配。`threadPriority`在iOS8已经被废弃，由QoS取代。

## NSOperationQueuePriority

```Objective-C
typedef NS_ENUM(NSInteger, NSOperationQueuePriority) {
    NSOperationQueuePriorityVeryLow = -8L,
    NSOperationQueuePriorityLow = -4L,
    NSOperationQueuePriorityNormal = 0,
    NSOperationQueuePriorityHigh = 4,
    NSOperationQueuePriorityVeryHigh = 8
};
```

# Quality of Service

`Quality of Service` 是  iOS 8 和 OS X Yosemite 引入的新概念。它为调度系统资源创建了一致的高级语义，因此 XPC 和 NSOperation 都引人了这个抽象API。

对于NSOperation，threadPriority属性已被弃用，取而代之的是这个新的qualityOfService属性。

Service的等级决定了一个 Operation 在系统层面上能分配到多少资源（CPU，网络，本地IO），更高的QoS意味着将提供更多的资源来更快地执行 Operation。

## NSQualityOfService

```Objective-C
typedef NS_ENUM(NSInteger, NSQualityOfService) {
    NSQualityOfServiceUserInteractive = 0x21,    
    NSQualityOfServiceUserInitiated = 0x19,    
    NSQualityOfServiceUtility = 0x11,    
    NSQualityOfServiceBackground = 0x09,
    NSQualityOfServiceDefault = -1
} NS_ENUM_AVAILABLE(10_10, 8_0);
```

- **.UserInteractive**：UserInteractiveQoS用于直接涉及到提供交互式UI的工作，例如处理事件或将图形绘制到屏幕上。

- **.UserInitiated**：UserInitiatedQoS于执行用户明确请求的工作，并且必须立即显示结果，以便允许进一步的用户交互。例如，在用户在邮件列表中选择电子邮件后加载它。

- **.Utility**：UtilityQoS：用于执行用户不太可能立即等待结果的工作。这项工作可能是由用户请求或自动启动的，不会阻止用户进一步交互，通常在用户可见的时间范围内操作，并且可以通过进度指示器向用户指示其进度。这项工作将以一种节能的方式运行，以配合在资源受限时更高级的QoS任务的工作。例如，定期内容更新或大容量文件操作（如媒体导入）
- **.Backgroud**：BackgroundQoS用于非用户启动或不可见的工作。一般来说，用户甚至不知道这项工作正在进行，它将以最有效的方式运行，同时给予Service等级更高的任务更多的资源。例如，预取内容、搜索索引、备份以及与外部系统同步数据。

- **.Default**：DefaultQoS表示缺少QoS信息。只要可能，将从其他来源推断出QoS信息。如果这样的推断是不可能的，那么将使用UserInitiated和Utility之间的QoS。

```Objective-C
NSOperation *backgroundOperation = [[NSOperation alloc] init];
backgroundOperation.queuePriority = NSOperationQueuePriorityLow;
backgroundOperation.qualityOfService = NSOperationQualityOfServiceBackground;
// 在主线程队列上添加了低优先级低服务等级的任务
[[NSOperationQueue mainQueue] addOperation:backgroundOperation];
```

# Asynchronous Operations

iOS 8/OS X Yosemite的另一个变化是不提倡使用 `NSOperation` 的 `concurrent` 属性，转而使用新的`Asynchronous`属性。新的asynchronous属性清除了concurrent的语义问题，现在是NSOperation应该在main中同步执行还是异步执行的唯一决定。

# Dependencies

根据你应用的复杂度不同，将大任务再分成一系列子任务一般都是很有意义的，而你能通过`NSOperation`的依赖性实现。

比如说，对于服务器下载并压缩一张图片的整个过程，你可能会将这个整个过程分为两个操作（可能你还会用到这个网络子过程再去下载另一张图片，然后用压缩子过程去压缩磁盘上的图片）。显然图片需要等到下载完成之后才能被调整尺寸，所以我们定义网络子操作是压缩子操作的依赖，通过代码来说就是：

```Objective-C
[resizingOperation addDependency:networkingOperation];
[operationQueue addOperation:networkingOperation];
[operationQueue addOperation:resizingOperation];
```

确保不要意外地创建依赖循环，像A依赖B，B又依赖A，这也会导致死锁。

# completionBlock

如果一个 `NSOperation` 结束，那么它会执行其 `completionBlock` 一次（exactly once）。我们通常可以在这个地方与UI层进行交互。

```Objective-C
NSOperation *operation = ...;
operation.completionBlock = ^{
    NSLog("Completed");
};

[[NSOperationQueue mainQueue] addOperation:operation];
```

# When to Use Grand Central Dispatch

对于一次性计算，或者只是提高现有方法的效率，使用轻量级GCD调度通常比使用NSOperation更方便。

# When to Use NSOperation

NSOperation可以用一组依赖任务并按特定的队列优先级和服务等级进行调度。与GCD队列上使用Block调度不同，NSOperation可以被取消并使用KVO查询其操作状态。通过子类化，NSOperation可以将其工作结果关联到自己身上，以备将来使用。