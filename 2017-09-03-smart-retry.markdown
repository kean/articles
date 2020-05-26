---
layout: post
title: "Smart Retry"
description: "Combining retryWhen operator with Reachability and delay options inspired by RxSwiftExt to implement auto-retries"
date: 2017-09-03 10:00:00 +0300
category: programming
tags: ios
permalink: /post/smart-retry
uuid: 134b8103-bc6b-4e34-bf8f-4c1a551a4473
---

In this post I'm going to combine [`retryWhen`](http://reactivex.io/documentation/operators/retry.html) operator, [Reachability](https://developer.apple.com/library/content/samplecode/Reachability/Introduction/Intro.html), and delay options inspired by [RxSwiftExt](https://github.com/RxSwiftCommunity/RxSwiftExt) to implement an effective retry strategy.

The requirements are fairly straightforward. Let's say you have an observable sequence which wraps a networking call. If the sequence fails with a network error what I would like to do is:

- Automatically retry up to N times
- Use exponential backoff or other delay options
- Retry immediately when a network connection is re-established

{% include ad-hor.html %}

This sounds like a tall order for a single `retryWhen` operator, but it's actually flexible enough to support all of those requirements. In this post, I'm going to create a new custom `retry` operator which would wrap this entire logic.

> The complete implementation is available [here](https://gist.github.com/kean/e2bc38106d19c249c04162714e7be321).

> Check out <a href="{{ site.url }}/post/api-client">**API Client in Swift**</a> and <a href="{{ site.url }}/post/introducing-rxnuke">**Introducing RxNuke**</a> for other RxSwift use-cases.

## Usage

Here's how our custom `retry` operator is defined:

```swift
extension ObservableType {
    /// Retries the source observable sequence on error using a provided retry
    /// strategy.
    /// - parameter maxAttemptCount: Maximum number of times to repeat the
    /// sequence. `Int.max` by default.
    /// - parameter delay: Algorithm for computing retry interval.
    /// - parameter didBecomeReachable: Trigger which is fired when network
    /// connection becomes reachable.
    /// - parameter shouldRetry: Always returns `true` by default.
    func retry(_ maxAttemptCount: Int = default,
               delay: DelayOptions,
               didBecomeReachable: Observable<Void> = default,
               shouldRetry: @escaping (Error) -> Bool = default) -> Observable<E>
}
```

> I've also added a couple of extension to some of the RxSwift [traits](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Traits.md) to make using `retry` easier.

You would use the `retry` operator just as any other of the built-in operators. For example, here's how it fits into a classic search example:

```swift
let isBusy = ActivityIndicator()

let results = input
    .throttle(0.3)
    .distinctUntilChanged()
    .flatMapLatest { input in
        guard input.characters.count > 1 else { return .just([]) }
        return service(input)
            .retry(delay: .exponential(initial: 3, multiplier: 1.5, maxDelay: 16))
            .trackActivity(isBusy)
            .asDriver(onErrorJustReturn: [])
```

> Notice that `trackActivity(isBusy)` is called _after_ the `retry` operator. If it were called before it then it would be tracking the activity of each of the individual requests.

## Implementation

Let's jump straight into the final implementation and then work our way down:

> The complete implementation is available [here](https://gist.github.com/kean/95b69ef1a90bb62e9b81e924a0a71437)

```swift
extension ObservableType {
    /// Retries the source observable sequence on error using a provided retry
    /// strategy.
    /// - parameter maxAttemptCount: Maximum number of times to repeat the
    /// sequence. `Int.max` by default.
    /// - parameter didBecomeReachable: Trigger which is fired when network
    /// connection becomes reachable.
    /// - parameter shouldRetry: Always returns `true` by default.
    func retry(_ maxAttemptCount: Int = Int.max,
               delay: DelayOptions,
               didBecomeReachable: Observable<Void> = Reachability.shared.didBecomeReachable,
               shouldRetry: @escaping (Error) -> Bool = { _ in true }) -> Observable<E> {
        return retryWhen { (errors: Observable<Error>) in
            return errors.flatMapWithIndex { error, attempt -> Observable<Void> in
                guard shouldRetry(error), maxAttemptCount > attempt + 1 else {
                    return .error(error)
                }

                let timer = Observable<Int>.timer(
                    RxTimeInterval(delay.make(attempt + 1)),
                    scheduler: MainScheduler.instance
                ).map { _ in () } // cast to Observable<Void>

                return Observable.merge(timer, didBecomeReachable)
            }
        }
    }
}
```

That's a lot to chew on. Let's approach it piece by piece.

### retryWhen Operator

The [`retryWhen`](http://reactivex.io/documentation/operators/retry.html) operator is the most important and the most complex part. It's very powerful, but it takes some time to grasp.

Here's how it is defined in `RxSwift`:

```swift
extension ObservableType {
    /**
     Repeats the source observable sequence on error when the notifier emits
     a next value. If the source observable errors and the notifier completes,
     it will complete the source sequence.

     - seealso: [retry operator on reactivex.io](http://reactivex.io/documentation/operators/retry.html)

     - parameter notificationHandler: A handler that is passed an observable
     sequence of errors raised by the source observable and returns and
     observable that either continues, completes or errors. This behavior is
     then applied to the source observable.
     - returns: An observable sequence producing the elements of the given
     sequence repeatedly until it terminates successfully or is notified to
     error or complete.
     */
    public func retryWhen<TriggerObservable: ObservableType, Error: Swift.Error>(
        _ notificationHandler: @escaping (Observable<Error>) -> TriggerObservable)
        -> Observable<E>
```

<img alt="RxSwift retryWhen marble diagram" class="AdaptiveImage" src="{{ site.url }}/images/misc/retryWhen.f.png" style="max-width:500px;">

The documentation describes it really nicely. The basic idea is the `retryWhen` gives you an observable sequence of errors over which you can then for example `flatMap` over to handle each of the individual errors.

A couple of important points to keep in mind are:

- You should _not_ ignore errors observable sequence in your `notificationHandler`. If you do and for example just return a timer from the handler, then the timer will simply run in parallel with the source sequence! This would probably not want you intended.
- In RxSwift 3.x if the trigger completes the source sequence also completes. This might not be what you expect. This behavior might change in [RxSwift 4](https://github.com/ReactiveX/RxSwift/issues/1082). I would suggest not to rely on it in your code.

### Delay Options

The `DelayOptions` is just a simple enum which defines a number of strategies for calculating delay interval for each attempt:

```swift
enum DelayOptions {
    case immediate()
    case constant(time: Double)
    case exponential(initial: Double, multiplier: Double, maxDelay: Double)
    case custom(closure: (Int) -> Double)
}

extension DelayOptions {
    func make(_ attempt: Int) -> Double {
        switch self {
        case .immediate: return 0.0
        case .constant(let time): return time
        case .exponential(let initial, let multiplier, let maxDelay):
            // if it's first attempt, simply use initial delay, otherwise calculate delay
            let delay = attempt == 1 ? initial : initial * pow(multiplier, Double(attempt - 1))
            return min(maxDelay, delay)
        case .custom(let closure): return closure(attempt)
        }
    }
}
```

### Reachability

[Reachability](https://developer.apple.com/library/content/samplecode/Reachability/Introduction/Intro.html) monitors network state. There are a couple of open source libraries that provide a convenient API on top of it. For example, I personally use `NetworkReachabilityManager` which is part of [Alamofire](https://github.com/Alamofire/Alamofire#network-reachability) since I already [use this library in my projects](https://kean.github.io/post/api-client).

```swift
final class Reachability {
    static let shared = Reachability()

    private let reachability = NetworkReachabilityManager()

    var didBecomeReachable: Observable<Void> { return _didBecomeReachable.asObservable() }
    private let _didBecomeReachable = PublishSubject<Void>()

    init() {
        if let reachability = self.reachability {
            reachability.listener = { [weak self] in
                self?.update($0)
            }
            reachability.startListening()
        }
    }

    private func update(_ status: NetworkReachabilityManager.NetworkReachabilityStatus) {
        if case .reachable = status {
            _didBecomeReachable.onNext(())
        }
    }
}
```

> The complete implementation is available [here](https://gist.github.com/kean/e2bc38106d19c249c04162714e7be321).

## Resoures

- [RxSwift](https://github.com/ReactiveX/RxSwift)
- [ReactiveX operators: retry](http://reactivex.io/documentation/operators/retry.html)
- [Reachability](https://developer.apple.com/library/content/samplecode/Reachability/Introduction/Intro.html)
