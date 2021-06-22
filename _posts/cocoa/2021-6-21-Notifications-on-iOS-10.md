---
Notification settingstitle: Notifications on iOS 10
author: strayRed
date: 2021-6-21 19:22:00 +0800
categories: [ios, cocoa, notification]
tags: [ios, cocoa, notification]
---

# Registration

## User Authorization

- Banners 

- Sound alerts 

- Badging 

Needed for local and remote notifications

```Swift
UNUserNotificationCenter.current().requestAuthorization([.alert, .sound, .badge])
 { (granted, error) in // ... }
```

## Notification settings

用户可以自定义通知的显示形式。

Configurable in Settings per app

Access to user-defined settings

```Swift
UNUserNotificationCenter.current().getNotificationSettings { (settings) in // ... }
```

# Token Registration

Remote Notifications 

Existing API

```Swift
UIApplication.shared().registerForRemoteNotifications()
```

Need network connection to talk to APNs 

Token must be included in remote payload

推送都是通过苹果的推送服务器(apns)进行的,token就相当于你设备的唯一识别符,就理解成身分证就行了. 推送的流程就是,在你打开应用的时候向苹果的服务器注册远程推送服务,注册成功后会返回一个token给你,然后你把这个token传给你自己的服务器,在你自己的服务器端将这个token和你的帐号对应起来,这样服务器端就可以做到定点推送,就是想推给谁就推给谁,推送的过程应该很简单,就是在服务器端把要对送的内容和要推送的token发送给苹果的apns服务器就行了.

开发环境获取的deviceToken和发布环境获取的deviceToken是不一样的。
在一台设备中，deviceToken是系统级别的，不同App获得的deviceToken是相同的。
deviceToken会过期。
单个App的更新deviceToken不会发生改变。
当进行备份恢复、或恢复出厂设置之类的操作时，deviceToken会发生改变，建议App在每次启动时都获取deviceToken。
用户抹除iPhone的数据时，为了保护隐私，deviceToken会改变。
升级系统deviceToken有可能变化，猜测是升级大的系统版本后deviceToken会变化。
在删除手机上的App之后，再次下载安装，deviceToken在部分系统上会改变。

在`iOS13`之后，如果需要将`deviceToken`转换为16进制的编码，可以像这样做：

```Swift
let deviceTokenString = deviceToken.map { String(format: "%02x", $0) }.joined()
```

# Content

## Local Notification

```Swift
let content = UNMutableNotificationContent()
content.title = "Introduction to Notifications"
content.subtitle = "Session 707"
content.body = "Woah! These new notifications look amazing! Don’t you agree?"
content.badge = 1
```

## Remote Notification

payload

```Json
{
 "aps" : {
 "alert" : {
 "title" : "Introduction to Notifications",
 "subtitle" : "Session 707"
,
 "body" : "Woah! These new notifications look amazing! Don’t you agree?"
 },
 "badge" : 1
 },
}
```

# Triggers

## Push
## Time Interval 

In 2 minutes from now

```Swift
UNTimeIntervalNotificationTrigger(timeInterval: 120, repeats: false)	
```

Repeat every hour starting now

```Swift
UNTimeIntervalNotificationTrigger(timeInterval: 3600, repeats: true)
```



## Calendar

8:00am tomorrow morning

Repeat every Monday at 6:00pm

```Swift
let dateComponents = DateComponents()
// Configure dateComponents
UNCalendarNotificationTrigger(dateMatching: dateComponents, repeats: false) 
```

## Location

When leaving home

When arriving in proximity of grocery store

```Swift
let region = CLRegion()
// Configure region
UNLocationNotificationTrigger(region: region, repeats: false);
```

# Schedule

## Local Notifications

```Swift
let requestIdentifier = "sampleRequest"
let request = UNNotificationRequest(identifier: requestIdentifier,
 content: content,
 trigger: trigger)

UNUserNotificationCenter.current().add(request) { (error) in // ... }	
```



## Remote Notifications

# Notification Handling

## Application in foreground

```Swift
protocol UNUserNotificationCenterDelegate : NSObjectProtocol
func userNotificationCenter(_ center: UNUserNotificationCenter,
 willPresent notification: UNNotification,
 withCompletionHandler completionHandler:
 (UNNotificationPresentationOptions) -> Void)	{
  //在这里决定当应用处在前台时，显示哪一种通知内容。
 handlerBlock([.alert, .sound]) 
}
```

# Notification Management

Access 

- Pending Notifications 
- Delivered Notifications 

Remove Notifications 

Update and promote Notifications

## Request Identifier

通知管理是通过id实现的。

本地通知id由开发者自己设置，而远程通知的id则是通过http头的`apns-collapse-id`来获取。

## Example

Remove Pending Notification

```Swift
// Pending Notification Removal
let gameStartIdentifier = "game1.start.identifier"
let gameStartRequest = UNNotificationRequest(identifier: gameStartIdentifier,
 content: content,
 trigger: startTrigger)
UNUserNotificationCenter.current().add(gameStartRequest) { (error) in // ... }
// Game was cancelled
UNUserNotificationCenter.current()
 .removePendingNotificationRequests(withIdentifiers: [gameStartIdentifier])
```

Update Pending Notification

```Swift
let gameStartIdentifier = "game1.start.identifier"
let gameStartRequest = UNNotificationRequest(identifier: gameStartIdentifier,
 content: content,
 trigger: startTrigger)
UNUserNotificationCenter.current().add(gameStartRequest) { (error) in // ... }
// Game start time was updated
let updatedGameStartRequest = UNNotificationRequest(identifier: gameStartIdentifier,
 content: content,
 trigger: newStartTrigger)
UNUserNotificationCenter.current().add(updatedGameStartRequest) { (error) in // ... }
```

Remove Delivered Notification

```Swift
let gameScoreIdentifier = "game1.score.identifier"
let gameScoreRequest = UNNotificationRequest(identifier: gameScoreIdentifier,
 content: scoreContent,
 trigger: trigger)
UNUserNotificationCenter.current().add(gameScoreRequest) { (error) in // ... }
// Wrong game score was published
UNUserNotificationCenter.current()
 .removeDeliveredNotifications(withIdentifiers: [gameScoreIdentifier])
```

Update Delivered Notification

```Swift
let gameScoreIdentifier = "game1.score.identifier"
let gameScoreRequest = UNNotificationRequest(identifier: gameScoreIdentifier,
 content: scoreContent,
 trigger: trigger)
UNUserNotificationCenter.current().add(gameScoreRequest) { (error) in // ... }
// Score game was updated
let updateGameScoreRequest = UNNotificationRequest(identifier: gameScoreIdentifier,
 content: newScoreContent,
 trigger: newTrigger)
UNUserNotificationCenter.current().add(updateGameScoreRequest) { (error) in // ... }
```

# Notification Actions

## Default Action

## Actionable Notifications

### Registration

```Swift
//options默认为后台action
let action = UNNotificationAction(identifier:"reply",title:"Reply",options:[])
let category = UNNotificationCategory(identifier: "message", actions: [action],
minimalActions: [action], intentIdentifiers: [], options: [])
//注册categories
UNUserNotificationCenter.current().setNotificationCategories([category])
```

### Presentation

Remote Notifications

```Json
{
aps : {
alert : “Welcome to WWDC !”,
  //通过id设置category
category: "message"
}
}
```

Local Notifications

```Swift
content.categoryIdentifier = "message"	
```

## Dismiss Action(iOS 10)

```Swift
customDismissAction: UNNotificationCategoryOptions

let category = UNNotificationCategory(identifier: "message", actions: [action],
minimalActions: [action], intentIdentifiers: [], options: [.customDismissAction])
```

当删除`category`为`message`的通知时，相应的代理方法就会收到回调。

## Response handling

```Swift
protocol UNUserNotificationCenterDelegate : NSObjectProtocol
func userNotificationCenter(_ center: UNUserNotificationCenter,
didReceive response: UNNotificationResponse,
withCompletionHandler completionHandler: () -> Void) 
```

```C

             actionIdentifier
Response --  userText                        Trigger
             Notification ------- Request--- 
 																						 Content
```

## Service Extension(iOS 10)

它的主要功能是在远程通知被展示之前，为其增加或者替换内容。

End-to-end encryption 

Add Attachments

```Swift
import UserNotifications
class NotificationService: UNNotificationServiceExtension {
 override func didReceive(request: UNNotificationRequest, withContentHandler
contentHandler:(UNNotificationContent) -> Void) {
 // Modify the notification content
 }

 override func serviceExtensionTimeWillExpire() {
 // Called before the extension will be terminated by the system
 }
}
```

```Json
{
aps : {
alert : “New Message Available”,
  // 表明使用NotificationService
mutable-content : 1
},
encrypted-content : “#myencryptedcontent”
}
```

```Swift
// Decrypt Remote Notification Payload in Service Extension and Update Notification Content
override func didReceive(request: UNNotificationRequest, withContentHandler contentHandler:
(UNNotificationContent) -> Void) {
 // Decrypt the payload
 let decryptedBody = decrypt(request.content.userInfo[“encrypted-content”])
 let newContent = UNMutableNotificationContent()
 // Modify the notification content
 newContent.body = decryptedBody
 // Call content handler with updated content
 contentHandler(newContent)
}
```