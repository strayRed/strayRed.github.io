---
title: dictionary optimizations
author: strayRed
date: 2021-3-23 21:12:00 +0800
categories: [ios, swift]
tags: [ios, swift]
---

Copy on write is a tricky thing, and you need to think carefully about how many things are sharing a structure that you're trying to modify. 

对于Swift的字典类型，直接通过下标访问对其值语义元素进行操作，系统会使用一个临时变量对该元素进行修改，并更新到字典中，这个过程中该元素会存在两个拷贝，因此会触发写时复制。

所以需要将下列代码

```Swift
if countToColorMap[colorCount] != nil {
    countToColorMap[colorCount]?.append(CountedColor(color: color as! UIColor, colorCount: colorCount))
} else {
    countToColorMap[colorCount] = [CountedColor(color: color as! UIColor, colorCount: colorCount)]
}
```

替换为

```Swift
//删除掉元字典中的引用，确保元素修改时只会存在一个引用
var countForColor = countToColorMap.removeValue(forKey: colorCount) ?? []
countForColor.append(CountedColor(color: color as! UIColor, colorCount: colorCount))
countToColorMap[colorCount] = countForColor
```

与 C 语言不同，苹果对 Swift 中的结构体有着优化，使其能够存储在栈上，进而加快访问速度与增加操作的安全性。

如果你的结构体只由其他结构体组成，那编译器可以确保不可变性。同样地，当使用结构体时，编译器也可以生成非常快的代码。举个例子，对一个只含有结构体的数组进行操作的效率，通常要比对一个含有对象的数组进行操作的效率高得多。这是因为结构体通常要更直接：值是直接存储在数组的内存中的。而对象的数组中包含的只是对象的引用。最后，在很多情况下，编译器可以将结构体放到栈上，而不用放在堆里。

```Swift
func uniqueIntegerProvider() -> AnyIterator<Int> {
    var i = 0
    return AnyIterator {
        i += 1
        return i
    }
}
```

Swift 的结构体一般被存储在栈上，而非堆上。不过这其实是一种优化：默认情况下结构体是存储在堆上的，但是在绝大多数时候，这个优化会生效，并将结构体存储到栈上。**当结构体变量被一个函数闭合的时候，优化将不再生效，此时这个结构体将存储在堆上**。因为变量 i 被函数闭合了，所以结构体将存在于堆上。这样一来，就算 uniqueIntegerProvider 退出了作用域，i 也将继续存在。与此相似，**如果结构体太大，它也会被存储在堆上**。