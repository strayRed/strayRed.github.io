---
title: Method Swizzling
author: strayRed
date: 2020-11-02 20:31:00 +0800
categories: [ios, objective-c]
tags: [ios, objective-c]
---

`Objective-C` 中的方法其实是`C`的一个结构体，类型为`Method`，它是`objc_method`结构体的类型别名。

```C
struct objc_method {
     SEL method_name         OBJC2_UNAVAILABLE;
     char *method_types      OBJC2_UNAVAILABLE;
     IMP method_imp          OBJC2_UNAVAILABLE;
}
```
我们可以使用`objc/runtime`的 `class_getClassMethod`方法来获取一个`Method`。还可对对象的方法进行添加或者替换。

```ObjectiveC
//获取类方法（从元类中获取）
Method class_getClassMethod(Class aClass, SEL aSelector);
//获取实例方法（从类中获取）
Method class_getInstanceMethod(Class aClass, SEL aSelector);

//添加方法
Bool class_addMethod(Class aClass, SEL aSelector, IMP aIMP, char * methodTypes);
//替换方法
Bool class_replaceMethod(Class aClass, SEL aSelector, IMP aIMP, char * methodTypes);
//替换方法实现
method_exchangeImplementations(Method methodA, Method methodB);
```

- method_name：方法名，`SEL` 是 `objc_selector`类型的结构体，广义上来说，它可以完全被理解为一个 char * 类型，也就是方法名的字符串，我们可以使用 `@selector(...)`将一个`NSString`转换为相应的`SEL`。

- method_types：方法参数和返回值的[编码方式](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)，它是一个C的字符串类型，数据类型都根据特定的编码方式做了转换，字符串的顺序是*返回值类型编码* + *参数类型编码*。对于方法的实现`IMP`而言，它的头两个参数是固定的，同时也是被隐藏的，一个是`id`类型，任意对象，也就是消息的接受者，二是`SEL`类型，所以对于一个没有返回值也没有参数的方法而言，它的编码为`v@:`，v = void，@ = Class，: = SEL。可以使用`@ecode(...)`传入一个类型的实例来获取对应的类型编码，还可以使用 `method_getTypeEncoding`来获取指定`Method`的编码方式。
```Objective-C
  char * method_getTypeEncoding(Method aMethod);
```
- method_imp：它是一个函数指针，指向实际的方法实现，方法有两个隐藏的参数，分别为消息的接受者和方法名。所以`IMP`类型被定义为 `id (*IMP)(id, SEL, …)`。可以使用`method_getImplementation`来获取指定`Method`的`IMP`，还可以使用`method_setImplementation`来替换掉指定`Method`的`IMP`。

```Objective-C
  IMP method_getImplementation(Method aMethod);
  //RETURN:- The original Imp
  IMP method_setImplementation(Method method, IMP imp);
```
 # The incorrect way to swizzle
一般而言，我们会使用下面的方式进行 `swizzle`
```ObjectiveC
@implementation UIViewController (Tracking)
//这个方法是在同一个类中，使用swizzle_viewWillAppear替换viewWillAppear

+ (void)load {
   static dispatch_once_t onceToken;
   dispatch_once(&onceToken, ^{
       Class class = [self class];
       SEL originalSelector = @selector(viewWillAppear:);
       SEL swizzledSelector = @selector(swizzle_viewWillAppear:);
       Method originalMethod = class_getInstanceMethod(class, originalSelector);
       Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
       
   
   //这个代码是给当前calss添加一个viewWillAppear:方法，并用swizzled方法实现替换。
   //比如有些子类没有实现viewWillAppear:，这样也可以达到swizzled的目的
   
   BOOL didAddMethod =
       class_addMethod(class,
           originalSelector,
           method_getImplementation(swizzledMethod),
           method_getTypeEncoding(swizzledMethod));
   
   if (didAddMethod) {
   //添加成功后就将xxx_viewWillAppear的实现换成原viewWillAppear:的实现
       class_replaceMethod(class,
           swizzledSelector,
           method_getImplementation(originalMethod),
           method_getTypeEncoding(originalMethod));
  
      } else {
       //没有成功添加就说明该类中两个方法都存在了，直接调用exchangeImplementations交换
          method_exchangeImplementations(originalMethod, swizzledMethod);
   }
});
   }
   #pragma mark - Method Swizzling

//这里的调用并不会引起死循环。
//因为这个方法的实现已经变成了viewWillAppear:
//viewWillAppear的实现变成了这里的内容，即原来的 viewWillAppear代码 + 新加入的代码。
- (void)swizzle_viewWillAppear:(BOOL)animated {
  [self swizzle_viewWillAppear:animated];
   NSLog(@"viewWillAppear: %@", self);
}
@end
```

然而直接使用`method_exchangeImplementations`进行`IMP`的交换会存在隐患。

`IMP`交换完成后，原来的 `viewWillAppear:` 的`Method`结构体会是这样：

```ObjectiveC
Method viewWillAppear { //this is the original Method struct. we want to switch this one with
             //our replacement method
     SEL method_name = @selector(viewWillAppear:)
     char *method_types = "B@:" //returns bool, params id(self),selector(_cmd)
     IMP method_imp = 0x1234AABA (MyBundle`[UIViewController swizzle_viewWillAppear])
 }
```


而交换后的`swizzle_viewWillAppear:`的`Method`结构体`则会是这样：

```ObjectiveC
Method swizzle_viewWillAppear { //this is the swizzle Method struct. We want this method //executed when [UIViewController viewWillAppear] is called
     SEL method_name = @selector(swizzle_viewWillAppear:)
     char *method_types = "B@:"
     IMP method_imp = 0x000FFFF (MyBundle`[UIViewController viewWillAppear])
 }
```

如果我们想调用原来的`viewWillAppear:`的 `method_imp`，那么只能调用`swizzle_viewWillAppear:`方法，但是我们只是交换了两个方法的`IMP`指针，并没有交换`method_name`，也就是`SEL`，**结果会导致传入原来`viewWillAppear`的`method_imp`的`_cmd`是当前结构体的`method_name`，也就是`@selector(swizzle_viewWillAppear:)`**。如果原来的方法实现依赖了原方法名的指针，那么就会产生错误。

# The correct way to swizzle

不是创建一个`Objective-C`方法`-[(void) swizzle_originalMethodName]`，而是根据`IMP`的定义创建一个相同签名的`C`方法作为新的`IMP`指针进行替换。

```C
void __Swizzle_OriginalMethodName(id self, SEL _cmd)
 {
      //code
 }
```

这样就可以将这个方法指针转换为`IMP`

```ObjectiveC
IMP swizzleImp = (IMP)__Swizzle_OriginalMethodName;
```

下面是完整的过程

```ObjectiveC
@interface SwizzleExampleClass : NSObject
 - (void) swizzleExample;
 - (int) originalMethod;
 @end

//使用一个静态变量存储原方法的指针
static IMP __original_Method_Imp;
//这是用于替换IMP指针的c方法
int _replacement_Method(id self, SEL _cmd)
 {
      //检查_cmd
      assert([NSStringFromSelector(_cmd) isEqualToString:@"originalMethod"]);
      //code
     //这里调用了原来的objc方法
     int returnValue = ((int(*)(id,SEL))__original_Method_Imp)(self, _cmd);
    return returnValue + 1;
 }

- (void) swizzleExample //call me to swizzle
 {	
  //获取objc Method
     Method m = class_getInstanceMethod([self class],
 @selector(originalMethod));
  //用 method_setImplementation 方法，将objc Method的IMP指针替换为C方法的指针。
     __original_Method_Imp = method_setImplementation(m,
 (IMP)_replacement_Method);
 }

//原来的objc方法
- (int) originalMethod
 {
        //code
        assert([NSStringFromSelector(_cmd) isEqualToString:@"originalMethod"]);
        return 1;
 }
```

为了避免与第三方库的冲突，我们需要使用 `method_setImplementation`来替代`method_exchangeImplementations`。将c方法转换为IMP，这避免了使用`Objective-C`方法的副作用，比如一个新的方法指针。使用`swizzle`最好的方式就是使用后不留下痕迹。

此外，在`ARC`中，系统会默认`IMP`的返回值为`id`类型，所以当自定义的c方法的返回值为`void`，我们需要进行强制类型转换，防止`ARC`对`void`的强引用。

```ObjectiveC
IMP anImp; //represents objective-c function
          // -UIViewController viewDidLoad;
 ((void(*)(id,SEL))anImp)(self,_cmd); //call with a cast to prevent
                                     // ARC from retaining void.
```

# +load vs +initialize

`+initialize`和`+load`的一个很大区别是，`+initialize`是通过`objc_msgSend`进行调用的，所以有以下特点：

* 如果子类没有实现`+initialize`方法，会调用父类的`+initialize`(所以父类的`+initialize`方法可能会被调用多次)
* 如果分类实现了`+initialize`，会覆盖类本身的`+initialize`调用。
## 相同点
* 在不考虑开发者主动使用的情况下，系统最多会调用一次

* 如果父类和子类都被调用，父类的调用一定在子类之前

* 都是为了应用运行提前创建合适的运行环境

* 在使用时都不要过重地依赖于这两个方法，除非真正必要

## +load

* 调用时机比较早，运行环境有不确定因素。具体说来，在iOS上通常就是App启动时进行加载，但当load调用的时候，并不能保证所有类都加载完成且可用，必要时还要自己负责做auto release处理。
* 补充上面一点，对于有依赖关系的两个库中，被依赖的类的load会优先调用。但在一个库之内，调用顺序是不确定的。
* 对于一个类而言，没有load方法实现就不会调用，不会考虑对NSObject的继承。
* 一个类的load方法不用写明[super load]，父类就会收到调用，并且在子类之前。
* Category的load也会收到调用，但顺序上在主类的load调用之后（分类不会覆盖类本身的调用）。
* 不会直接触发initialize的调用。
* 即写了才会掉用，不写不会掉用，子类的调用会触发父类的调用（父类实现了的话）分类不会覆盖自身类的实现，父类->子类->分类

## +initialize

* initialize的自然调用是在第一次主动使用当前类的时候（lazy，这一点和Java类的“clinit”的很像）。

* 在initialize方法收到调用时，运行环境基本健全。

* initialize的运行过程中是能保证线程安全的。

* 和load不同，即使子类不实现initialize方法，会把父类的实现继承过来调用一遍。注意的是在此之前，父类的方法已经被执行过一次了，同样不需要super调用。(对子类而言， 即子类不实现而父类实现会调用父类方法，子类实现就只调用子类方法，父类的先调用是对父类而言。)

* 只有当子类的initialize的方法未实现的时候才调用会触发父类的该方法调用。**基本上只要对该类发送了消息，那么对应的initialize就会调用，我们只需要观察对应类，来看其和其父类的方法实现情况来进行判断**。

* 由于initialize的这些特点，使得其应用比load要略微广泛一些。可用来做一些初始化工作，或者单例模式的一种实现方案。


  > 如果存在这样的关系 `SubClass2: SubClass: SuperClass`，那么如果只有`SuperClass`实现了initialize，那么这个initialize会被调用三次。
  >
  > 首先看SubClass2，其initialize没有实现，就会调用父类的initialize，父类的initialize同样没有实现，就会调用SuperClass的initialize。
  > 其实是SubClass，其initialize没有实现，就会调用父类的initialize。
  > 最后是SuperClass，它的initialize实现了，直接调用自身就行。
