---
title: NSURLProtocol
author: strayRed
date: 2020-11-15 16:21:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

URLProtocol 是属于 Foundation 框架里的 [URL Loading System](https://link.jianshu.com/?t=https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html) 的一部分，它是一个抽象类,不能去实例化它,只能子类化NSURLProtocol，然后使用的时候注册子类。一个相对晦涩难解的类。

那么如果开发者自定义的一个 NSURLProtocol 并且注册到 app 中，那么在这个自定义的 NSURLProtocol 中我们可以拦截所有的NSURLRquest，进行修改，重定向，或者更新缓存策略等。

通过声明一个类继承自NSURLProtocol，可以拦截各种网络请求。（Http，UIWebView）

```Swift
class HttpsURLProtocol: URLProtocol, URLSessionDataDelegate 	
```

常用的父类方法有：

```Swift
//是否拦截该请求，并处理
class func canInit(with request: URLRequest) -> Bool
//将拦截的请求转换为另一个请求处理
class func canonicalRequest(for request: URLRequest) -> URLRequest
//开始处理
func startLoading()
//结束处理
func stopLoading()
//前4个需要实现

// 验证两个请求是否使用同样的缓存
class func requestIsCacheEquivalent(_ a: URLRequest, to b: URLRequest) -> Bool
```
# canInit

首先从类方法 canInit 开始，在这里需要通过key来判断请求是否已经被拦截过，防止一个URLRequest被多次处理。返回true表示拦截该请求，并进行下一步。

```Swift
private let forgeroundHandledKey = "ForgeroundhandledKey"
private let backgroundHandledKey = "BackgroundHandledKey"    

override class func canInit(with request: URLRequest) -> Bool {
        guard request.url?.scheme == "https" || request.url?.scheme == "http" else { return false }
        
        let urlString = String(request.url?.absoluteString.split(separator: "?")[0] ?? "")
        guard handledComponents.contains(urlString) else {  return false }
        
        if URLProtocol.property(forKey: forgeroundHandledKey, in: request) != nil || URLProtocol.property(forKey: backgroundHandledKey, in: request) != nil  {
          // 这里表明已经处理过这个request了
            return false
        }
        return true

    }
```
# canonicalRequest

然后在 canonicalRequest 方法中这个方法用来统一处理请求 request 对象，可以修改头信息，或者重定向。没有特别要求就直接返回request。

```Swift
override class func canonicalRequest(for request: URLRequest) -> URLRequest {
        return request
  }
```

# startLoading

之后就是最重要的startLoading方法，这里可以对拦截的request做处理（根据不同情况动态地做出操作），并标记传入的reuquset。

```Swift
    override func startLoading() {
    //标记设置key
        URLProtocol.setProperty(true, forKey: forgeroundHandledKey, in: self.request as! NSMutableURLRequest)
        let configuration = URLSessionConfiguration.default
        //建立一个新的URLSession进行网络请求
        session = URLSession.init(configuration: URLSessionConfiguration.default, delegate: self, delegateQueue: queue)
        let task = session.dataTask(with: self.request)
        task.resume()
    }
```

# stopLoading

```Swift
    override func stopLoading() {
        session?.invalidateAndCancel()
        session = nil
    }
```


# URLProtocolClient

URLProtocolClient是一个协议类型，用来接受拦截网络请求后处理的结果（就是我们拦截前处理网络请求的对象），在 URLProtocol 中可以使用 `clinet` 属性访问。

```Swift
func urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive response: URLResponse, completionHandler: @escaping (URLSession.ResponseDisposition) -> Void) {
//网络请求返回response
        self.client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .allowed)
        completionHandler(URLSession.ResponseDisposition.allow)
    }
    
    func urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive data: Data) {
    //网络请求返回数据
        self.client?.urlProtocol(self, didLoad: data)
    }
    func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
    //网络请求完成
        if let error = error {
            self.client?.urlProtocol(self, didFailWithError: error)
        }
        else {
            self.client?.urlProtocolDidFinishLoading(self)
        }
        
    }
```

# URLProtocolClass

要使 URLProtocol 生效，我们需要首先需要进行注册。

```Swift
//注册
URLProtocol.registerClass(HttpsURLProtocol.self)
//取消注册
URLProtocol.unregisterClass(HttpsURLProtocol.self)
```

除此之外，还需要设置 需要拦截请求的 URLSession 对应的 URLSessionConfiguration 的属性 `protocolClasses`。有两种方法设置这个属性。

1) 直接修改

```Swift
//default
let configuration = URLSessionConfiguration.default
//更该其protocolClasses 才能拦截URLSession的网络请求
configuration.protocolClasses = [HttpsURLProtocol.self]
```

2) 运行时更改 `protocolClasses` 属性的 IMP。

```Objective-C
//这是自己的protocolClasses
- (NSArray *)protocolClasses {
    // 如果还有其他的监控protocol，也可以在这里加进去
    return @[[HttpsURLProtocol class]];
}
//swizzle方法用于交换两个属性的get方法
- (void)swizzleSelector:(SEL)selector fromClass:(Class)original toClass:(Class)stub {
    Method originalMethod = class_getInstanceMethod(original, selector);
    Method stubMethod = class_getInstanceMethod(stub, selector);
    if (!originalMethod || !stubMethod) {
        [NSException raise:NSInternalInconsistencyException format:@"Couldn't load NEURLSessionConfiguration."];
    }
    method_exchangeImplementations(originalMethod, stubMethod);
}


//交换
- (void)load {
    self.isSwizzle=YES;
    Class cls = NSClassFromString(@"__NSCFURLSessionConfiguration") ?: NSClassFromString(@"NSURLSessionConfiguration");
    [self swizzleSelector:@selector(protocolClasses) fromClass:cls toClass:[self class]];
    
}

- (void)unload {
    self.isSwizzle=NO;
    Class cls = NSClassFromString(@"__NSCFURLSessionConfiguration") ?: NSClassFromString(@"NSURLSessionConfiguration");
    [self swizzleSelector:@selector(protocolClasses) fromClass:cls toClass:[self class]];
}
```

