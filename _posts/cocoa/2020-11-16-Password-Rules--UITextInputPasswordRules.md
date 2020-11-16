---
title: Password Rules / UITextInputPasswordRules
author: strayRed
date: 2020-11-16 15:36:00 +0800
categories: [ios, cocoa, uikit]
tags: [ios, cocoa, uikit]
---

# Automatic Strong Passwords

Safari 的自动填充从 iOS 8 起就可以生成密码了，但是它有一个缺点就是不能保证生成的密码符合某些服务的要求。

Apple 通过 iOS 12 和 macOS Mojave 里 Safari 中的自动式强密码功能来解决这个问题。

WebKit 工程师 Daniel Bates 在 3 月 1 日给 WHATWG 提交了[这个提案](https://github.com/whatwg/html/issues/3518)。6 月 6 日，WebKit 团队[发布了 Safari Technology Preview 58](https://webkit.org/blog/8327/safari-technology-preview-58-with-safari-12-features-is-now-available/)，使用新属性 `passwordrules` 来支持强密码生成。同时，WWDC 发布了 iOS 12 beta SDK，包括新的 `UITextInputPasswordRules` API，还有验证码自动输入和联合身份验证等其他一些密码管理功能。

# Password Rules

密码规则就像是密码生成器的配方。根据一些简单的规则，密码生成器就可以随机生成满足服务提供方需求的新密码。

密码规则由一个或多个键值对组成：`required: lower; required: upper; required: digit; allowed: ascii-printable; max-consecutive: 3;`

## Keys

- `required`: 需要的字符类型
- `allowed`: 允许使用的字符类型
- `max-consecutive`: 允许字符连续出现次数的最大值
- `minlength`: 密码最小长度
- `maxlength`: 密码最大长度

`required` 和 `allowed` 键使用下面列出的 character classes 作为值。`max-consecutive`、`minlength` 和 `maxlength` 使用非负整数作为值。

## Character Classes

`required` 和 `allowed` 键可以使用下面的字符类别作为值：

- `upper` (`A-Z`)
- `lower` (`a-z`)
- `digits` (`0-9`)
- `special` (`-~!@#$%^&\*\_+=`|(){}[:;"'<>,.? ]` 和空格)
- `ascii-printable` (U+0020 — 007f)
- `unicode` (U+0 — 10FFFF)

除了这些预置字符类别，还可以用方括号包住 ASCII 字符来指定自定义字符类别（比如 `[abc]`）。

> Apple 的 [Password Rules Validation Tool](https://developer.apple.com/password-rules/) 让你可以对不同的规则进行实验，并得到实时的结果反馈。甚至可以生成并下载上千个密码用来开发和测试！
>
> 更多有关于密码规则的语法，请查看 Apple 的文档[「Customizing Password AutoFill Rules」](https://developer.apple.com/documentation/security/password_autofill/customizing_password_autofill_rules)。

# Specifying Password Rules

在 iOS 上，给 `UITextField` 的 `passwordRules` 属性设置一个 `UITextInputPasswordRules` 对象（同时也应该将 `textContentType` 属性设置为 `.newPassword`）：

```
let newPasswordTextField = UITextField()
newPasswordTextField.textContentType = .newPassword
newPasswordTextField.passwordRules = UITextInputPasswordRules(descriptor: "required: upper; required: lower; required: digit; max-consecutive: 2; minlength: 8;")
```

在网页上，设置 `<input>` 元素（且 `type="password"`）的 `passwordrules` 属性：

```
<input type="password" passwordrules="required: upper; required: lower; required: special; max-consecutive: 3;"/>
```

> 如果没有指定，默认的密码规则是 `allowed: ascii-printable`。如果表单中有密码验证区域，它的密码规则会从上一个区域继承下来。

# Generating Password Rules in Swift

不光是只有你会觉得直接使用没有良好抽象的字符串格式令人感到不安。

下面是一种将密码规则封装成 Swift API 的方式

```Swift
enum PasswordRule {
    enum CharacterClass {
        case upper, lower, digits, special, asciiPrintable, unicode
        case custom(Set<Character>)
    }

    case required(CharacterClass)
    case allowed(CharacterClass)
    case maxConsecutive(UInt)
    case minLength(UInt)
    case maxLength(UInt)
}

extension PasswordRule: CustomStringConvertible {
    var description: String {
        switch self {
        case .required(let characterClass):
            return "required: \(characterClass)"
        case .allowed(let characterClass):
            return "allowed: \(characterClass)"
        case .maxConsecutive(let length):
            return "max-consecutive: \(length)"
        case .minLength(let length):
            return "minlength: \(length)"
        case .maxLength(let length):
            return "maxlength: \(length)"
        }
    }
}

extension PasswordRule.CharacterClass: CustomStringConvertible {
    var description: String {
        switch self {
        case .upper: return "upper"
        case .lower: return "lower"
        case .digits: return "digits"
        case .special: return "special"
        case .asciiPrintable: return "ascii-printable"
        case .unicode: return "unicode"
        case .custom(let characters):
            return "[" + String(characters) + "]"
        }
    }
}
```

有了这个，我们就可以在代码里指定一些规则，然后用它们生成有效的密码规则语法字符串：

```Swift
let rules: [PasswordRule] = [ .required(.upper),
                              .required(.lower),
                              .required(.special),
                              .minLength(20) ]

let descriptor = rules.map{ "\($0.description);" }
                      .joined(separator: " ")

// "required: upper; required: lower; required: special; max-consecutive: 3;"
```

只要你愿意，你甚至可以扩展 `UITextInputPasswordRules` 给它添加一个接收 `PasswordRule` 数组的 convenience initializer。

```Swift
extension UITextInputPasswordRules {
    convenience init(rules: [PasswordRule]) {
        let descriptor = rules.map{ $0.description }
                              .joined(separator: "; ")

        self.init(descriptor: descriptor)
    }
}
```