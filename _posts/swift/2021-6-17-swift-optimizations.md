---
title: swift optimizations
author: strayRed
date: 2021-6-17 19:02:00 +0800
categories: [ios, swift]
tags: [ios, swift]
---

## Allocation

### Stack

简单的数据结构，通过栈指针的增减来控制内存的释放与分配。

Swift中所有值类型（不含类似`String`的值语义类型），其内存都是在栈上进行分配的，高效且快捷。对值类型使用赋值语句虽然不会共享状态，但是会直接造成栈上内存的重复分配（非临时变量需要考虑这个问题）。

### Heap

比Stack更为复杂的数据结构，查找可用的内存块来进行分配，重新插入内存块来进行释放，线程安全（锁消耗）。

引用类型则是在堆上分配内存，效率会相对低下。对于临时变量而言，其用于存储的栈中存放的是指向对应堆中内存块的指针，对引用类型使用赋值语句会直接传递指针，这虽然会共享实例的状态，但是因为两者都指向同一内存，所以不会造成堆内存的重复分配。

### Conclusion

对于频繁创建的临时变量而言，使用值类型往往比引用类型更加高效。而对于那些需要频繁赋值的类型，使用引用类型更加正确。
```Swift
enum Color { case blue, green, gray }
enum Orientation { case left, right }
enum Tail { case none, tail, bubble }
var cache = [Attributes : UIImage]()

struct Attributes : Hashable {
 var color: Color
 var orientation: Orientation
 var tail: Tail
}

func makeBalloon(_ color: Color, orientation: Orientation, tail: Tail) -> UIImage {
  // 使用自定义的值类型Attributes代替String作为Key可以有效地避免从堆上分配内存。
 let key = Attributes(color: color, orientation: orientation, tail: tail)
 if let image = cache[key] {
 return image
 }
 …
}
```

## Reference counting

ARC并不是单纯地对整数的增减，它还会涉及到对象的层级关系（属性之间的引用），线程安全（锁的性能损耗）。

只有引用类型会涉及到引用计数，而值类型不会，正如上文提及，对值类型赋值只会copy它的所有内容而不是内存指针。

不过，对于复合类型，即值类型中包含引用类型的属性，那么对这样的值类型进行赋值操作时，会根据其内部的**引用类型属性的数量**来增加引用计数。

### Conclusion

在赋值过程中，尽可能地避免引用计数的增加。

- 对于值类型中的多个引用类型，可以使用一个额外的引用类型来对它们进行包装，从而减少赋值过程中的多次引用计数的增加。

- 也可以使用值类型来替换掉引用类型，从而从根本上防止引用计数的增减。

```Swift
struct Attachment {
 let fileURL: URL
  //使用UUID替换String
 let uuid: UUID
  //使用MimeType替换String
 let mimeType: MimeType
 init?(fileURL: URL, uuid: UUID, mimeType: String) {
 guard let mimeType = MimeType(rawValue: mimeType)
 else { return nil }
 self.fileURL = fileURL
 self.uuid = uuid
 self.mimeType = mimeType
 }
}	
```



## Method dispatch

## Static

在运行时直接跳转到具体方法实现，可以有效地实现内联和其他优化方式。

在final引用类型和值类型中，其自身的方法都是静态派发的，协议类型中未声明的扩展方法也是静态派发的。

## Dynamic

在运行时在表中查找方法实现，然后跳转到具体实现，阻止了内联和其他优化方式。
### Class

#### Inheritance-Based Polymorphism

继承是通过每一个类类型的`V-Table`来实现方法调用的多态。

#### Polymorphism Through Reference Semantics

因为引用类型在栈上只存储指针，所以对于数组这样连续的内存地址，其只需要存储指针就能够保证类型的多态性。

### Protocol Type

#### Polymorphism without inheritance or reference semantics

协议类型的方法多态性是通过`Protocol Witness Table`来实现的，与引用类型的`V-Table`类似，每一个实现了具体协议的具体类型，系统都会创建一个pwt，其包含了该类型的方法相关信息，在实际执行时就会查找对应的table中的具体实现。

而协议类型的类型多态性是通过 Existential Container（存在容器来实现的），任何一个可以单独作为类型接口的协议类型（不含泛型），其在内存中都是通过存在容器进行保存的。

#### Existential Container

每一个存在容器的结构都是相同的，前三个word被称作`inline value buffer`，如果该具体类型中保存的值（值类型）不超过其容量，被称为小值（small value）那么它就会被内联地存储到`buffer`中。而对于超出容量的大值（large value），它们就会被存储在堆上，`buffer`中则只会存储指向其内存的指针。因此在使用Protocol Type作为接口时，也有很可能涉及到堆内存的分配。

另一方面，Swift对于值的声明周期管理采用了`The Value Witness Table`，每一个类型都存在对应的值目击表，通过该表可以对任意类型进行`Allocation`,` Copy`,` Destruction`。

在存在容器的`value buffer`之后，就会存储该类型的`Value Witness Table`指针，再之后就会存储`Protocol Witness Table`指针。

```c
[]    
[]    inline value buffer
[]

[]    The Value Witness Table pointer(实际内容存储在静态存储区)，在参数传递或局部变量的使用中，这个表主要用于形参或者临时变量的创建与销毁
[]    The Protocol Witness Table pointer(实际内容存储在静态存储区)，这个表主要用于在运行时查找方法的实现
```

#### Protocol Type Stored Properties

如果直接通过`Protocol Type`来作为属性进行存储值，需要考虑到值类型的`large value`会引发堆内存分配，而且因为值类型的特性，`large value`所需要的内存会随着该实例的赋值语句的调用而继续在堆上分配。

```Swift
struct Pair { Supports dynamic polymorphism
 init(_ f: Drawable, _ s: Drawable) {
 first = f ; second = s
 }
 var first: Drawable
 var second: Drawable
}
// 下列代码会进行4次堆内存分配
let aLine = Line(1.0, 1.0, 1.0, 3.0)
let pair = Pair(aLine, aLine)
let copy = pair
```

#### Indirect Storage with Copy-on-Write

为了解决上述问题，应该使用引用类型而非值类型来存储`large value`，这样各个存在容器的`value buffer`中都会存储相同额指针，从而减少内存分配的次数。但是这样会使实例之间共享状态，因此需要为该引用类型实现写时复制的特性。

```Swift
class LineStorage { var x1, y1, x2, y2: Double }
struct Line : Drawable {
 var storage : LineStorage
 init() { storage = LineStorage(Point(), Point()) }
 func draw() { … }
 mutating func move() {
 if !isUniquelyReferencedNonObjc(&storage) {
 storage = LineStorage(storage)
 }
 storage.start = ...
 }
}
```

### Generic Code

对于协议类型约束的泛型，在通过该泛型实例调用其方法时，swift可以将泛型绑定到具体类型上，所以这个参数传递不会使用存在容器，而是通过vwt和pwt来实现临时变量的创建与方法的调用。不过swift仍然会在stack上创建一个`value buffer`同样通过 three words来保存泛型实例的值，大值同样也会被存储在堆上。

虽然看起来和使用`ProtocolType`时并无差别，但是对于泛型这种`Static Polymorphism`，Swift编译器会存在特殊的优化（Specialization of Generics）。

#### Specialization of Generics

Swift会为每一种泛型可能会传入的具体类型创建一个专门的函数实现，以此直接根据具体类型派发函数。

在使用泛型约束来存储值时，因为类型不能在运行时改变，这意味着Swift可以在为类型**内联**地分配内存，即这个泛型实例会当作一个具体类型被存储，如果它与保存它的类型都是值类型，那么这个值的赋值过程**不会**涉及到任何堆内存的分配。

### Summary

对于值类型的泛型优化，有以下几个特性。

- No heap allocation on copying

- No reference counting

- Static method dispatch

对于引用类型的泛型优化，有以下几个特性。

- Heap allocation on creating an instance 
- Reference counting 
- Dynamic method dispatch through V-Table

对于未经过泛型优化的小值，有以下几个特性。

- No heap allocation: value fits in Value Buffer 
- No reference counting
- Dynamic dispatch through Protocol Witness Table 

对于未经过泛型优化的大值，有以下几个特性。

- Heap allocation (use indirect storage as a workaround) 
- Reference counting if value contains references 
- Dynamic dispatch through Protocol Witness Table

Choose fitting abstraction with the least dynamic runtime type requirements 

- struct types: value semantics 
- class types: identity or OOP style polymorphism 
- Generics: static polymorphism 
- Protocol types: dynamic polymorphism Use indirect storage to deal with large values