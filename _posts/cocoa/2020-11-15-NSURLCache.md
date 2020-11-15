---
title: NSURLCache
author: strayRed
date: 2020-11-15 13:46:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

`NSURLCache` 为您的应用的 URL 请求提供了内存中以及磁盘上的综合缓存机制。 作为基础类库 [URL 加载系统](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html#//apple_ref/doc/uid/10000165i) 的一部分，任何通过 `NSURLSession` 加载的请求都将被 `NSURLCache` 处理。

当一个请求完成下载来自服务器的回应，一个缓存的回应将在本地保存。下一次同一个请求再发起时，本地保存的回应就会马上返回，不需要连接服务器。`NSURLCache` 会 自动 且 透明地返回回应。

为了好好利用 `NSURLCache`，你需要初始化并设置一个共享的 URL 缓存。在 iOS 中这项工作需要在 `-application:didFinishLaunchingWithOptions:` 完成，而 OS X 中是在 `–applicationDidFinishLaunching:`：

```Objective-C
- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  NSURLCache *URLCache = [[NSURLCache alloc] initWithMemoryCapacity:4 * 1024 * 1024
                                                       diskCapacity:20 * 1024 * 1024
                                                           diskPath:nil];
  [NSURLCache setSharedURLCache:URLCache];
}
```

# NSURLSessionConfiguration & NSURLRequestCachePolicy

在 iOS7 引入的`NSURLSession`是由`NSURLSessionConfiguration`实例来初始化的。使用 new 或者 init 的初始化`NSURLSessionConfiguration`的方式在 iOS 13 已经被废弃。一般用系统给定的者几个类方法来创建其实例。

- **+defaultSessionConfiguration** A default session configuration object.

- **+ephemeralSessionConfiguration** A session configuration that uses no persistent storage for caches, cookies, or credentials.
- **+backgroundSessionConfigurationWithIdentifier: **Creates a session configuration object that allows HTTP and HTTPS uploads or downloads to be performed in the background.

> 对于`NSURLSession`的任务而言，默认是在子线程进行，这个线程对开发者不可见，我们也没有办法为期分配调度系统资源，只能使用 `NSURLSessionTask` 的 `priority`属性设置任务的优先级。

在创建完 NSURLSessionConfiguration 实例后，我们还可以为其设置各种属性。

- **identifier**（readonly, copy） The background session identifier of the configuration object.
- **HTTPAdditionalHeaders**（copy） A dictionary of additional headers to send with requests.
- **networkServiceType** The type of network service for all tasks within sessions based on this configuration.
- **allowsCellularAccess** A Boolean value that determines whether connections should be made over a cellular network.
- **timeoutIntervalForRequest** The timeout interval to use when waiting for additional data.
- **timeoutIntervalForResource** The maximum amount of time that a resource request should be allowed to take.
- **sharedContainerIdentifier** The identifier for the shared container into which files in background URL sessions should be downloaded.
- **waitsForConnectivity** A Boolean value that indicates whether the session should wait for connectivity to become available, or fail immediately.

对于 NSURLCache，NSURLSessionConfiguration 同样有一个 `requestCachePolicy` 属性来设置缓存策略。

关于`NSURLRequestCachePolicy`，只需要了解：

|                    常量                     |                            意义                            |
| :-----------------------------------------: | :--------------------------------------------------------: |
|          `UseProtocolCachePolicy`           |                          默认行为                          |
|       `ReloadIgnoringLocalCacheData`        |                         不使用缓存                         |
| ~~`ReloadIgnoringLocalAndRemoteCacheData`~~ |               ~~我是认真地，不使用任何缓存~~               |
|          `ReturnCacheDataElseLoad`          | 使用缓存（不管它是否过期），如果缓存中没有，那从网络加载吧 |
|          `ReturnCacheDataDontLoad`          |  离线模式：使用缓存（不管它是否过期），但是*不*从网络加载  |
|      ~~`ReloadRevalidatingCacheData`~~      |                  ~~在使用前去服务器验证~~                  |

# HTTP Cache Semantics

HTTP 请求和回应用 [headers](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html) 来交换元数据，如字符编码、MIME 类型和缓存指令等。

## Request Cache Headers

在默认情况下，`NSURLRequest` 会用当前时间决定是否返回缓存的数据。为了更精确地控制，使用以下请求头：

- `If-Modified-Since` - 这个请求头与 `Last-Modified` 回应头相对应。把这个值设为同一endpoint最后一次请求时返回的 `Last-Modified` 字段的值。
- `If-None-Match` - 这个请求头与 `Etag` 回应头相对应。使用同一endpoint最后一次请求的 `Etag` 值。

## Response Cache Headers

`NSHTTPURLResponse` 包含多个 HTTP 头，当然也包括以下指令来说明回应应当如何缓存：

- `Cache-Control` - 这个头必须由服务器端指定以开启客户端的 HTTP 缓存功能。这个头的值可能包含 `max-age`（缓存多久），是公共 `public` 还是私有 `private`，或者不缓存 `no-cache` 等信息。详情请参阅 [`Cache-Control` section of RFC 2616](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9)。

除了 `Cache-Control` 以外，服务器也可能发送一些附加的头用于根据需要有条件地请求（如上一节所提到的）：

- `Last-Modified` - 这个头的值表明所请求的资源上次修改的时间。例如，一个客户端请求最近照片的时间线，`/photos/timeline`，`Last-Modified` 的值可以是最近一张照片的拍摄时间。
- `Etag` - 这是 “entity tag” 的缩写，它是一个表示所请求资源的内容的标识符。在实践中，`Etag` 的值可以是类似于资源的 [`MD5`](https://en.wikipedia.org/wiki/MD5) 之类的东西。这对于那些动态生成的、可能没有明显的 `Last-Modified` 值的资源非常有用。

# NSURLSessionDataDelegate

NSURLSession的请求都会在其代理中响应。

- session-level events：使用 NSURLSessionDelegate。
- task-level events：使用 NSURLTaskDelegate。
- data-level events：使用 NSURLSessionDataDelegate，主要是数据处理和上传。
- download-level events：使用NSURLSessionDownloadDelegate。

此外，有以下继承关系 

NSURLSessionDataDelegate: NSURLTaskDelegate:  NSURLSessionDelegate

NSURLSessionDownloadDelegate: NSURLTaskDelegate: NSURLSessionDelegate

> 我们也可以不设置代理，使用闭包的形式接受数据 。
>
> ```Objective-C
> //NSURLSession的实例方法
> - (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request 
>                             completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;
> ```

正如上述，如果我们要获取网络请求的数据且更改NSCachedURLResponse，我们需要使用 NSURLSessionDataDelegate。

```Objective-C
- (void)URLSession:(NSURLSession *)session 
          dataTask:(NSURLSessionDataTask *)dataTask 
 willCacheResponse:(NSCachedURLResponse *)proposedResponse 
 completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler {
   //NSCachedURLResponse 为不可变的类型，所以需要对其属性使用 mutableCopy 获取 可变的深拷贝
   NSMutableDictionary *mutableUserInfo = [[cachedResponse userInfo] mutableCopy];
    NSMutableData *mutableData = [[cachedResponse data] mutableCopy];
   // 只在内存中缓存
    NSURLCacheStoragePolicy storagePolicy = NSURLCacheStorageAllowedInMemoryOnly;
   
   NSCachedURLResponse *newCachedResponse = [[NSCachedURLResponse alloc] initWithResponse
                                             [cachedResponse response] data:mutableData
                                          userInfo:mutableUserInfostoragePolicy:storagePolicy];
   completionHandler(newCachedResponse);
 }
```

上面的方法只会在 NSURLProtocol 处理请求并决定缓存后才会调用（即调用 client 的 `URLProtocol:cachedResponseIsValid:`方法）。

此外，缓存还有以下条件：

- The request is for an HTTP or HTTPS URL (or your own custom networking protocol that supports caching).
- The request was successful (with a status code in the `200–299` range).
- The provided response came from the server, rather than out of the cache.
- The session configuration’s cache policy allows caching.
- The provided `URLRequest` object's cache policy (if applicable) allows caching.
- The cache-related headers in the server’s response (if present) allow caching.
- The response size is small enough to reasonably fit within the cache. (For example, if you provide a disk cache, the response must be no larger than about 5% of the disk cache size.)