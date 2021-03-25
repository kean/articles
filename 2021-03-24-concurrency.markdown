---
layout: post
title: "Concurrency"
subtitle: The actor model and other concurrency patterns used in Nuke
description: The actor model and other concurrency patterns used in Nuke
date: 2021-03-24 10:00:00 -0500
category: programming
tags: programming
permalink: /post/concurrency
uuid: 199e4057-2c6b-41ef-87c3-1ad6ef934fa8
image:
  path: /images/posts/concurrency/concurrency-cover.png
  height: 1280
  width: 640
---

Some apps can afford to have _no_ concurrency. The rest are the reason why books on concurrency exist[^3].

[^3]: Or are Xcode [blocking the main thread](https://twitter.com/a_grebenyuk/status/1371519070122692609?s=20) for 20+ seconds at a time when checking run destinations with WiFi deployment enabled. Sorry, Xcode, I didn't mean to be mean.

I have two examples from my recent experience. [Pulse](https://github.com/kean/Pulse) has practically no concurrency, no parallelism, and does _everything_ on the main thread. On the other end of the spectrum is [Nuke](https://github.com/kean/Nuke) that has to be massively concurrent, parallel, and thread-safe.

Nuke is often used during scrolling, so it has to be fast and never add unnecessary contention to the main thread. This is why Nuke does _nothing_ on the main thread. At the same time, it requires very few context switches. And that's the key to its performance (among numerous other performance-related [features](https://kean.blog/post/nuke-9)).

<div style="max-width:540px;" class="BlogVideo">
<video controls muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/rate_limiter.mp4" type="video/mp4">
</video>
</div>

There isn't much to say about Pulse: you can't have threading issues if you [don't have]({{ site.url }}/images/misc/m1.jpg) threading. But when you need it, concurrency is notoriously hard to get right. So what do you do?

## Overview

If you group the primary components in Nuke based on the concurrency primitives, you'll end up with roughly four groups. There isn't one pattern that dominates. Each situation requires a unique approach.

<img class="NewScreenshot" src="{{ site.url }}/images/posts/concurrency/concurrency-cover.png">

There is a tiny portion of code that is not thread-safe and can only run from the main thread because it updates UI components, be it UIKit or SwiftUI. This code has assertions to catch errors early: `assert(Thread.isMainThread)`. The rest of the code operates in the background and this is what the article covers.

## Actor Model

> [Actors](https://en.wikipedia.org/wiki/Actor_model) allow you as a programmer to declare that a bag of state is held within a concurrency domain and then define multiple operations that act upon it. Each actor protects its data through data isolation, ensuring that only a single thread will access that data at a given time, even when many clients are concurrently making requests of the actor.
>
> *From [SE-0306](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)*

The actor model is a pattern. It doesn't *have* to be included in the language, but the goal of the Swift team is to add to one of the main Swift strengths: safety via compile-time checks. With the actor model, you will get isolation checks at compile-time along with other safety features.

I've been designing classes as Actors in Objective-C since [GCD](https://developer.apple.com/documentation/DISPATCH) was added in iOS 4. I'm designing Swift classes as Actors now. And I can't wait to start designing them using a formal actor model.

> While the formal definition seems a bit complicated, you can simply think of an actor as a class with some private state and an associated *serial* dispatch queue[^7]. The only way to access the state is to do it on the serial queue. It ensures there are never any [data races](https://en.wikipedia.org/wiki/Race_condition#Data_race).
{:.info}

[^7]: It appears that the actual implementation in Swift won't be using dispatch queues. I'm also curious to learn more about a lighter-weight implementation of the actor runtime mentioned in the Swift Evolution proposal that _isn't_ based on serial `DispatchQueue`.

I designed most of the public components in Nuke that have to be thread-safe as actors: `ImagePipeline`, `ImagePrefetcher`, `DataLoader`.

### ImagePipeline

[`ImagePipeline`](https://github.com/kean/Nuke#image-pipeline) is the main component you interact with as a user to fetch images. It has to be fast because you often use it during scrolling, and it has to be thread-safe. The pipeline (or most of it in the current iteration) is designed as an actor.

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

Now the pipeline is fully thread-safe _and_ does its work in the background reducing contention on the main thread. Why is it important? The pipeline does quite a lot: coalescing, creating, and starting URL requests, which can take up to `0.6ms` on older devices. Apps can't always afford it when scrolling a collection view with multiple images per row. Scheduling async work on a [`DispatchQueue`](https://developer.apple.com/documentation/dispatch/dispatchqueue), on the other hand, is relatively fast – maybe a few *micro*seconds. It's not free but is faster than the work in this case.

### ImagePrefetcher

[`ImagePrefetcher`](https://github.com/kean/Nuke#image-preheating) helps you manage image prefetching per screen and is also an Actor.

```swift
/// Not the actual implementation, just a demonstration
public final class ImagePrefetch {
    private let pipeline: ImagePipeline
    private let queue = DispatchQueue(label: "ImagePrefetch", qos: .userInitiated))

    /// Private state...
    private let tasks = [URL: ImageTask]()

    public func startPrefetching(urls: [URL]) {
        queue.async {
            for url in urls {
                self.tasks[url] = self.pipeline.loadImage(with: url) { [weak self] in
                    self?.didCompleteTask(url: url)
                }
            }
        }
    }

    private func didCompleteTask(url: URL) {
        queue.async {
            self.tasks[url] = nil
        }
    }
}
```

Now here is a problem. Let's say you start prefetching images with 4 URLs. How many `queue.async` calls can you count?

- 1 made by the prefetcher to access its state in `startPrefetching(urls:)`
- 4 made by the pipeline to access _its_ state in `loadImage(url:)`
- 4 made by the prefetcher, one per completion callback

That's a bit too much. Can we optimize it?

One option is to forget about thread-safety and background execution and synchronize on the main thread. But prefetcher is also used during scrolling. A collection view [prefetch API](https://developer.apple.com/documentation/uikit/uicollectionviewdatasourceprefetching) can ask you to start prefetching for 20 or more cells at a time. If sending it to background is faster than executing on the main queue, I would prefer to do it. When [120Hz displays](https://www.macrumors.com/2020/12/28/120hz-promotion-display-iphone-13/) for iPhones drop, you will be happy to get any optimizations you can.

> Always measure. Sending work to the background will often be _slower_ than executing it synchronously on the current thread and synchronizing with locks.
{:.warning}

To avoid the excessive number of context switches, Nuke synchronizes two actors (pipleine and prefetcher) on a single dispatch queue, dropping the number of `queue.async` calls to just one!

> Can you still call this approach an actor model? I think it's a stretch, but I will let it slide. I definitely won't be able to implement it this way with just features from [SE-0306: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md), but [Global Actors Pitch](https://github.com/DougGregor/swift-evolution/blob/global-actors/proposals/nnnn-global-actors.md) hints at a solution. One could also think about this as a confirmation that actors synchronization can be too granular.

### DataLoader

I apply a similar optimization to the default data loader (`DataLoader`). When the pipeline is initialized, it injects the synchronization queue into the data loader. And it gets even more exciting!

This is the initial implementation of the `DataLoader`:

```swift
/// Not the actual implementation, just a demonstration
public final class DataLoader {
    public init() {
        let queue = OperationQueue()
        queue.maxConcurrentOperationCount = 1
        session = URLSession(configuration:configuration, delegate: self, delegateQueue: queue)
    }

    func loadData(for request: URLRequest, completion: @escaping (Data?) -> Void) {
        let task = session.dataTask(with: request)
        let handler = _Handler(didReceiveData: didReceiveData, completion: completion)
        session.delegateQueue.addOperation { // Dispatch #1
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

    /// ...
}

private final class LoadDataTask {
    func didReceiveData(_ data: Data) {
        pipeline.queue.async { // Dispatch #2
            // ...
        }
    }
}
```

`DataLoader` used a background operation queue as a [`URLSession`](https://developer.apple.com/documentation/foundation/urlsession) delegate queue, which is great because it reduces contention on the main thread. But it is a new private queue – the same problem as with the prefetcher. Here is a trick, [`OperationQueue`](https://developer.apple.com/documentation/foundation/operationqueue) allows you to set an [`underlyingQueue`](https://developer.apple.com/documentation/foundation/operationqueue/1415344-underlyingqueue). In the optimized version, `DataLoader` uses the pipeline's serial queue as a delegate queue.

```swift
extension DataLoader {
    func inject(_ queue: DispatchQueue) {
        session.delegateQueue.underlyingQueue = queue
    }
}
```

Now the entire system is synchronized on a single serial dispatch queue!

### Main Queue?

If you synchronize on the main queue, the important words are "synchronizing" and "queue". The queue doesn't have to be your main queue. Using the main queue does make it easier to avoid UI updates from the background queue, but [Thread Sanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html) catches those immediately, so it's relatively easy to avoid.

There is also a flaw in synchronizing on main. If your subsystems assume they are only called from a single queue, they are not thread-safe. If you accidentally call any of them from the background, you can _introduce subtle bugs_ which are harder to find than simple UI updates from the background, unless you fill your code with assertions. And if your app becomes big enough that it can no longer afford to operate strickly on the main queue, you are going to be up to major rethinking of your components (been there).

If you need thread-safety and mutable state, the actor model is probably your best bet. From the two [concurrency models](https://web.mit.edu/6.005/www/fa14/classes/17-concurrency/): shared memory and messages passing, I find the latter to be more appealing.

## Tasks

Nuke has an incredible number of performance [features](https://kean.blog/post/nuke-9): progressive decoding, prioritization, coalescing of tasks, cooperative cancellation, parallel processing, backpressure, prefetching. It forces Nuke to be massively concurrent. The actor modal is just part of the solution. To manage individual image requests, it needed a structured approach for async tasks.

The solution is [`Task`](https://github.com/kean/Nuke/blob/93c187ab98ab02f8c891d1fa40ffe92a1591f524/Sources/Tasks/Task.swift#L18), a is part of the internal infrastructure. I don't want to go into too much detail because it's impossible to cover in one post, so I'll just throw this complex diagram here.

<img class="NewScreenshot" src="{{ site.url }}/images/posts/concurrency/tasks.png">

When you request an image, Nuke creates a dependency tree with multiple tasks. When a similar image request arrives (e.g. the same URL, but different processors), an existing subtree can serve as a dependency of another task. Nuke has to manage prioritization and cooperativ ecancellation, so I sane structured approach was a must.

Tasks perform their work incrementally (to support progressive decoding) so they are inspired by reactive programming, but are optimized for Nuke. Tasks are much simpler and faster than a typical generalized reactive programming implementation. The complete implementation takes just 237 lines.

Will Nuke adopt [async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md) for its internal infrastructure? Unlikely, as it probably won't cover all the scenarios I need. But I am considering replacing tasks with [Combine](https://developer.apple.com/documentation/combine) when Nuke drops iOS 12 support.

## Locks

Nuke also extensively uses locks([`NSLock`](https://developer.apple.com/documentation/foundation/nslock)) for synchronization. `ImageCache`, `DataCache`, `ResumableData`  – there are a few components that use them. They all have public synchronous APIs, so they have to be thread-safe.

> Locks allow you to protect a critical section of code. You [`lock()`](https://developer.apple.com/documentation/foundation/nslocking/1416318-lock) when the section starts, and [`unlock()`](https://developer.apple.com/documentation/foundation/nslocking/1418241-unlock) when you are done. Unlike semaphores, unlock has to be sent from the same thread that sent the initial lock messages.
{:.info}

There isn't much to say about locks. There are easy to use and fast[^6]. I don't trust any of the performance benchmarks that measure the performance difference between different synchronization instruments. If there even is a measurable difference, it's irrelevant in the scenarios where I use them.

[^6]: Especially after the recent-ish performance optimizations, which I can't find a link for right now. Locks are especially efficient in situations where there isn't a lot of contention, so they don't need to go to the kernel level.

> `DataCache` is a bit more complicated than that. It writes data asynchronously and does so in parallel to reads, while reads can also be parallel to each other. It's a bit experimental, I haven't tested the full impact of this approach on the performance.

## Atomics

I had been using atomic operations (`OSAtomicIncrement64`, `OSCompareAndSwap`) in a couple of places in Nuke before, for example for generating unique task IDs in the pipeline. But when Thread Sanizer was introduced it started emiting warnings[^8].

[^8]: You can learn more about why these warnings get emitted in the [following post](http://www.russbishop.net/the-law).

<img class="NewScreenshot" src="{{ site.url }}/images/posts/concurrency/access-race.png">

I replaced all atomics usages with locks or refactored some code to not require synchronization in the first place. There wasn't much impact on performance for the scenarios where I was using them. I will revisit this topic when/if [Swift Atomics](https://swift.org/blog/swift-atomics/) are part of the Standard Library.

## OperationQueues

The expensive operations, e.g. decoding, processing, data loading, are modeled using [`Foundation.Operation`](https://developer.apple.com/documentation/foundation/operation). Operation [queues](https://developer.apple.com/documentation/foundation/operationqueue) are used for parallelism with a limited number of concurrent operations. They are also used for prioritization which is crucial for some user scenarios. There isn't much else to say about operation queues that haven't already been said.

## Conclusion

It's always best to avoid concurrency or mutable state. But sometimes it's not feasible and to write responsive client-side programs[^5], you often have to embrace concurrency. There are, of course, kinds of tasks that obviously have to be asynchronous, such as networking. But if you want to use background processing as an optimization, always measure! Performance is about doing less, not more.

[^5]: And servers! Modern server-side frameworks, such as [SwiftNIO](https://github.com/apple/swift-nio) also embraced concurrency and non-blocking I/O.

Measuring is an art on its own. Modern CPUs with different cache levels and other optimizations make it hard to reasons about performance. But at least make sure to measure in Release mode (with compiler optimizations on) and check that you are getting consistent results. The absolute measurements might not always be accurate, but the relative values will guide you in the right direction. Avoid premature optimization, but also don't paint yourself into a corner where the only way to improve performance is to rewrite half of the app.

Nuke has many optimizations, some impractical, so don't just go and copy them. I just finished watching [F1 Season 3](https://www.formula1.com/en/latest/article.watch-the-unmissable-trailer-for-season-3-of-netflixs-drive-to-survive.1ZMsRiYIvxZEPV9iqzatsx.html) (which was phenomenal, by the way). I can't say that I'm a huge F1 fan, but I appreciate things designed for performance and a bit of competition. I'm not stopping where a sane person should.

<div class="References" markdown="1">

<h2 class="PostLink SectionTitle">References</h2>

- [**Nuke**](https://github.com/kean/Nuke), an image loading and caching system
- [**Pulse**](https://github.com/kean/Pulse), a structured logging system
- [**SE-0306**: Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)

</div>

<div class="FootnotesSection" markdown="1">
</div>
