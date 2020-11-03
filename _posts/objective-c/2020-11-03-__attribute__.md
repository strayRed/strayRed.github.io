---
title: __attribute__
author: strayRed
date: 2020-11-03 19:48:00 +0800
categories: [objective-c]
tags: [objective-c]
---

`__attribute__` 是一个用于在声明时指定一些特性的编译器指令，它可以让我们进行更多的错误检查和高级优化工作。

使用这个关键字的语法是 `__attribute__` 后面跟两组括号（两个括号可以让它很容易在宏里面使用，特别是有多个属性的时候）。在括号里面是用逗号分隔的属性列表。`__attribute__` 指令可以放在函数，变量和类型声明之后。

```ObjectiveC
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

```c
// 这里是第二个和第三个参数使用printf的格式检查
extern int
my_printf (void *my_object, const char *my_format, ...)
  __attribute__((format(printf, 2, 3)));
```

Objective-C 程序员还可以使用 `__NSString__` 来应用跟 `NSString +stringWithFormat:` 和 `NSLog()` 一样的格式化字符串规则。

## nonnull

`nonnull` 属性表明一些函数参数应该是非空的指针。

```c
// 这里是第一个和第二个参数不为空指针
extern void *
my_memcpy (void *dest, const void *src, size_t len)
  __attribute__((nonnull (1, 2)));
```

使用 `nonnull` 把对于值的预期进行了显式的硬编码，可以帮助我们找到所有调用函数时可能潜伏的 `NULL` 指针 bug。

