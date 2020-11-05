---
title: Apple Push Notification Device Tokens
author: strayRed
date: 2020-11-05 13:12:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

# A Push Notifications Primer

`Push Notifications` 允许 app 与其他用户之间进行远程通讯（即使他们处于未活跃的状态）与短信或电子邮件不同，短信或电子邮件允许发送者直接使用唯一标识符（分别是电话号码和电子邮件地址）与收件人进行通信。而在`iOS`中，用户的本地设备和远程服务器的通讯都是由 `Apple Push Notification service (apns)`来进行维护的。

工作方式如下：

- app启动后，会调用 [`registerForRemoteNotifications()`](https://developer.apple.com/documentation/uikit/uiapplication/1623078-registerforremotenotifications)方法，会提示用户是否允许app发送通知。
- 如果用户允许，那么app delegate 就会调用 [`application(_:didRegisterForRemoteNotificationsWithDeviceToken:)`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1622958-application)方法来获取到`deviceToken`。

`deviceToken`是一个不透明的`Data`类型，类似于电话号码和邮件地址这样的唯一标示，它让 Notification 能通过`APNs`定位到当前的app。

# The Enduring Challenges of Sending Device Tokens Back to the Server

当获取到`deviceToken`后，整个通讯过程还没结束，我们还需要从客户端请求通知的数据，这时候我们需要在`body`中带上`deviceToken`字段。比如我们可以以`application/x-www-form-urlencoded`格式设置`https body`，如`username=jappleseed&deviceToken=____`。

根据服务端的要求，我们可能会使用`Base64`编码来将`deviceToken`转换为字符串。

> `Base64`是网络上最常见的用于传输8Bit字节码的编码方式之一，Base64就是一种基于64个可打印字符来表示二进制数据的方法。我们可以使用  [`NSData -base64EncodedStringWithOptions:`](https://developer.apple.com/documentation/foundation/nsdata/1413546-base64encodedstringwithoptions?language=objc)方法将`Data`以一个字节（8bit）为单位转换为`Base64` 编码后的`String` 。

# NSData, in its Own Words

`deviceToken`是一个不透明的`NSData`实例，很多开发者会想知道它到底是什么值，他们可能会使用`NSLog`表达式将`deviceToken`输出。

```Objective-C
NSLog(@"%@", deviceToken);
// Prints "<965b251c 6cb1926d e3cb366f dfb16ddd e6b9086a 8a3cac9e 5f857679 376eab7C>"
```

我们可以看到他是一个16进制的值，但是有些对数据和编码不了解的开发者会认为`deviceToken`就是一个单纯的字符串。他们会用以下方式将`deviceToken`转换为`NSString`

```Objective-C
// ⚠️ Warning: Don't do this
NSString *token = [[[[deviceToken description]
                    stringByReplacingOccurrencesOfString:@" " withString:@""]
                    stringByReplacingOccurrencesOfString:@"<" withString:@""]
                    stringByReplacingOccurrencesOfString:@">" withString:@""];
```

目前还不清楚为什么通知服务的提供商会使用16进制的数据来作为`token`。不过近10年来相当大比例的应用程序都是这样做的。

> 实际上，将二进制数据转化为字符串有很多种方法，一般来讲，比较常用的是`Base64`（也就是64个可打印的字符串），或者[Ascii85 ](https://en.wikipedia.org/wiki/Ascii85)，再或是 `Base16 Encoding`（*这个Base16指的是将二进制数据以8bit为单位[也就是1字节]分别转换为16进制，与Base64意义不同*）。如果我们需要``Base16`编码的`deviceToken`，那么可以采用上述的做法。

# Relitigating the Past with Swift 3

在`Swift3`中，`Data`去掉了`NS`前缀，标准库中的`NSData`也被替换为`Data`，这时候，如果输出一个`Data`类型的`deviceToken`会是这样：

```Swift
// Swift 2: deviceToken is NSData
deviceToken.description // "<965b251c 6cb1926d e3cb366f dfb16ddd e6b9086a 8a3cac9e 5f857679 376eab7C>"

// Swift 3: deviceToken is Data
deviceToken.description // "32 bytes"
```

但是在`iOS13`之前，我们同样可以将`Data`转换为`NSData`来输出16进制的值。

```Swift
// ⚠️ Warning: Don't do this
let tokenData = deviceToken as NSData
let token = "\(tokenData)".replacingOccurrences(of: " ", with: "")
                          .replacingOccurrences(of: "<", with: "")
                          .replacingOccurrences(of: ">", with: "")
```

# Overturned in iOS 13

`iOS13`改变了Foundation对象的描述格式，包括`NSData`。

```Swift
// iOS 12
(deviceToken as NSData).description // "<965b251c 6cb1926d e3cb366f dfb16ddd e6b9086a 8a3cac9e 5f857679 376eab7C>"

// iOS 13
(deviceToken as NSData).description // "{length = 32, bytes = 0x965b251c 6cb1926d e3cb366f dfb16ddd ... 5f857679 376eab7c }"
```

在以前可以将`NSData`转换为字符串来泄露它的内容， 而现在只会显示其长度以及部分内容。在`iOS13`之后，如果需要将`deviceToken`转换为16进制的编码，可以像这样做：

```Swift
let deviceTokenString = deviceToken.map { String(format: "%02x", $0) }.joined()
```

- `map`方法对`Data`中的每一个字节进行操作（Swift中的Data为一个字节序列），每一个字节传回闭包。
- 使用 [`String(format:)`](https://developer.apple.com/documentation/swift/string/3126742-init)方法，将每一个字节（UInt8）转换为进制，`%02x`表示多余位补0，2位16进制数（8位二进制只需要2位十六进制表示）。
- 最后使用`joined`将每个字符串结合成完成的字符串。

>当我们将每个一个`UInt8`转换为16进制字符串的时候，我们使用了`String(_:radix:)`，并指定了`"%02x"`格式。
>
>在 Stack Overflow，相关问题置顶的答案表示[更推荐用`"%02.2hhx"`代替`"%02x"`](https://stackoverflow.com/questions/9372815/how-can-i-convert-my-device-token-nsdata-into-an-nsstring/24979958#24979958)
>
>```Swift
>// Overflow UInt.max (255)
>String(format: "%02.2hhx", 256) // "00"
>String(format: "%02x", 256) // "100"
>
>// Underflow UInt.min (0)
>String(format: "%02.2hhx", -1) // "ff"
>String(format: "%02x", -1) // "ffffffff"
>```
>
>`"%02.2hhx"`能保证在`UInt8`的有效值之外，仍可以得到2位数的16进制值。
>
>当然在`UInt8`的有效值以内，`"%02.2hhx"`和`"%02x"`是相同的。
>
>```Swift
>(UInt.min...UInt.max).map {
>    String(format: "%02.2hhx", $0) == String(format: "%02x", $0)
>}.contains(false) // false
>```
>
>

>转载自[NSHipster](https://nshipster.com/apns-device-tokens/)