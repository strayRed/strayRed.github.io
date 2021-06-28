---
title: Efficient Interaction with Frameworks
author: strayRed
date: 2021-6-21 23:11:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

# Copy on Write Collections

```Objective-C
@implementation Container {
 NSMutableArray<Item *> *_elements;
}
- (NSArray<Item *> *)elements {
  // 现在对于可变集合类型的copy方法，Foundation已经实现了写时复制
 return [_elements copy];
}
@end	
```
```Objective-C
// Copies are safer
@property (copy) NSArray<Item *> *items;

(NSArray<Item *> *)items {
  //只要mutable的集合类型无脑用copy就完事了
 NSMutableArray *items = [[NSMutableArray alloc] init];
 [self buildItems:items];
 // The copy is completely safe here and also is nearly free so avoid bad things later
 //
return [items copy];
}

```

# Data

Swift4对`Data`进行了优化，现在它具有更好的性能（内存分配，边界，遍历效率）。

```Swift
// Leveraging Data, Don’t Believe the Lore

//var bytes: [UInt8] = [0xcf, 0xfa, 0xed, 0xfe]
var bytes = Data(bytes: [0xcf, 0xfa, 0xed, 0xfe])
//var buffer = malloc(250).assumingMemoryBound(to: UInt8.self)
//defer { free(buffer) }
var buffer = Data(count: 250)
//let header = buffer.subdata(in: buffer.startIndex..<buffer.startIndex.advanced(by: 4))
//使用slice更加高效，因为它无需进行复制
let header = buffer[..<buffer.startIndex.advanced(by: 4)]
```

# String Bridging

对于UILabel的`text`属性，将其从`NSString`转换为`String`需要进行 bridge，即需要将引用类型复制（copy）并封装为值类型，并让其具有值语义，对于不可变的`NSString`而言，当进行复制时，将经过优化，这并不会造成大量的开销，只会增加引用计数。

```Swift
//这并不会造成大量开销
var text = label.text	
```
不过对于`NSTextStorage`而言，因为其本身为`NSMutableString`的子类，调用其`string`属性获取到的实例也仍然是指向的`mutableSting`本身。所以对这一可变字符串桥接`String`过程中的复制操作（copy）就会消耗大量资源（也是因为`UITextView`的字符串较长）。
```Swift
//这会消耗大量资源
var text = textView.textStorage.string
//这是目前比较好的做法，因为NSMutableString在Swift中没有对应的类型，所以不会产生桥接，也就不会进行复制操作。
var text = textView.textStorage.mutableString
```

造成上述问题的根本原因是因为Swift的值语义特性与`NSTextStorage`的设计不匹配所导致的，因此需要使用引用类型针对这类字符串进行性能管理。

# Text layout and rendering

- Use standard label controls

- Use modern layout practices（auto layout）

  对于文本信息而言，自动布局会缓存大小以此提升性能。

-  Set rendering attributes for attributed strings

   显式地设置部分Attributes，也可以优化部分性能。

   ```Swift
   let attributes: [NSAttributedStringKey : Any] = [
    .font: UIFont.systemFont(ofSize:.systemFontSize),
    .paragraphStyle: NSParagraphStyle.default,
    .foregroundColor: UIColor.darkText]
   let myString = NSAttributedString(string: "Hello", attributes: attributes)
   
   // Only do this if you’re absolutely sure your text doesn’t have mixed writing directions
   var myParagraphStyle = NSMutableParagraphStyle()
   myParagraphStyle.baseWritingDirection = .leftToRight
   myParagraphStyle.alignment = .left
   
   // Only do this if you’re sure your text doesn’t require wrapping
   var myParagraphStyle = NSMutableParagraphStyle()
   myParagraphStyle.lineBreakMode = .byClipping
   ```

   

