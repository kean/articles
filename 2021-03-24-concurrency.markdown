---
layout: post
title: "Concurrency Done Right"
subtitle: The actor model and other concurrency patterns used in Nuke
description: The actor model and other concurrency patterns used in Nuke
date: 2021-03-24 10:00:00 -0500
category: programming
tags: programming
permalink: /post/concurrency
uuid: 199e4057-2c6b-41ef-87c3-1ad6ef934fa8
---

<blockquote class="quotation">
   <p>The best way to handle concurrency is just to not do it.</p>
</blockquote>

Some apps can afford not to. The rest are the reason why books on concurrency exist[^3].

[^3]: Or are Xcode [blocking the main thread](https://twitter.com/a_grebenyuk/status/1371519070122692609?s=20) for 20+ seconds at a time when checking run destinations with WiFi deployment enabled. Sorry, Xcode, I didn't mean to be mean.

I have two examples from my experience. [Pulse](https://github.com/kean/Pulse) has practically no concurrency, no parallelism, and does _everything_ on the main thread. On the other end of the spectrum is [Nuke](https://github.com/kean/Nuke) that has to be massively concurrent, parallel, and, of course, thread-safe.

Nuke is often used during scrolling, so it has to be fast and never add unnecessary contention to the main thread. This is why Nuke does _nothing_ on the main thread. At the same time, it requires very few context switches. And that's the key to its performance (among numerous other performance-related [features](https://kean.blog/post/nuke-9)).

<div class="BlogVideo">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/rate_limiter.mp4" type="video/mp4">
</video>
</div>

There isn't much to say about Pulse: [you can't have]({{ site.url }}/images/misc/m1.jpg) threading issues if you don't have threading. But when you need concurrency, it's notoriously hard to get it right. So what do you do?

## Actor Model

> [Actors](https://en.wikipedia.org/wiki/Actor_model) allow you as a programmer to declare that a bag of state is held within a concurrency domain and then define multiple operations that act upon it. Each actor protects its data through data isolation, ensuring that only a single thread will access that data at a given time, even when many clients are concurrently making requests of the actor.
>
> *From [SE-0306](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)*

The actor model is a pattern. It doesn't *have* to be included in the language, but the goal of the Swift team is to add to one of the main Swift strengths: safety via compile-time checks. With the actor model, you will get isolation checks at compile-time along with other safety features.

In the meantime, I've been designing classes as Actors in Objective-C since [GCD](https://developer.apple.com/documentation/DISPATCH) was added in iOS 4. I'm designing Swift classes as Actors now. And I can't wait to start designing them using a formal actor model.

I designed most of the user-facing components in Nuke that has to be thread-safe as actors: `ImagePipeline`, `ImagePrefetcher`, `DataLoader`.

### ImagePipeline

[`ImagePipeline`](https://github.com/kean/Nuke#image-pipeline) is the main component you interact with as a user to fetch images. It has to be fast because you often use it during scrolling, and it has to be thread-safe. The pipeline (or most of it in the current iteration) is designed as an actor. All you need to do to implement it is create a [`DispatchQueue`](https://developer.apple.com/documentation/dispatch/dispatchqueue) and perform work exclusively in a critical section provided by it.

```swift
/// Not the actual implementation, just a demonstration
public final class ImagePipeline {
    private let queue = DispatchQueue(label: "ImagePipeline", qos: .userInitiated))

    /// Private state ...
    private var tasks = [ImageTask: TaskSubscription]()

    public func loadImage(_ url: URL, completion: @escaping (Image?) -> Void) {
        queue.async {
            // Access instance properties in a thread-safe manner and act upon them
        }
    }
}
```

Now the pipeline is fully thread-safe _and_ does its work in the background reducing contention on the main thread. Why is it important? The pipeline does quite a lot: coalescing, creating, and starting URL requests, which can take up to `0.6ms` on older devices. Apps can't afford it when scrolling a collection view with multiple images per row. Scheduling async work on a [`DispatchQueue`](https://developer.apple.com/documentation/dispatch/dispatchqueue), on the other hand, is fast – a couple of *micro*seconds. It's not free but is significantly faster than the work in this case.

> In the current implementation, `ImagePipeline` also returns an `ImageTask` instance from `loadImage()` method. It won't fly with an actor model. Instead of making `ImagePipeline` itself an actor, I'll consider factoring out most of its implementation into a separate private component that can be.

### ImagePrefetcher

[`ImagePrefetcher`](https://github.com/kean/Nuke#image-preheating) is used for prefetching images (duh). You create one per screen to help you manage prefetching. `ImagePrefetcher` is also an Actor.

```swift
/// Not the actual implementation, just a demonstration
public final class ImagePrefetch {
    private let pipeline: ImagePipeline
    private let queue = DispatchQueue(label: "ImagePrefetch", qos: .userInitiated))

    /// Private state...
    private let tasks = [URL: ImageTask]()

    public func startPrefetching(for urls: [URL]) {
        queue.async {
            for url in urls {
                self.tasks[url] = self.pipeline.loadImage(url: url) { [weak self] in
                    self.queue.async {
                        self.tasks[url] = nil
                    }
                }
            }
        }
    }

    public func stopPrefetching(for urls: [URL]) {
        // Find pending tasks associated with the given URLs and cancel them
    }
}
```

Now here is a problem. Let's say you start prefetching for 4 URLs. How many `queue.async` calls can you count?

- 1 made by the prefetcher to access its state
- 4 made by the pipeline to in turn access _its_ state in `loadImage(url:)`
- 4 made by the prefetcher, one per completion callback

That's a bit much and probably adds up to more than the actual work being performed by these classes. Can we optimize it?

One option is to forget about background execution and thread-safety and synchronize on the main thread. But remember, prefetcher is also used during scrolling. A collection view [prefetch API](https://developer.apple.com/documentation/uikit/uicollectionviewdatasourceprefetching) can ask you to start prefetching for 20 or more cells at a time. The amount of work is significant enough to justify sending it to the background.

Instead, Nuke synchronizes two actors (`ImagePipeline` and `ImagePrefetcher`) on a single dispatch queue. When `ImagePrefetcher` calls `ImagePipeline`, you are already on the syncrhonization queue, so the pipeline doesn't have to do any synchronization. And when the task is completed, `ImagePipeline` is already on its serial queue, so dispatch is not needed again. From 9 `queue.async` calls we are down to just one!

### DataLoader

I apply a similar optimization to the default data loader (`DataLoader`). When the pipeline is initialized, it injects the synchronization queue into the data loader. Nuke can't perform this optimization for custom data loaders (types that implement `DataLoading` protocol where all bets are off), but it can for the default one. And it gets even more exciting!

Here is the initial implementation of the `DataLoader` and the pipeline task that uses it:

```swift
/// Not the actual implementation, just a demonstration
public final class DataLoader {
    public init() {
        let queue = OperationQueue()
        queue.maxConcurrentOperationCount = 1
        session = URLSession(configuration:configuration, delegate: self, delegateQueue: queue)
    }

    /// ...

    func loadData(for request: URLRequest, completion: @escaping (Data?) -> Void) {
        let task = session.dataTask(with: request)
        let handler = _Handler(didReceiveData: didReceiveData, completion: completion)
        // Dispatch #1
        session.delegateQueue.addOperation {
            self.handlers[task] = handler
        }
        task.resume()
        return task
    }

    func dataTask(_ dataTask: URLSessionDataTask, didReceive data: Data) {
        guard let handler = handlers[dataTask] else {
            return
        }
        handler.didReceiveData(data)
    }
}

private final class LoadDataTask {
    func didReceiveData(_ data: Data) {
        // Dispatch #2
        pipeline.queue.async {
            // ...
        }
    }
}
```

`DataLoader` uses a background [operation queue](https://developer.apple.com/documentation/foundation/operationqueue) as a [`URLSession`](https://developer.apple.com/documentation/foundation/urlsession) delegate queue, which is great because it reduces contention on the main thread. But it is a new private queue – the same problem as with the prefetcher. But here is a trick, `OperationQueue` allows you to set an [`undelryingQueue`](https://developer.apple.com/documentation/foundation/operationqueue/1415344-underlyingqueue). In the optimized version, `DataLoader` uses the pipeline's serial queue as a delegate queue.

```swift
extension DataLoader {
    func inject(_ queue: DispatchQueue) {
        session.delegateQueue.underlyingQueue = queue
    }
}
```

With this optimization, we can get rid of _all_ synchronization calls in `DataLoader` (see dispatch #1 and #2).

### Main Queue?

When you hear advice about synchronizing on the main queue, the important words here are "synchronizing" and "queue". The queue doesn't have to be your main queue. Using the main queue does make it easier to avoid UI updates from the background queue, but [Thread Sanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html) catches those immediately, so it's relatively easy to avoid.

There is also a major flaw in the "just synchronize on main" approach. If you do, it means none of your subsystems are thread-safe. If you accidentally call any of them from the background, you can _introduce subtle bugs_ which are much harder to find than background UI updates. And it gets worse. If your app becomes big enough that it can no longer afford to operate strickly on the main queue, you are going to be up to major rethinking of your components (been there).

The actor model is a great approach to achieve thread-safety and performance. One thing to keep in mind with the dispatch queues is that switching execution contexts is not free. There is such thing as too much of a good thing. Nuke solves it by synchronizing all of its subsystems on a single serial dispatch queue.

## Tasks

Nuke has an incredible number of performance [features](https://kean.blog/post/nuke-9): progressive decoding, prioritization, coalescing of tasks, cooperative cancellation, parallel processing, backpressure, prefetching. It forces Nuke to be massively concurrent. Actor Modal is just part of the solution. To manage individual image downloads, it needs something more.

The solution is `Task`. `Task` is part of the internal infrastructure built specifically for Nuke. I don't want to go into too much detail on this because it's impossible to cover in one post, so I'll just throw this complex diagram here.

<img class="NewScreenshot" src="{{ site.url }}/images/posts/concurrency/tasks.png">

Tasks are inspired by reactive programming but are optimized for Nuke. Tasks are much simpler and faster than a typical generalized reactive programming implementation. The complete implementation takes just 237 lines.

Will Nuke adopt [async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md) for its internal when it's available? Highly unlikely, as it won't cover all the scenarios I need. But I am considering replacing tasks with [Combine](https://developer.apple.com/documentation/combine) when Nuke drops iOS 12 support.

## Locks

Nuke also extensively uses locks([`NSLock`](https://developer.apple.com/documentation/foundation/nslock)) for syncrhonization. `ImageCache`, `DataCache`, `ResumableData`  – there are plenty of components that use them. They all have public synchronous APIs, so they have to be thread-safe.

There isn't much to say about locks. There are easy to use and efficient. I don't trust any of the performance benchmarks that measure the performance difference between different synchronization instruments. If there even is a measurable difference, it's irrelevant in the scenarios where I use them.

## Atomics

I've been using atomic operations (`OSAtomicIncrement64`, `OSCompareAndSwap`) in a couple of places in Nuke before, for example for generating unique task IDs in the pipeline. when Thread Sanizer was introduced it started emiting fasle positive warnings to the users, so I replace all instances of CAS with locks or refactored some code to not require syncrhonization in the first place. There wasn't much impact on performance for the scenarios where I was using them.

<img class="NewScreenshot" src="{{ site.url }}/images/posts/concurrency/access-race.png">

I will revisit this topic when/if [Swift Atomics](https://swift.org/blog/swift-atomics/) are part of the Standard Library.

## OperationQueues

The expensive operations, e.g. decoding, processing, data loading, are modeled using operations ([`Foundation.Operation`](https://developer.apple.com/documentation/foundation/operation). Operation queues are used for parallelism with a limited number of concurrent operations to avoid a thread explosion problem. They are also used for prioritization which is crucial for some user scenarios.

## Conclusion

To write responsive client-side programs[^5], you often have to embrace concurrency. There are, of course, applications that barely need it. Take [Pulse](https://github.com/kean/Pulse) for example. All it does is perform database queries on the main thread, and it's fast enough. There are, of course, kinds of tasks that obviously have to be asynchronous, such as networking. But if you want to use background processing as an optimization, always measure!

[^5]: And servers! Modern server-side frameworks, such as [SwiftNIO](https://github.com/apple/swift-nio) also embraced concurrency and non-blocking I/O.

Your assumptions about performance often won’t match the reality. Measuring is an art on its own. Modern CPUs with different cache levels and other optimizations make it hard to reasons about performance. But at least make sure to measure in Release mode (with compiler optimizations on) and check that you are getting consistent results. The absolute measurements might not always be accurate, but the relative values should give you confidence that your changes improve performance, not degrade it. Never do premature optimizations, but also don't paint yourself into a corner where the only way to improve performance is to rewrite half of the app.

Nuke has a lot of optimizations, some are not practical, so don't just go and copy them. I just finished watched [F1 Season 3](https://www.formula1.com/en/latest/article.watch-the-unmissable-trailer-for-season-3-of-netflixs-drive-to-survive.1ZMsRiYIvxZEPV9iqzatsx.html) (which was fantastic by the way) in basically two sittings. Not that's I'm a huge fan of F1, but I like things to go fast. I'm not stopping where any sane person should.

<div class="References" markdown="1">

<h2 class="PostLink SectionTitle">References</h2>

- [**Nuke**](https://github.com/kean/Nuke), an image loading and caching system
- [**Pulse**](https://github.com/kean/Pulse), a structured logging system
- [**SE-0306**: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)

</div>

<div class="FootnotesSection" markdown="1">
</div>

