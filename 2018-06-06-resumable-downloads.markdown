---
layout: post
title: "Resumable Downloads"
description: Exploring how resumable downloads - one of my favorite new features in Nuke 7 - are implemented using HTTP range requests
date: 2018-06-06 18:00:00 +0300
category: programming
tags: programming
permalink: /post/resumable-downloads
uuid: 7ba0839a-0983-419a-b74e-43a75af31520
---

Resumable downloads were introduced in [Nuke 7](https://github.com/kean/Nuke/releases/tag/7.0). When the image download fails or gets canceled and the image is only partially loaded, the next request will resume where the previous one left off. This sounds like a must-have feature, but most image loading frameworks don't support it.

The resumable downloads are built using [HTTP range requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests). There are at least two ways to implement range requests with `URLSession`. The first one is to use `URLSessionDownloadTask` which has resumable downloads built-in, the second is to use `URLSessionDataTask` and handle HTTP range requests manually. I'm going to cover both in this article.

{% include ad-hor.html %}

## URLSession Download Tasks

[`URLSessionDownloadTask`](https://developer.apple.com/documentation/foundation/urlsessiondownloadtask) handles all the intricacies of HTTP range requests for you. Let's quickly go through how to use `URLSessionDownloadTasks`.

> `URLSession` has two ways of using session tasks (`URLSessionTask`) - the convenience closure-based way, and the delegate-based way. I'm going to focus on the latter.

Unlike other `URLSessionTask` subclasses there are two ways to create a download task - either with a `URL` (or `URLRequest`) or with a resumable data.

```swift
let urlSession: URLSession
var resumableData: Data?

func startRequest() {
    let url = URL(string: "https://example.com/image.jpeg")!
    if let resumableData = self.resumableData {
        urlSession.downloadTask(withResumeData: resumableData).resume()
    } else {
        urlSession.downloadTask(with: url).resume()
    }
}
```

> Actually there is also a third indirect way to create `URLSessionDownloadTask`. You can implement `func urlSession(_:dataTask:didReceive response:completionHandler:)` method from `URLSessionDataTaskDelegate` protocol and call completion handler with `ResponseDisposition` `.decomeDownload` in this method.

When you create `URLSessionDownloadTask` for the given URL for the first time you do so by passing either `URL` or `URLRequest` in initializer. But if you already have resumable data, the request is no longer needed - it's stored as part of the resumable data. But where does the resumable data come from?

You can retrieve resumable data from the `userInfo` of the task's error using [`NSURLSessionDownloadTaskResumeData`](https://developer.apple.com/documentation/foundation/nsurlsessiondownloadtaskresumedata) key. The error is going to contain the resumable data when either a transfer error occurs or when you call `cancel(byProducingResumeData:)` method (if you call `cancel()` the resumable data is not produced).

> The server must also indicate that it supports HTTP range request for resumable data to be produced.

```swift
func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
    if let resumableData = (error as? URLError)?.userInfo[NSURLSessionDownloadTaskResumeData] as? Data  {
        // Store resumable data somewhere
    }
}
```

When you receive resumable data you need to store it somewhere depending on what your requirements are.

`URLSessionDownloadTask` works great but has a few limitations:

- It can read from `URLCache` (native HTTP cache) but it's not going to save responses there. If you'd like to do that, you'll have to do it manually.
- There is always disk I/O happening which is not always desirable.
- It doesn't seem to support unconditional HTTP range requests, the documentation says that at least one HTTP validator must be present - either `ETag` or `Last-Modified`.

Download tasks are designed to be used when you actually want to _download_ some data to disk. This isn't what `URLSession` is used for in Nuke. It uses `URLSessionDataTasks` and fortunately, there is a way to make them support HTTP range requests.

## URLSession Data Tasks

Lets first quickly go through the [HTTP range requests spec](https://tools.ietf.org/html/rfc7233) and then see how it can be implemented using Swift and `URLSessionDataTask`.

### HTTP Range Requests

If the [`Accept-Ranges`](https://tools.ietf.org/html/rfc7233#section-2.3) header is present in the response and has value `bytes`, the server supports range requests:

```
curl -I https://cloud.githubusercontent.com/assets/1567433/9781817/ecb16e82-57a0-11e5-9b43-6b4f52659997.jpg

HTTP/1.1 200 OK
...
Accept-Ranges: bytes
Content-Length: 31038
```

To issue a range request pass the [`Range`](https://tools.ietf.org/html/rfc7233#section-3.1) of bytes that you'd like the server to send:

```
curl https://cloud.githubusercontent.com/assets/1567433/9781817/ecb16e82-57a0-11e5-9b43-6b4f52659997.jpg \
    -i -H "Range: bytes=30000-"
```

> There are different ways to format `Range` value. In our case, we use one-sided range which means that we want all the remaining data starting from index `30000`.

If everything goes well the server will respond with status code [`206 Partial Content`](https://tools.ietf.org/html/rfc7233#section-4.1) and the requested data:

```
HTTP/1.1 206 Partial Content
...
Content-Range: bytes 30000-31037/31038
Content-Length: 1038
Accept-Ranges: bytes
...
(binary)
```

When resuming the request that contains a validator - either [`Last-Modified`](https://tools.ietf.org/html/rfc7232#section-2.2) or [`ETag`](https://tools.ietf.org/html/rfc7232#section-2.3) - you can also provide an optional [`If-Range`](https://tools.ietf.org/html/rfc7233#section-3.2) header in the request which makes a request _conditional_. It means that if the representation is unchanged, send the requested part; otherwise, send the entire representation (with status code `200 OK`).

If the client doesn't send `If-Range` header but makes a request conditional by providing either or both `If-Unmodified-Since` and `If-Match`, then if the condition fails the request is also going to fail with status code `412 Precondition Failed` which isn't very convenient.

```
curl https://cloud.githubusercontent.com/assets/1567433/9781817/ecb16e82-57a0-11e5-9b43-6b4f52659997.jpg \
    -i -H "Range: bytes=0-1023" -H "If-Match: aa2fa107a9618d19df8d37dd0e40d2fa"

HTTP/1.1 412 Precondition Failed
```

> Mozilla is doing a fantastic job documenting web technologies. If you'd to learn a bit more check their [HTTP Range Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests) guide.

### Implementing ResumableData

First, let's create a type that would implement the HTTP range requests spec. Let's start by defining a type itself and its initializer:

```swift
struct ResumableData {
    let data: Data
    let validator: String // Either `Last-Modified` or `ETag`

    // - returns `nil` if the request can't be resumed. 
    init?(response: URLResponse, data: Data) {
        // Check if "Accept-Ranges" is present and the response is valid.
        guard !data.isEmpty,
            let response = response as? HTTPURLResponse,
            response.statusCode == 200 /* OK */ || response.statusCode == 206, /* Partial Content */
            let acceptRanges = response.allHeaderFields["Accept-Ranges"] as? String,
            acceptRanges.lowercased() == "bytes",
            let validator = ResumableData.validator(from: response) else {
                return nil
        }
        self.data = data
        self.validator = validator
    }

    private static func validator(from response: HTTPURLResponse) -> String? {
        if let entityTag = response.allHeaderFields["ETag"] as? String {
            return entityTag
        }
        // There seems to be a bug with ETag where HTTPURLResponse would canonicalize
        // it to Etag instead of ETag
        // https://bugs.swift.org/browse/SR-2429
        if let entityTag = response.allHeaderFields["Etag"] as? String {
            return entityTag
        }
        if let lastModified = response.allHeaderFields["Last-Modified"] as? String {
            return lastModified
        }
        return nil
    }
}
```

We also need methods that we could use to "resume" a request:

```swift
func resume(request: inout URLRequest) {
    var headers = request.allHTTPHeaderFields ?? [:]
    headers["Range"] = "bytes=\(data.count)-"
    headers["If-Range"] = validator
    request.allHTTPHeaderFields = headers
}

// Check if the server resumed the response.
static func isResumedResponse(_ response: URLResponse) -> Bool {
    return (response as? HTTPURLResponse)?.statusCode == 206
}
```

> `ResumableData` is covered by a complete suite of unit tests in Nuke to make sure the HTTP spec is implemented correctly.

Now that we have `ResumableData` in place it's only a matter of using it when working with data tasks.

### Using ResumableData

When the data task fails we need to try and save the resumable data somewhere.

```swift
func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
    if let error = error, let response = task.urlResponse,
        let resumableData = ResumableData(response: response, data: session.data) {
        storeResumableData(resumableData, for: session.request.urlRequest)
    }
}
```

When we create a new data task we need to check whether we have a resumable data for the request.

When we receive the response we need to check whether it was resumed and then append the data to the existing resumable data, or if we received a new response (`200 OK`) discard the resumable data.

```swift
func urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive data: Data) {
    guard let handler = self.handlers[dataTask] else { return }
    if let resumableData = handler.resumableData {
        // See if the server confirmed that we can use the resumable data.
        if ResumableData.isResumedResponse(response) {
            handler.data = resumableData.data
        }
        handler.resumableData = nil
    }
    handler.data.append(data) // Append new data
}
```

> You might have noticed that `ResumableData` doesn't support unconditional range requests. This matches the [`URLSessionDownloadTask` behavior](https://developer.apple.com/documentation/foundation/urlsessiondownloadtask/1411634-cancel) which states that a download can only be resumed if the server provides either the ETag or Last-Modified header (or both) in its response.
>
> I'm not entirely sure why this is the case, please leave a comment if you know. My guess is that range requests are somewhat dangerous. If the server returns `Accept-Ranges` but fails to send validators for content which can actually change in the future, the client might end up downloading parts of different resources and combining them together.

## Resumable Downloads in Nuke

It was very satisfying to implement resumable downloads in Nuke and see them in action. I think it can be a major improvement to the user experience, especially on mobile networks.

{% include references-start.html %}

- [Nuke](https://kean.github.io/nuke)
- [MDN web docs: HTTP Range Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests)
- [RFC 7233. Hypertext Transfer Protocol (HTTP/1.1): Range Requests](https://tools.ietf.org/html/rfc7233)
- [Apple Developer Library: URLSession](https://developer.apple.com/documentation/foundation/urlsession)

{% include references-end.html %}