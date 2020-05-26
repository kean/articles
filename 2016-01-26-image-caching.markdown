---
layout: post
title: "Image Caching"
subtitle: "In-depth look at HTTP cache and Foundation's URL Loading System to cache images on disk"
description: Using URLCache (HTTP cache) and Foundation's URL Loading System to cache and validate images. Storing images in memory using NSCache.
date: 2016-01-26 15:30:05 +0300
category: programming
tags: swift, ios, http, nsurlsession
permalink: /post/image-caching
redirect_from: /blog/image-caching
uuid: d9cd6b09-75ec-4873-a9ef-c7457cc40b34
---

Caching is a great way to improve application performance and end-user experience. Some popular iOS libraries try to reinvent caching, especially when it comes to storing images. They frequently overlook [HTTP cache](https://tools.ietf.org/html/rfc7234) in [Foundation's URL Loading System](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html).

Why HTTP cache? It's an industry standard that fits the needs of most users. HTTP provides all kinds of tools for caching and chances are that your server already supports it.

{% include ad-hor.html %}

In addition to on-disk cache, it's imperative to have a separate in-memory cache for fast access to decompressed images that are ready for display.

*This guide focuses on images, but it most certainly applies to the other areas too. Its purpose is to answer some of the common questions about caching, and to provide references to more comprehensive sources.*

> ## TL;DR
- Each resource can define its caching policy via HTTP cache headers
- HTTP cache supports expiration and validation of cached responses
- Aggressive caching is a most viable strategy for static images, validation is useful for profile pictures, etc
- Foundation's URL Loading System supports HTTP caching
- There are multiple ways to adjust URL Loading System's cache management (NSURLRequestCachePolicy etc)
- NSCache can be used for in-memory cache, it requires proper configuration (total cost limit, cost per object)

## HTTP Caching

There are several aspects of HTTP related to caching. In order to enable HTTP caching the server should attach proper cache headers to each response specifying the desired cache behavior.

HTTP cache is quite flexible. It allows servers to:

- Set restrictions on which responses are cacheable
- Set an expiration age for responses either using `max-age` (part of composite [Cache-Control](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9) header) and/or `Expires`
- Provide validators (`ETag`, `Last-Modified`) that are used to check stale responses with the server
- Force revalidation on each request

Here's an example of what you should look for in HTTP response headers:

```
HTTP/1.1 200 OK
Cache-Control: public, max-age=3600
Expires: Mon, 26 Jan 2016 17:45:57 GMT
Last-Modified: Mon, 12 Jan 2016 17:45:57 GMT
ETag: "686897696a7c876b7e"
```

This response is cacheable and it's going to be *fresh* for 1 hour. When the response becomes *stale*, the client validates it by making a *conditional* request using the `If-Modified-Since` and/or `If-None-Match` headers. If the response is still fresh the server returns status code `304 Not Modified` to instruct the client to use cached data, or it would return `200 OK` with a new data otherwise.

Most of the images are static assets that will not change in the future. The most viable caching strategy in this case is an *aggressive* caching. The server should simply set the `Cache-Control` header with a `max-age` value of a year in the future from the time of the request. It is recommended that `Expires` should be set to a similar value.

```
Cache-Control:public; max-age=31536000
Expires: Mon, 25 Jan 2017 17:45:57 GMT
```

*For more info about HTTP caching see some of the [guides from the reference list](#references) including [this one](https://devcenter.heroku.com/articles/increasing-application-performance-with-http-cache-headers).*

## Caching in Foundation's URL Loading System

Foundation framework provides [a set of classes](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html#//apple_ref/doc/uid/10000165-BCICJDHA) for communicating with the servers using standard internet protocols including HTTP. It also implements cache management:

* It has a composite on-disk and in-memory cache
* It's hip to cache control
* It handles revalidation *transparently*, you never have to deal with status code [304 (not modified)](http://www.codeproject.com/Articles/866319/HTTP-Not-Modified-An-Introduction)

There is no additional configuration required on the client side to make cache management work. However, there are certain things that you can do to make sure that the system would work the way you expect.

According to the [Apple's documentation](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSessionDataDelegate_protocol/index.html#//apple_ref/occ/intfm/NSURLSessionDataDelegate/URLSession:dataTask:willCacheResponse:completionHandler:), the responses are cached only when all of the following are true:

> - The request is for an HTTP or HTTPS URL (or your own custom networking protocol that supports caching).
- The request was successful (with a status code in the 200–299 range).
- The provided response came from the server, rather than out of the cache.
- The session configuration’s cache policy allows caching.
- The provided `NSURLRequest` object's cache policy (if applicable) allows caching.
- The cache-related headers in the server’s response (if present) allow caching.
- The response size is small enough to reasonably fit within the cache. (For example, if you provide a disk cache, the response must be no larger than about 5% of the disk cache size.)

Your app should already comply with most of them by default (for instance, the session configuration allows caching by default). However, one thing that a client should definitely do is to set an appropriate cache size.

## Configuring Caching

Let's dive a little deeper into URL Loading System and see which parts of it we can use to modify its caching behavior.

### NSURLCache

One of the classes in this system is a [NSURLCache](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSURLCache_Class/) which is a part of cache management. It provides a composite in-memory and on-disk cache. It provides methods to configure cache size and its location on disk. It also has methods to manage `NSCachedURLResponse` objects that contain the cached responses. This class isn't that useful by itself but it's important how it fits into the entire system.

### NSURLSession

The primary interface for URL Loading System is [NSURLSession](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSession_class/). It is a [huge improvement](https://www.objc.io/issues/5-ios7/from-nsurlconnection-to-nsurlsession/) over `NSURLConnection` which was deprecated in iOS 9. Some of those improvements apply to caching too. `NSURLSession` gives you a way to provide per-session cache (`NSURLCache`), cache policy (`NSURLRequestCachePolicy`), and set other options.

Let's focus on a [NSURLRequestCachePolicy](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSURLRequest_Class/#//apple_ref/c/tdef/NSURLRequestCachePolicy) which is a primary way for modifying caching behavior. The cache policy can be set either per `NSURLRequest`, or per `NSURLSession`. The default value is `.UseProtocolCachePolicy` which works as described in the example in "HTTP Caching" section. Some of the other most useful policies are:

- `.ReturnCacheDataDontLoad` - existing cache data should be used, regardless of its age or expiration date. If there is no existing data in the cache, no attempt is made to load the data. This policy might be useful when your app is in "offline" mode and you want to [show stale images](https://github.com/kean/Nuke/wiki/FAQ#my-app-is-offline-and-cached-images-are-not-showing) without having to validate them with a server.
- `.ReturnCacheDataElseLoad` - similar to the previous policy, but allows client to load data from the server. Sometimes it's convenient to set this property as a default to enable aggressive caching without worrying about "offline" mode. Note that this policy effectively disables cache validation.
- `.ReloadIgnoringCacheData` - always load data from the server. This option prevents cache data from ever being used, you can't use it to force validation.

`NSURLSession` also provides a comprehensive set of delegate methods. One those methods is [URLSessionSession(_:&#8203;dataTask:&#8203;willCacheResponse:&#8203;completionHandler:)](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSessionDataDelegate_protocol/index.html#//apple_ref/occ/intfm/NSURLSessionDataDelegate/URLSession:dataTask:willCacheResponse:completionHandler:) from `NSURLSessionDataDelegate` which. It might be used to prevent caching of specific URLs, providing a custom `userInfo` for cache responses and more. Note that this method is called only if the `NSURLSession` decides to cache the response. You can't use it to force `NSURLSession` to cache responses which headers explicitly disable caching.

One of the other great things about `NSURLSession` is that it has its own way of limiting the number of concurrent connections via `HTTPMaximumConnectionsPerHost` property of the `NSURLSessionConfiguration`. It only limits the number of HTTP connections and not the number of concurrent session tasks (`NSURLSessionTask`). Given that, if the client starts a new request which can be served by a fresh cached response then it would be served immediately, no matter how many other tasks are executing at the given moment.

*Again, I've just scratched the surface here, definitely check out [URL Session Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html#//apple_ref/doc/uid/10000165-BCICJDHA) if you haven't done that yet.*

## Memory Cache

The downside of `NSURLCache` is that its in-memory cache is not very useful because it stores instances of `NSCachedURLResponse` class which contain unprocessed `NSData`. We don't want to pollute precious RAM with that. What we need is a memory cache for fast access to decompressed images ready for display. Foundation already provides a class to do just that - `NSCache`. It is very straightforward to use, however it has some caveats.

Starting with iOS 7 `NSCache` will no longer remove cached objects automatically unless you explicitly set either its `totalCostLimit` or `countLimit`. The total cost limit is more flexible than the count limit, however, it requires clients to provide a reasonable cost with which to associate each cached object.

The obvious total cost limit is the number of bytes in memory. It might be computed as a percentage of available memory:

```swift
func totalCostLimit() -> Int {
    let physicalMemory = NSProcessInfo.processInfo().physicalMemory
    let ratio = physicalMemory <= (1024 * 1024 * 512 /* 512 Mb */) ? 0.1 : 0.2
    let limit = physicalMemory / UInt64(1 / ratio)
    return limit > UInt64(Int.max) ? Int.max : Int(limit)
}
```

Now let's compute a cost for `UIImage` object. Most of the space taken by `UIImage` is a bitmap which can be used to approximate its size in memory:

```swift
func costFor(image: UIImage) -> Int {
    let imageRef = image.CGImage
    return CGImageGetBytesPerRow(imageRef) * CGImageGetHeight(imageRef) // Cost in bytes
}
```

This configuration will provide arguably the best `NSCache` performance. Memory cache will hold a lot of images while still being under the certain limit. It's also important to immediately dispose of all cached objects when the app receives a memory warning.

The downside of a separate memory cache is that it doesn't have any expiration and validation mechanisms. It's fine for most use cases because `NSCache` evicts objects rather frequently. If you need more control over memory cache you can easily implement such features (expiration age, request policy, etc).

## <a name="references"></a>References

1. [Increasing Application Performance with HTTP Cache Headers](https://devcenter.heroku.com/articles/increasing-application-performance-with-http-cache-headers)
2. [RFC 7234. HTTP/1.1 Caching](https://tools.ietf.org/html/rfc7234)
3. [Cache-Control HTTP Headers](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9)
4. [URL Loading System Programming Guide. Understanding cache control.](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/URLLoadingSystem/Concepts/CachePolicies.html#//apple_ref/doc/uid/20001843-BAJEAIEE)
7. [Google Developers: HTTP caching](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=en)
8. [From NSURLConnection to NSURLSession](https://www.objc.io/issues/5-ios7/from-nsurlconnection-to-nsurlsession/)
