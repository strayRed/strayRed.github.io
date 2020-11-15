---
title: NSUndoManager
author: strayRed
date: 2020-11-15 16:21:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

Cocoa 有一套简单强壮的 NSUndoManager API 管理撤销和重做。

默认地，每个应用的 window 都有一个 undo manager，每一个响应链条中的对象都可以管理一个自定义的 undo manager 来管理各自页面上本地操作的撤销和重做操作。`UITextField` 和 `UITextView` 用这个功能自动提供了文本编辑的撤销重做支持。每一个`UIView`对象继承自`UIResponder`类型，这个类定义了响应对象的接口并且处理事件。

`UIResponder`类声明了`undoManager`属性。当应用接收到`undo`事件，`UIResponder`搭建起响应者链并通过`undoManager`返回一个`NSUndoManager`类型的对象来找到这个响应者。**找到的第一个响应者将被用来处理`undo`或者`redo`操作**。

`NSUndoManager`实例支持重做操作，所以才能逆转操作。你可以认为这个管理器拥有两个栈。实际上，它管理两个栈，`undo`(撤销)栈和`redo`（重做）栈 - 对应`NSUndoManager`的私有属性`_undoStack`和`_redoStack`，里面存储着一些操作。

注册`undo`操作时，它会被添加到`undo`栈中。当调用`undo()`方法时管理器就会进行撤销，执行栈中的操作并把这个操作移动到`redo`栈中，这样你就可以重做它。当你拥有多个`undo`操作时，按照逆序来撤销和重做这些操作。你肯定不会将操作直接注册到`redo`栈中，实际上这根本无法实现。

你可以为`undo`操作设置一个级别，这指的是一个管理器可以在它的栈中存储多少`undo`操作。如果添加的`undo`操作数量超过了这个级别，最早加入的那个操作将会从栈中移除。

# undo operations

为了标明某个动作可以被撤销，需要在执行动作的时候注册一个 “撤销操作”。[撤销架构文档](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/UndoArchitecture/Articles/UndoManager.html#//apple_ref/doc/uid/20000205-CJBDJCCJ) 中定义 “undo operation” 为：

> 可以对一个对象进行逆向操作的方法，并且需要传递相应必需的参数。

共有两种撤销操作，简单的以 selector(或者block) 为基础的撤销和复杂的以 NSInvocation 为基础的撤销。

# Registering a Simple Undo Operation

注册一个简单的撤销操作，如果目标可进行撤销操作，调用其 `NSUndoManger -registerUndoWithTarget:selector:object:` 方法就可以了。目标不必是那个被改变的对象，通常是管理对象状态的工具或容器。同时调用 `NSUndoManager -setActionName:` 指定撤销操作的名称。

```Objective-C
- (void)updateScore:(NSNumber*)score {
  // 当有撤销操作发生时
  //因为 target设置的为self，当用户进行了撤销操作，就会将消息转发到target，并找到其 updateScore: 方法。
  // 传入 myMovie.score 这个值，并执行。
    [undoManager registerUndoWithTarget:self selector:@selector(updateScore:) object:myMovie.score];
    [undoManager setActionName:NSLocalizedString(@"actions.update", @"Update Score")];
    myMovie.score = score;
}
```



## Registering a Complex Undo Operation with NSInvocation

简单的撤销操作在某些使用场景下可能太粗糙了，比如说撤销某个动作需要不只一个参数。在这些情况下，我们可以使用 `NSInvocation` 来记录所需 selector 和相应参数。调用 `prepareWithInvocationTarget:` 记录哪些对象会接收哪些发生改变的消息。

```Objective-C
- (void)movePiece:(ChessPiece*)piece toRow:(NSUInteger)row column:(NSUInteger)column {
  // 使用 prepareWithInvocationTarget 可以显示调用方法，也就可以传入多个参数。
    [[undoManager prepareWithInvocationTarget:self] movePiece:piece ToRow:piece.row column:piece.column];
    [undoManager setActionName:NSLocalizedString(@"actions.move-piece", @"Move Piece")];
    piece.row = row;
    piece.column = column;
    [self updateChessboard];
}
```

你将会得到一个`NSUndoManagerProxy`类型对象，可以用它调用任何方法（但是只能调用目标对象遵守的那些协议方法，否则应用会抛出运行时异常）。注册之后代理对象将会在内部创建`NSInvocation`对象来记录你的操作，这个对象会在传入的目标对象执行`undo`操作时被调用。

值得强调的是，在注册过程中目标对象没有被持有，需要你去管理它。如果`undo`操作被调用而目标对象已经被销毁，就会产生运行时异常。

## Performing an Undo

一旦注册了撤销操作，就可以在需要时调用 `NSUndoManager -undo` 和 `NSUndoManager -redo`。

## Responding to the Shake Gesture on iOS

默认情况下，用户通过摇晃设备来触发撤销操作。如果一个 view controller 需要处理一个撤销请求，那么这个 view controller 必须：

1. 能成为 first responder
2. 一旦页面显示(view appears)，即变成 first responder
3. 一旦页面消失(view disappears)，即放弃 first responder

```Objective-C
@implementation ViewController

- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    [self becomeFirstResponder];
}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    [self resignFirstResponder];
}

- (BOOL)canBecomeFirstResponder {
    return YES;
}

// ...

@end
```

# Customizing the Undo Stack

## Grouping Actions Together

在同一 run loop 中被注册的所有的撤销操作可以被一同撤销，撤销组合允许同时进行许多undo和redo操作。

```Objective-C
- (void)readAndArchiveEmail:(Email*)email {
  // 使用 beginUndoGrouping 来进行组撤销
    [undoManager beginUndoGrouping];
    [self markEmail:email asRead:YES];
    [self archiveEmail:email];
    [undoManager setActionName:NSLocalizedString(@"actions.read-archive", @"Mark as Read and Archive")];
  // end
    [undoManager endUndoGrouping];
}

- (void)markEmail:(Email*)email asRead:(BOOL)isRead {
    [[undoManager prepareWithInvocationTarget:self] markEmail:email asRead:[email isRead]];
    [undoManager setActionName:NSLocalizedString(@"actions.read", @"Mark as Read")];
    email.read = isRead;
}

- (void)archiveEmail:(Email*)email {
    [[undoManager prepareWithInvocationTarget:self] moveEmail:email toFolder:@"Inbox"];
    [undoManager setActionName:NSLocalizedString(@"actions.archive", @"Archive")];
    [self moveEmail:email toFolder:@"All Mail"];
```

## Clearing the Stack

栈可以通过 `NSUndoManager -removeAllActions` 来清空或使用 `NSUndoManager -removeAllActionsWithTarget:` 在更细的力度上清空。

## Notifications

NSUndoManager 也有许多通知可以被监听，使用其**实例**和**通知名**来做区分。

- **NSUndoManagerCheckpointNotification **Posted whenever an NSUndoManager object opens or closes an undo group (except when it opens a top-level group) and when checking the redo stack in canRedo.
- **NSUndoManagerDidOpenUndoGroupNotification** Posted whenever an NSUndoManager object opens an undo group, which occurs in the implementation of the beginUndoGrouping method.
- **NSUndoManagerDidRedoChangeNotification** Posted just after an NSUndoManager object performs a redo operation (redo).
- **NSUndoManagerDidUndoChangeNotification** Posted just after an NSUndoManager object performs an undo operation.
  **NSUndoManagerWillCloseUndoGroupNotification** Posted before an NSUndoManager object closes an undo group, which occurs in the implementation of the endUndoGrouping method.
- **NSUndoManagerDidCloseUndoGroupNotification** Posted after an NSUndoManager object closes an undo group, which occurs in the implementation of the endUndoGrouping method.
- **NSUndoManagerWillRedoChangeNotification **Posted just before an NSUndoManager object performs a redo operation (redo).
- **NSUndoManagerWillUndoChangeNotification** Posted just before an NSUndoManager object performs an undo operation.

## Others

你可以为`undo`操作设置一个级别，这指的是一个管理器可以在它的栈中存储多少`undo`操作。如果添加的`undo`操作数量超过了这个级别，最早加入的那个操作将会从栈中移除。

你可以通过`canUndo`和`canRedo`来检查`undo`和`redo`栈的状态。这些状态很重要，你可能会基于这些栈的状态来更新 UI。

假设你已经设置了`undo`操作的级别并且`redo`栈中还有一个操作，如果`undo`操作超过了这个级别，那你就需要使用`canUndo`和`canRedo`来检测可用性。之所以要这样做，是因为在这种情况下`NSUndoManager`将会移除`redo`操作，因为它是历史操作中最新的`undo`操作。（校对注：这里确实很绕，大家可以类比一下编辑器的撤销和重做操作，如果你在撤销之后进行了新的改动，那之前撤销过的操作其实已经无法再被重做了，因此可以被直接删掉，从而把更多的空间留给`undo`操作。）