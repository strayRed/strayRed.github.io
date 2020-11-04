---
title: Methods Dispatch
author: strayRed
date: 2020-11-03 16:19:00 +0800
categories: [ios, objective-c]
tags: [ios, objective-c, swift]
---

# Message Dispatch and the Objective-C Runtime

`Objective-C`程序的实现就是一系列对象通过方法（消息）来进行交互，使用下面这个语法：

```ObjectiveC
[someObject aMethod:withAnArgument];
```

此外，我们也可以使用`objc_msgSend`这个c方法来进行消息的发送。

```ObjectiveC
objc_msgSend(object, @selector(message), withAnArgument);
```

对于`SEL`对应的`IMP`指针指向的方法，同样，含有两个隐藏的参数，`id`和`SEL`类型，它们的值为当前消息的接受者和当前消息的`selector`。

`Objective-C`的方法调度是动态的，每一个对象（类也是对象）都维护着一个`dispatch table`为了在程序运行时进行方法的查找。这个表的每一个条目都是一个`Method`结构体，它将`SEL`到`IMP`进行了映射。当一个对象收到消息时，它首先会查找它的类的调度表（分类的相同`seletor`会覆盖本类），找到就直接调用对应的实现，未找到就会调用相关的转发函数，再未找到就会查询类的父类，直到查找到根类（`NSObject`或 `NSObject元类`），最后没找到就会crash。

对于`Objective-C`，函数派发都是基于消息派发的，这种机制极具动态性，既可以通过`swizzling`修改函数的实现，也可以通过 `isa-swizzling`修改对象。整个派发过程中的方法都会被缓存，从而提高效率。

# Table Dispatch

与 `Message Dispatch`类似，`Table Dispatch`同样也是每一个类维护者一个函数表，大部分语言把这个称为 `virtual table`(虚函数表)，Swift 里称为 `witness table`，面记录着类所有的函数，用指针标识，。同时子类在创建时也会创建一个函数表，如果函数是 `override` 的，则使用一个新的指针，用于区分父类中相同函数的指针。如果这个函数是父类中有且没被 `override`的，则存储的就是原先的（父类的）指针。

与 `Message Dispatch`不同的是，`Table Dispatch`只会查找当前类的方法，因为继承关系，子类会直接继承父类的函数指针（如果这个函数没有被重写），调用父类的方法就会直接跳转，而不是在父类的方法表中进行查找。

> 这是因为`Objective-C`是一门动态性很强的语言，在运行时方法的`IMP`指针可能会被替换，所以OC使用selector指针而不是方法指针作为唯一标示。如此，子类是无法直接继承父类的函数指针的，那么只能父类和子类分别维护自己的函数表，在方法查找的时候逐一通过 selector 指针进行比对。
>
> 。对于一些非动态语言，如Swift，因为其同样拥有OOP的特性，对于部分类和协议方法仍然是使用了函数表派发，但是其函数的指针在编译时就能够确定，我们可以使用函数指针作为唯一标示，并且子类可以继承父类的函数表，从而提高效率。

# Direct Dispatch

直接派发也叫静态派发。在直接派发中，编译器直接找到相关指令的位置。当函数调用时，系统直接跳转到函数的内存地址执行操作。这样的好处就是执行快，同时允许编译器能够执行例如内联等优化。事实上，编译期在编译阶段为了能够获取最大的性能提升，都尽量将函数静态化。不过静态派发是有条件的，方法内部的代码必须对编译器透明，并且在运行时不能被更改，这样编译器才能帮助我们。

Swift 中的值类型不能被继承，也就是说值类型的方法实现不能被修改或者被复写，因此值类型的方法满足静态派发的要求。

```Swift
protocol Noisy {
     func makeNoise() -> Int  //函数表派发
}

extension Noisy {
    func makeNoise() -> Int { return 0 }  //函数表派发
    func isAnnoying() -> Bool { return true}  //直接派发
}

class Animal: Noisy {
    func makeNoise() -> Int { return 1 } //函数表派发
    func isAnnoying() -> Bool { return false } //函数表派发
    @objc func sleep() {} //函数表派发
}

extension Animal {
    func eat() {} //直接派发
    @objc func getWild() {} //消息派发
}

struct rectangle {
    func getArea() { } //直接派发
}
```



# Direct Dispatch with a C Function

在`Objective-C`中，我们也可以使用C函数来达到直接派发的目的。

```ObjectiveC
@interface MyClass: NSObject
- (void)dynamicMethod;
@end
```

声明一个静态C函数，传入原本方法的调用者。

```C
static void directFunction(MyClass *__unsafe_unretained object);
```

```ObjectiveC
MyClass *object = [[[MyClass] alloc] init];

// Dynamic Dispatch
[object dynamicMethod];

// Direct Dispatch
directFunction(object);
```

# Direct Methods

用C写的直接方法看起来会像一个便利方法，但是它确实具有C方法的特性，也就是说当这个方法被调用时，这个方法的实现会被直接调用，而不是经由`objc_msgSend`来调用。

有了这个 LLVM 的新特性（*Xcode 11 above*），开发者可以自由选择Objective-C方法的调度方式。

## objc_direct, @property(direct), and objc_direct_members

更方便的是，我们可以使用`objc_direct` 关键字来标示方法，让它可以被直接派发。对于属性，可以为其添加`direct`关键字，让其`getter`和`stter`函数变为直接派发。

```ObjectiveC
@interface MyClass: NSObject
@property(nonatomic) BOOL dynamicProperty;
@property(nonatomic, direct) BOOL directProperty;

- (void)dynamicMethod;
- (void)directMethod __attribute__((objc_direct));
@end
```

> 因为加入了`direct`，`@property`的关键字达到了16个
>
> - getter and setter
>- readwrite and readonly,
> - atomic and nonatomic
>- weak, strong, copy, retain, and unsafe_unretained
> - nullable, nonnullable, and null_resettable
>- class

此外，我们也可以在分类和类扩展前使用`objc_direct_members`让所有声明的属性和方法实现动态派发。

```ObjectiveC
__attribute__((objc_direct_members))
@interface MyClass ()
@property (nonatomic) BOOL directExtensionProperty;
- (void)directExtensionMethod;
@end
```

在`@implementation`之前使用`__attribute__((objc_direct_members))`也有同样的效果，不过仅限于在`@implementation`代码块中声明的方法。

```ObjectiveC
__attribute__((objc_direct_members))
@implementation MyClass
- (BOOL)directProperty {…}
- (void)dynamicMethod {…}
- (void)directMethod {…}
- (void)directExtensionMethod {…}
- (void)directImplementationMethod {…}
@end
```

⚠️不能在主要的`@interface`块（也就是声明类的`@interface`）中使用`objc_direct_members`。

⚠️动态派发的方法不能在子类中被重写为直接派发，直接派发方法不能够被重写。协议的方法或者属性也不能做直接派发的约束，也不能将协议的方法在类中声明为直接派发。

# Performance

但是，在大多数的情况下，对Objective-C方法使用直接派发并不会明显的性能优势。

事实证明，`objc_msgSend`速度惊人。由于先进的缓存、广泛的低层优化和现代处理器固有的性能特性，`objc_msgSend`的开销非常低。

# Hidden Motives

当Objective-C方法被标记为`direct`的时候，它的具体实现其实是被隐藏了，直接方法只能在同一模块（或者 [linkage unit](https://clang.llvm.org/docs/LTOVisibility.html)）中被调用），这些方法也不会出现在`Objective-C runtime`中。

隐藏方法的可见性有两个好处

- 更小的二进制文件大小

- 不会引起外部调用

如果没有外部接口，同时也不能在运行时被观测到，那么直接方法实际上就是私有方法。

> 这里也反向说明了为什么Objective-C方法没有`privte`或者`public`这样的访问控制修饰，因为`Objective-C`方法都是动态派发，它们不具有隐藏可见性的特性。
>

虽然隐藏可见性的特性可以被苹果用于防治`swizzling`和调用私有API，实际上它最大的作用是减少代码大小。使用直接派发后编译的二进制文件的文本会减少5-10%左右。

对于第三方SDK开发而言，是一个很不错的优化，开发者可以使用`objc_direct `和  `direct` 注释私有方法和属性，使用`objc_direct_members`注释私有类。