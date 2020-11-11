---
title: NSPredicate
author: strayRed
date: 2020-11-11 16:19:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

`NSPredicate`是一个Foundation类，它指定数据被获取或者过滤的方式。它的查询语言就像SQL的`WHERE`和正则表达式的交叉一样，提供了具有表现力的，自然语言界面来定义一个集合被搜寻的逻辑条件。

```Objective-C
@interface Person : NSObject
@property NSString *firstName;
@property NSString *lastName;
@property NSNumber *age;
@end

@implementation Person

- (NSString *)description {
    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
}

@end

#pragma mark -

NSArray *firstNames = @[ @"Alice", @"Bob", @"Charlie", @"Quentin" ];
NSArray *lastNames = @[ @"Smith", @"Jones", @"Smith", @"Alberts" ];
NSArray *ages = @[ @24, @27, @33, @31 ];

NSMutableArray *people = [NSMutableArray array];
[firstNames enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
    Person *person = [[Person alloc] init];
    person.firstName = firstNames[idx];
    person.lastName = lastNames[idx];
    person.age = ages[idx];
    [people addObject:person];
}];

NSPredicate *bobPredicate = [NSPredicate predicateWithFormat:@"firstName = 'Bob'"];
NSPredicate *smithPredicate = [NSPredicate predicateWithFormat:@"lastName = %@", @"Smith"];
NSPredicate *thirtiesPredicate = [NSPredicate predicateWithFormat:@"age >= 30"];

// ["Bob Jones"]
NSLog(@"Bobs: %@", [people filteredArrayUsingPredicate:bobPredicate]);

// ["Alice Smith", "Charlie Smith"]
NSLog(@"Smiths: %@", [people filteredArrayUsingPredicate:smithPredicate]);

// ["Charlie Smith", "Quentin Alberts"]
NSLog(@"30's: %@", [people filteredArrayUsingPredicate:thirtiesPredicate]);
```

# Using NSPredicate with Collections

Foundation提供使用谓词（predicate）来过滤`NSArray`／`NSMutableArray`&`NSSet`／`NSMutableSet`的方法。

不可变的集合，`NSArray`&`NSSet`，有可以通过评估接收到的predicate来返回一个不可变集合的方法`filteredArrayUsingPredicate:`和`filteredSetUsingPredicate:`。

可变集合，`NSMutableArray`&`NSMutableSet`，可以使用方法`filterUsingPredicate:`，它可以通过运行接收到的谓词来移除评估结果为`FALSE`的对象。

`NSDictionary`可以用谓词来过滤它的键和值（两者都为`NSArray`对象）。

# Using NSPredicate with Core Data

`NSFetchRequest`有一个`predicate`属性，它可以指定管理对象应该被获取的逻辑条件。谓词的使用规则在这里同样适用，唯一的区别在于，在管理对象环境中，谓词由持久化存储调度器（persistent store coordinator）评估，而不像集合那样在内存中被过滤。

# Predicate Syntax

## Substitutions

- `%@`是对值为字符串，数字或者日期的对象的替换值。
- `%K`是key path的替换值。

```Objective-C
NSPredicate *ageIs33Predicate = [NSPredicate predicateWithFormat:@"%K = %@", @"age", @33];

// ["Charlie Smith"]
NSLog(@"Age 33: %@", [people filteredArrayUsingPredicate:ageIs33Predicate]);
```

- `$VARIABLE_NAME`是可以被`NSPredicate -predicateWithSubstitutionVariables:`替换的值。

```Objective-C
//这里的letter是一个变量
NSPredicate *namesBeginningWithLetterPredicate = [NSPredicate predicateWithFormat:@"(firstName BEGINSWITH[cd] $letter) OR (lastName BEGINSWITH[cd] $letter)"];

//使用 predicateWithSubstitutionVariables 方法将 letter 替换为 A
NSLog(@"'A' Names: %@", [people filteredArrayUsingPredicate:[namesBeginningWithLetterPredicate predicateWithSubstitutionVariables:@{@"letter": @"A"}]]);
// ["Alice Smith", "Quentin Alberts"]
```

## Basic Comparisons

- `=`, `==`：左边的表达式（Expression）和右边的表达式相等。
- `>=`, `=>`：左边的表达式大于或者等于右边的表达式。
- `<=`, `=<`：左边的表达式小于等于右边的表达式。
- `>`：左边的表达式大于右边的表达式。
- `<`：左边的表达式小于右边的表达式。
- `!=`, `<>`：左边的表达式不等于右边的表达式。
- `BETWEEN`：左边的表达式等于右边的表达式的值或者介于它们之间。右边是一个有两个指定上限和下限的数值的数列（指定顺序的数列）。比如，`1 BETWEEN { 0 , 33 }`，或者`$INPUT BETWEEN { $LOWER, $UPPER }`。

## Basic Compound Predicates

- `AND`, `&&`：逻辑`与`.
- `OR`, `||`：逻辑`或`.
- `NOT`, `!`：逻辑`非`.

## String Comparisons

字符串比较在默认的情况下是区分大小写和音调的。你可以在方括号中用关键字符c和d来修改操作符以相应的指定不区分大小写和变音符号，比如  `firstname BEGINSWITH[cd] $FIRST_NAME`。

> c 为 case，大小写，d为 diacritic，变音符

- `BEGINSWITH`：左边的表达式以右边的表达式作为开始。
- `CONTAINS`：左边的表达式包含右边的表达式。
- `ENDSWITH`：左边的表达式以右边的表达式作为结束。
- `LIKE`：左边的表达式等于右边的表达式：`?`和`*`可作为通配符，其中`?`匹配1个字符，`*`匹配0个或者多个字符。
- `MATCHES`：左边的表达式根据ICU v3（更多内容请查看[ICU User Guide for Regular Expressions](http://userguide.icu-project.org/strings/regexp)）的regex风格比较，等于右边的表达式。

## Aggregate Operations

### Relational Operations

- `ANY`，`SOME`：指定下列表达式中的任意元素。比如，`ANY children.age < 18`。
- `ALL`：指定下列表达式中的所有元素。比如，`ALL children.age < 18`。
- `NONE`：指定下列表达式中没有的元素。比如，`NONE children.age < 18`。它在逻辑上等于`NOT (ANY ...)`。
- `IN`：等于SQL的`IN`操作，左边的表达必须出现在右边指定的集合中。比如，`name IN { 'Ben', 'Melissa', 'Nick' }`。

### Array Operations

- `array[index]`：指定数组中特定索引处的元素。
- `array[FIRST]`：指定数组中的第一个元素。
- `array[LAST]`：指定数组中的最后一个元素。
- `array[SIZE]`：指定数组的大小。

## Boolean Value Predicates

- `TRUEPREDICATE`：结果始终为真的谓词。
- `FALSEPREDICATE`：结果始终为假的谓词。

# NSCompoundPredicate

`NSCompoundPredicate`为`NSPredicate`的子类，用于组合谓词。

下列谓词是相等的：

```Objective-C
[NSCompoundPredicate andPredicateWithSubpredicates:@[[NSPredicate predicateWithFormat:@"age > 25"], [NSPredicate predicateWithFormat:@"firstName = %@", @"Quentin"]]];

[NSPredicate predicateWithFormat:@"(age > 25) AND (firstName = %@)", @"Quentin"];
```

虽然语法字符串文字更加容易输入，但是在有的时候，你需要结合现有的谓词。在那些情况下，你可以使用`NSCompoundPredicate -andPredicateWithSubpredicates:`&`-orPredicateWithSubpredicates:`。

# NSComparisonPredicate

`NSComparisonPredicate`为`NSPredicate`的另一个子类，`NSComparisonPredicate`从子部件构建了一个`NSPredicate`，在这种情况下，左侧和右侧都是`NSExpression`。即：NSComparisonPredicate = （NSExpression 运算符号 NSExpression ）

```Objective-C
+ (NSComparisonPredicate *)predicateWithLeftExpression:(NSExpression *)lhs 
                                       rightExpression:(NSExpression *)rhs 
                                              modifier:(NSComparisonPredicateModifier)modifier 
                                                  type:(NSPredicateOperatorType)type 
                                               options:(NSComparisonPredicateOptions)options;
```

- `modifier`：`NSComparisonPredicateModifier`为一个枚举类型，应用的修改符，Any（`NSAnyPredicateModifier`）或者ALL（`NSAnyPredicateModifier`）或者直接比较（`NSDirectPredicateModifier`）

- options：`NSComparisonPredicateOptions`为一个枚举类型。
  > - `NSCaseInsensitivePredicateOption`：A case-insensitive predicate（大小写不敏感）。你通过在谓词格式字符串中加入后面带有[c]的字符串操作（比如，”NeXT” like[c] “next”）来表达这一选项。
  >- `NSDiacriticInsensitivePredicateOption`：A diacritic-insensitive predicate（变音不敏感）。你通过在谓词格式字符串中加入后面带有[d]的字符串操作（比如，”naïve” like[d] “naive”）来表达这一选项。
  > - `NSNormalizedPredicateOption`：Indicates that the strings to be compared have been preprocessed（比较的字符串已经被预处理）。旨在用作性能优化的选项。你可以通过在谓词格式字符串中加入后面带有[n]的字符串（比如，”WXYZlan” matches[n] “.lan”）来表达这一选项。
  >- `NSLocaleSensitivePredicateOption`：Indicates that strings to be compared using `<`, `<=`, `=`, `=>`, `>` should be handled in a locale-aware fashion。表明要使用`<`，`<=`，`=`，`=>`，`>` 作为比较的字符串应该使用区域识别的方式处理。你可以通过在`<`，`<=`，`=`，`=>`，`>`其中之一的操作符后加入[l]（比如，”straße” >[l] “strasse”）以便在谓词格式字符串表达这一选项。
  
- type：`NSPredicateOperatorType`也是一个枚举类型，基本就是使用枚举列举出谓词字符串中使用的 operations。

  ````Objective-C
  enum {
     NSLessThanPredicateOperatorType = 0,
     NSLessThanOrEqualToPredicateOperatorType,
     NSGreaterThanPredicateOperatorType,
     NSGreaterThanOrEqualToPredicateOperatorType,
     NSEqualToPredicateOperatorType,
     NSNotEqualToPredicateOperatorType,
     NSMatchesPredicateOperatorType,
     NSLikePredicateOperatorType,
     NSBeginsWithPredicateOperatorType,
     NSEndsWithPredicateOperatorType,
     NSInPredicateOperatorType,
     NSCustomSelectorPredicateOperatorType,
     NSContainsPredicateOperatorType,
     NSBetweenPredicateOperatorType
  };
  typedef NSUInteger NSPredicateOperatorType;
  ````

  

# Block Predicates

最后，如果你实在不愿意学习`NSPredicate`的格式语法，你也可以学学`NSPredicate +predicateWithBlock:`。

```Objective-C
NSPredicate *shortNamePredicate = [NSPredicate predicateWithBlock:^BOOL(id evaluatedObject, NSDictionary *bindings) {
            return [[evaluatedObject firstName] length] <= 5;
        }];

// ["Alice Smith", "Bob Jones"]
NSLog(@"Short Names: %@", [people filteredArrayUsingPredicate:shortNamePredicate]);
```

因为block可以封装任意的计算，所以有一个查询类是无法以`NSPredicate`格式字符串形式来表达的（比如对运行时被动态计算的值的评估）。而且当同一件事情可以用`NSExpression`结合自定义选择器来完成时，block为完成工作提供了一个方便的接口。

> **由`predicateWithBlock:`生成的`NSPredicate`不能用于由`SQLite`存储库支持的Core Data数据的提取要求。**