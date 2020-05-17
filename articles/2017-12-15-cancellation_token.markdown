---
layout: post
title: "Cancellation Token"
description: "A unified model for cooperative cancellation of asynchronous operations"
date: 2017-12-15 18:00:00 +0300
category: programming
tags: ios
permalink: /post/cancellation-token
uuid: f6bfea67-880f-462f-a08c-5b065a70f573
---

The cancellation tokens have recently (turns out actually not so recently!) [surfaced in Swift Evolution](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170821/039226.html) in a conversation about async/await. I've been using this cancellation model in [Nuke](https://github.com/kean/Nuke) for more than a year now, so I decided to share some of my experiences with it.

{% include ad-hor.html %}

## Cancellation Token

Nuke has to manage cancellation of lots of chained asynchronous operations. In the earlier versions cancellation was implemented using a few different ad-hoc mechanisms, including `Foundation.Operation` cancellation, some ad-hoc tasks responsible for cancelling multiple underlying operations, and more. It was a mess. In an effort to simplify cancellation I've looked at some ideas outside of the Swift world.

One of the most promising patterns that I found was C# [Cancellation Token](https://docs.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads). It is a unified model for cooperative cancellation of asynchronous operations:

> This model is based on a lightweight object called a cancellation token. The object that invokes one or more cancelable operations, for example by creating new threads or tasks, passes the token to each operation. Individual operations can in turn pass copies of the token to other operations. At some later time, the object that created the token can use it to request that the operations stop what they are doing. Only the requesting object can issue the cancellation request, and each listener is responsible for noticing the request and responding to it in an appropriate and timely manner.


## Implementation

The general pattern for implementing the cooperative cancellation model consists of two components:

- A cancellation token (`CancellationToken`) which provides each operation with a way to register for cancellation notifications. 
- A cancellation token source (`CancellationTokenSource`) which manages tokens and sends cancellation notifications to each of the cancellation tokens.

Here's an API for those two types as implemented in Nuke:

```swift
public struct CancellationToken {
    /// Returns `true` if cancellation has been requested for this token.
    public var isCancelling: Bool { get }

    /// Registers the closure that will be called when the token is canceled.
    /// If this token is already cancelled, the closure will be run immediately
    /// and synchronously.
    public func register(closure: @escaping () -> Swift.Void)
}

final public class CancellationTokenSource {
    /// Returns `true` if cancellation has been requested.
    public var isCancelling: Bool { get }

    /// Creates a new token associated with the source.
    public var token: CancellationToken { get }

    /// Communicates a request for cancellation to the managed tokens.
    public func cancel()
}
```


## Usage

To use this model you first create a token source at a point where you start one or more asynchronous operations. Then you create a token using the token source and pass the token to each of the operations, which in turn register for cancellation notifications using those tokens. Here's a code example from Nuke:

```swift
let cts = CancellationTokenSource()
Manager.shared.loadImage(with: url, token: cts.token) {
    print("result \($0)")
}
```

```swift
func loadData(with request: URLRequest, token: CancellationToken, completion: @escaping (Result<Data>) -> Void) {
    let task = session.dataTask(with: request)
    // <...>
    token.register { task.cancel() }
}
```

When `cancel()` is called on the `CancellationTokenSource` all the tokens are cancelled.


## Pros

There are quite a few advantages of using cancellation tokens.

- It works extremely well for cooperative cancellation of multiple tasks. For example, `Loader` class in Nuke simply passes a cancellation token from one operation to another.

- It makes cancellation an orthogonal concept which simplifies other parts of the system. You no longer have to bake cancellation into operations / tasks / promises. This is the primary reason why cancellation token has [surfaced in Swift Evolution](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170821/039226.html). Async/await (as well as many Promise implementations) doesn't offer a cancellation mechanism of its own. However, you can always use a cancellation token.

- Operations can register for cancellation after the token was created. This is very useful when you start executing the work asynchronously the way Nuke does:

```swift
public func loadImage(with request: Request, token: CancellationToken?, completion: @escaping (Result<Image>) -> Void) {
    queue.async {
        let task = Task() // create task asynchronously
        token.register { task.cancel() } // register asynchronously
    }
}
```

## Performance

The main con of cancellation token model is that it is a bit tricky to implement. It requires two types which coordinate with each other. And a token source some way to synchronize access to the underlying storage for observers, which might make it a bit heavy-weight and have a negative effect on performance. Fortunately, Nuke has solutiona for those performance issues.

Thanks to the way `CancellationTokenSource` is implemented in Nuke, it is able to use a single shared lock for all of the sources. This is possible for two reasons. First, `CancellationTokenSource` never executes any of the registered closures inside a lock, eliminating the possibility of deadlocks. And second, a critical sections executed inside the lock are extremely fast.

Thanks to that optimizations, `CancellationTokenSource` becomes super cheap to use.

## Alternatives

Cancellation tokens get the job done in Nuke, however, it does have some cons. It might be a bit cumbersome to use and might feel unfamiliar to Swift developers. My favorite cancellation model remains [disposing](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/GettingStarted.md#disposing) from RxSwift (not actually originated there). It's more convenient to use and offers a rich set of features. [RxNuke](https://github.com/kean/RxNuke) wraps Nuke's cancellation tokens in disposables. However, I find cancellation tokens to be a better fit for core Nuke project because of how lightweight the pattern is.


## Resources

- [Cancellation in Managed Threads](https://docs.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads)
- [Nuke: CancellationToken](https://github.com/kean/Nuke/blob/master/Sources/CancellationToken.swift) 
