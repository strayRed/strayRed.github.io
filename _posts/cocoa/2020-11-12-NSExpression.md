---
title: NSExpression
author: strayRed
date: 2020-11-12 13:58:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

`NSExpression`，用于创建表达式，并可以计算其结果，主要的使用方式是组成`NSComparisonPredicate`。

# Evaluating Math

关于`NSExpression`你所要知道的第一件事就是它的主要目的是减少表达。一个`NSPredicate`的组成一般都是两个表达式加上一个运算符号，我们可以将表达式用`NSExpression`来进行表达。

```Objective-C
//数学运算
NSExpression *expression = [NSExpression expressionWithFormat:@"4 + 5 - 2**3"];
计算结果
id value = [expression expressionValueWithObject:nil context:nil]; // => 1
```

# Functions

我们同样可以在`NSExpression`调用标准库的计算函数。

```Objective-C
NSArray *numbers = @[@1, @2, @3, @4, @4, @5, @9, @11];
NSExpression *expression = [NSExpression expressionForFunction:@"stddev:" arguments:@[[NSExpression expressionForConstantValue:numbers]]];
id value = [expression expressionValueWithObject:nil context:nil]; // => 3.21859...
```

`NSExpression` 函数以给定数目的子表达式作为参数。比如，在上述例子中，要得到集合的标准差，数列中的数字要被`+expressionForConstantValue:`封装。

## Statistics

- `average:`
- `sum:`
- `count:`
- `min:`
- `max:`
- `median:`
- `mode:`
- `stddev:`

## Basic Arithmetic

这些函数需要用两个`NSExpression`对象来表达数字。

- `add:to:`
- `from:subtract:`
- `multiply:by:`
- `divide:by:`
- `modulus:by:`
- `abs:`

## Advanced Arithmetic

- `sqrt:`
- `log:`
- `ln:`
- `raise:toPower:`
- `exp:`

## Bounding Functions

- `ceiling:` - *（不小于数组中的值的最小积分值）*
- `trunc:` - *（最接近但不大于数组中的值的积分值）*

## Functions Shadowing math.h Functions

`ceiling`非常容易和`ceil(3)`混淆。`ceiling`作用于数字数组，而`ceil(3)`作用于一个`double`值（且它并没对应的内置`NSExpression`函数）。`floor:`在这里的作用和`floor(3)`一样。

- `floor:`

## Random Functions

两个变量–一个带参数，一个不带参数。不带参数时，`random`返回`rand(3)`的等值，而`random:`则从`NSExpression`的数字数组中取任意元素。

- `random`
- `random:`

## Binary Arithmetic

- `bitwiseAnd:with:`
- `bitwiseOr:with:`
- `bitwiseXor:with:`
- `leftshift:by:`
- `rightshift:by:`
- `onesComplement:`

## Date Functions

- `now`

## String Functions

- `lowercase:`
- `uppercase:`

## No-op

- `noindex:`

# Custom Functions

除了这些内置的函数，你也可以在`NSExpression`中调用自定义函数。

首先，在类别中定义一个对应的函数：

```Objective-C
@interface NSNumber (Factorial)
- (NSNumber *)factorial;
@end

@implementation NSNumber (Factorial)
- (NSNumber *)factorial {
    return @(tgamma([self doubleValue] + 1));
}
@end
```

然后，这样使用函数（`+expressionWithFormat:` 中的`FUNCTION()`宏是构造`-expressionForFunction:`等等的过程的简写。）:

```Objective-C
NSExpression *expression = [NSExpression expressionWithFormat:@"FUNCTION(4.2, 'factorial')"];
id value = [expression expressionValueWithObject:nil context:nil]; // 32.578...
```

这样的优势在于， 通过直接调用`-factorial`，我们可以调用`NSPredicate`查询中的函数。

