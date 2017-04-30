---
title: iOS网络请求缓存：NSURLCache详解
date: 2016-12-11 16:18:57
tags:
	- iOS
	- NSURLCache
	- 网络请求
	- 缓存
---
我读过一些开源项目的网络请求缓存的代码，基本上都是采用在本地存文件的方式进行缓存。如果你打算在你的项目中加入网络请求的缓存，可能你并不需要自己造一个轮子，了解一下`NSURLCache`就足够。

这是一个Apple已经为你准备好了的网络请求缓存类。网上对这个类的介绍并不多，并且有的文章讲得很不详细。希望这篇文章能让你对`NSURLCache`有一个比较详细的了解。

<!--more-->

# 缓存

首先，`NSURLCache`提供的是内存以及磁盘的综合缓存机制。许多文章谈到，使用`NSURLCache`之前需要在`AppDelegate`中缓存空间的设置：

``` objc
- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  NSURLCache *URLCache = [[NSURLCache alloc] initWithMemoryCapacity:4 * 1024 * 1024
                                                       diskCapacity:20 * 1024 * 1024
                                                           diskPath:nil];
  [NSURLCache setSharedURLCache:URLCache];
}

```

然而如果你不添加上面的代码，并且运行如下代码，可以看到：

``` swift
print(NSURLCache.sharedURLCache().diskCapacity)
//output:
//10000000

print(NSURLCache.sharedURLCache().memoryCapacity)
//output:
//512000

```

也就是说，其实默认就已经设置好了512kb的内存缓存空间，以及10MB的磁盘缓存空间。可能你的代码中并没有写任何与`NSURLCache`有关的东西，但其实它已经默默的开始帮你进行缓存了。

已经缓存上了，但是怎么使用缓存呢？请继续往下。

# 缓存策略

#### GET

不用多说，`NSURLCache`只会对你的`GET`请求进行缓存。

#### NSURLRequestCachePolicy

`NSURLRequest`中有个属性：

``` swift
public var cachePolicy: NSURLRequestCachePolicy { get }

```

你可以通过这个属性来设置请求的缓存策略，

``` swift
public enum NSURLRequestCachePolicy : UInt {
    
    case UseProtocolCachePolicy // 默认值
    
    case ReloadIgnoringLocalCacheData // 不使用缓存数据
    case ReloadIgnoringLocalAndRemoteCacheData // Unimplemented
    public static var ReloadIgnoringCacheData: NSURLRequestCachePolicy { get }
    
    case ReturnCacheDataElseLoad // 无论缓存是否过期都是用缓存，没有缓存就进行网络请求
    case ReturnCacheDataDontLoad // 无论缓存是否过期都是用缓存，没有缓存也不会进行网络请求
    
    case ReloadRevalidatingCacheData // Unimplemented
}
```

其实其他几个值都比较好理解，唯独默认值`UseProtocolCachePolicy`让我不太懂。

字面上的意思是**按照协议的缓存策略进行缓存**，那么这是什么协议呢？**http协议**。

[详细：RFC 2616, Section 13](https://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html#sec13)

服务器返回的响应头中会有这样的字段：`Cache-Control: max-age` or `Cache-Control: s- maxage`，通过`Cache-Control`来指定缓存策略，`max-age`来表示过期时间。根据这些字段缓存机制再采用如下策略：

* 如果本地没有缓存数据，则进行网络请求。
* 如果本地有缓存，并且缓存没有失效，则使用缓存。
* 如果缓存已经失效，则询问服务器数据是否改变，如果没改变，依然使用缓存，如果改变了则请求新数据。
* 如果没有指定是否失效，那么系统将自己判断缓存是否失效。（通常认为是6-24小时的有效时间）

其实我以前对`Cache-Control`之类的也并不太了解 T_T，自己默默的print了一下响应头，你可以看到：

``` swift
print((response as? NSHTTPURLResponse)?.allHeaderFields)

//响应头中：Cache-Control: no-cache
```

这也就是为什么，虽然`NSURLCache`一直在默默的缓存，但是我并没有感受到，当然或许你那里不一样。~~这个`no-cache`就表示不缓存。~~**（勘误）** *修正：no-cache表示不使用缓存，但是会缓存，no-store表示是不进行缓存。*

这里要额外提一句，看到网上有同学说自己出现了某个请求数据一直使用缓存，没有被更新。这种情况可能就是服务器返回的`Cache-Control`有误。

打开沙盒路径下的Library/Caches 中，你可以看到缓存文件：

![沙盒中的缓存文件.png](https://user-gold-cdn.xitu.io/2016/12/11/ca10144bb74a4b98f8a8ed435640e5f3)

这可以说明存在磁盘上的数据是存在数据库里的，性能不用担心。打开数据库文件就可以看到请求的数据。

![缓存数据.png](https://user-gold-cdn.xitu.io/2016/12/11/a8e70d2e1d3255127addf221c9643ff7)

在`cfurl_cache_response`表中可以看到有一个字段是`request_key`，通过里面的值可以推断每一个`response`是通过请求的`url+参数`来作为`key`储存的。

当然，经过我的多次试验，在`Cache-Control: no-cache`的情况下，`NSURLCache`也会进行缓存，但是并不使用缓存数据。

**总结一下：**默认情况下`NSURLCache`的缓存策略是根据http协议来的，服务器通过`Cache-Control: max-age`字段来告诉`NSURLCache`是否需要缓存数据。

# 缓存封装

如果你不打算采用http协议的缓存策略，依然可以使用`NSURLCache`进行缓存。

``` swift
public func cachedResponseForRequest(request: NSURLRequest) -> NSCachedURLResponse?
```

你可以通过这个方法，传入请求，来获取缓存。`NSCachedURLResponse`保存了上次请求的数据以及响应头。


``` swift
public func storeCachedResponse(cachedResponse: NSCachedURLResponse, forRequest request: NSURLRequest)
```
`NSURLSessionDelegate`协议中有如下方法，可以对即将缓存的数据进行修改，添加userInfo,在代理方法中必须调用completionHandler，传入将要缓存的数据，如果传nil则表示不缓存。

``` swift
optional public func URLSession(session: NSURLSession, 
                               dataTask: NSURLSessionDataTask, 
	 willCacheResponse proposedResponse: NSCachedURLResponse, 
                      completionHandler: (NSCachedURLResponse?) -> Void)
```
在`Alamofire`中可以这样写：

``` swift
Alamofire.Manager
.sharedInstance
.delegate
.dataTaskWillCacheResponse = { (session, task, cachedResponse) -> NSCachedURLResponse? in
	var userInfo = [NSObject : AnyObject]()
    // 设置userInfo
	return NSCachedURLResponse(response: cachedResponse.response,
                               data: cachedResponse.data,
                               userInfo: userInfo,
                               storagePolicy: cachedResponse.storagePolicy)
}
```

#参考


我也只是简单的对`NSURLCache`进行了介绍，需要深入了解的话大家还是需要拜读一下文章，希望能给大家一些帮助：

* [NSURLCache](http://nshipster.com/nsurlcache/)
* [Caching and NSURLConnection](https://blackpixel.com/writing/2012/05/caching-and-nsurlconnection.html)
* [RFC 2616, Section 13](https://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html#sec13)