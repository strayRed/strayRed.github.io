---
title: __attribute__
author: strayRed
date: 2020-11-03 19:48:00 +0800
categories: [ios, objective-c]
tags: [ios, objective-c]
---

`__attribute__` 是一个用于在声明时指定一些特性的编译器指令，它可以让我们进行更多的错误检查和高级优化工作。

使用这个关键字的语法是 `__attribute__` 后面跟两组括号（两个括号可以让它很容易在宏里面使用，特别是有多个属性的时候）。在括号里面是用逗号分隔的属性列表。`__attribute__` 指令可以放在函数，变量和类型声明之后。

```Objective-C
// Return the square of a number
int square(int n) __attribute__((const));

// Declare the availability of a particular API
void f(void)
  __attribute__((availability(macosx,introduced=10.4,deprecated=10.6)));

// Send printf-like message to stderr and exit
extern void die(const char *format, ...)
  __attribute__((noreturn, format(printf, 1, 2)));	
```

# GCC

## format

`format` 属性用于指定一个函数接收类似 `printf`， `scanf`， `strftime` 和 `strfmon` 风格的参数，应该按照参数对格式化字符串进行类型检查。

```C
// 这里是第二个和第三个参数使用printf的格式检查
extern int
my_printf (void *my_object, const char *my_format, ...)
  __attribute__((format(printf, 2, 3)));
```

Objective-C 程序员还可以使用 `__NSString__` 来应用跟 `NSString +stringWithFormat:` 和 `NSLog()` 一样的格式化字符串规则。

## nonnull

`nonnull` 属性表明一些函数参数应该是非空的指针。

```C
// 这里是第一个和第二个参数不为空指针
extern void *
my_memcpy (void *dest, const void *src, size_t len)
  __attribute__((nonnull (1, 2)));
```

使用 `nonnull` 把对于值的预期进行了显式的硬编码，可以帮助我们找到所有调用函数时可能潜伏的 `NULL` 指针 bug。

## noreturn

少数的几个标准库函数，例如 `abort` 和 `exit`，是不能返回的。GCC 了解这一点。`noreturn` 这个属性用于声明其他不能返回的函数。

```Objective-C
// 这个方法用于开启常驻线程，线程生命周期为整个程序的生命周期
+ (void) __attribute__((noreturn)) networkRequestThreadEntryPoint:(id)__unused object {
do {
        @autoreleasepool {
            [[NSRunLoop currentRunLoop] run];
        }
    } while (YES);
}
```

## pure / const

- pure：表示函数的返回值只依赖参数或者全局变量，也就是所谓的 `pure function`，这种函数可以通过常见的子表达式消除和循环优化技术进行优化，就像算术操作符一样。

- const：表示函数的返回值只依赖参数，也是纯函数，注意，一个参数是指针类型且同时检查指针指向的数据的函数，一定不要声明为 `const`。同样的，一个调用在内部非 `const` 函数的函数通常也不能是 `const`。`const` 函数返回 `void` 类型是没有意义的。

```Objective-C
int square(int n) __attribute__((const));
```


`pure` 和 `const` 都是为了支持高效的性能优化而营造出函数式编程范例的属性。`const` 可以被看做是更加严格的 `pure`，因为它不依赖于全局变量或者指针。

> 举个例子，因为被声明为 `const` 的函数结果除了传入参数之外不依赖于任何东西，这个函数的结果就可以被缓存起来，当之后用同样的参数调用的时候可以直接把缓存返回（就像我们知道一个数字的平方是另一个常数，所以我们只需要计算一次就可以了）。

## unused

当一个函数增加了这个属性声明的时候，意味着它可能不会被使用，GCC 不会对这个函数产生警告。

使用 `__unused` 关键字可以达到同样的效果，可以在方法实现中声明没有被使用的参数。通过了解这个上下文，编译器可以进行相应的优化。你更可能会在 delegate 的方法实现里使用 `__unused`，因为 protocols 为了支持更多可能的用例经常会提供必要的参数之外的上下文。

```Objective-C
+ (void) __attribute__((noreturn)) networkRequestThreadEntryPoint:(id)__unused object;
```

## LLVM

和其他 GCC 特性一样，Clang 支持了 `__attribute__`， 还加入了一小部分扩展特性。

要检查能否使用特定的属性，可以用 `__has_attribute` 这个指令。

## availability

Clang 引入了可用性属性，可以放在声明之后，表明这个声明在操作系统版本层次上的生命周期。考虑下面这个虚构的函数 f 的声明：

```Objective-C
void f(void) __attribute__((availability(macosx,introduced=10.4,deprecated=10.6,obsoleted=10.7)));
```

`availability` 属性指出 `f` 在 OS X Tiger 中被引入，在 OS X Snow Leopard 中被废弃，在 OS X Lion 中被淘汰

`availability` 属性是用一个逗号分隔的列表，列表的第一项是平台名称（支持`ios`和`macosx`），然后是指出声明的生命周期当中重要的里程碑时间（如果有的话）的语句，最后是额外的信息。

- `introduced`: 声明被引入的第一个版本。
- `deprecated`: 声明被废弃的第一个版本，意味着用户应当从这个 API 迁移到另外的方法。
- `obsoleted`: 声明被废弃的第一个版本，意味着它被彻底删除不能使用了。
- `unavailable`: 声明在这个平台上从来就是不可用的
- `message`: 额外的文本信息，Clang 在对于废弃和淘汰声明给出警告或者错误的时候会提供这些信息，可以用于指导用户进行 API 替换。

## overloadable

Clang 在 C 语言中提供了 C++ 函数重载支持，通过 `overloadable` 这个属性实现。例如我们要提供多个不同重载版本的 `tgsin` 函数，它会调用合适的标准库函数，分别提供对 `float`，`double` 和 `long double` 精度的值计算 `sine`值。

```C
#include <math.h>
float __attribute__((overloadable)) tgsin(float x) { return sinf(x); }
double __attribute__((overloadable)) tgsin(double x) { return sin(x); }
long double __attribute__((overloadable)) tgsin(long double x) { return sinl(x); }
```

注意 `overloadable`只能用于函数。你可以通过使用 `id` 和 `void *` 这种泛型的返回值和参数类型，在一定程度上实现方法声明的重载。

> 转载自[NSHipster](https://nshipster.cn/__attribute__/)