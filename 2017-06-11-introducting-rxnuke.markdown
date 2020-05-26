---
layout: post
title: "Introducing RxNuke"
description: "I'm excited to introduce a new addition to Nuke - RxNuke which brings the power of RxSwift to your image loading pipelines"
date: 2017-06-11 21:00:00 +0300
category: programming
tags: ios
permalink: /post/introducing-rxnuke
uuid: 687bb115-e04c-4c6f-9990-786793ba8685
---

I'm excited to introduce a new addition to Nuke - [RxNuke](https://github.com/kean/RxNuke) which brings the power of [RxSwift](https://github.com/ReactiveX/RxSwift) to your image loading pipelines.

<img alt="RxNuke logo" src="{{ site.url }}/images/posts/introducing_rxnuke_01.png" class="Screenshot">

{% include ad-hor.html %}

Nuke's design has always prioritized simplicity, reliability, and performance. The core framework has a small API surface and only contains a minimum number of features to built upon. This is great for many reasons, but unfortunately, that left a gap between what Nuke supports out of the box and what users need. I received many requests about particular use cases like these:

> - "show stale response while validating the image"
> - "wait until N images are loaded and only then proceed"
> - "show a low-res image while loading a high-res one".

Each app has a different idea about how to configure their image loading pipeline. It's not feasible to support all of them in a single framework. [RxNuke](https://github.com/kean/RxNuke) aims at bridging this gap by leveraging the power of reactive programming to serve all of the aforementioned use cases, as well as many others.

> Check out <a href="{{ site.url }}/post/api-client">**API Client in Swift**</a> for more awesome use-cases of RxSwift. You could also find <a href="{{ site.url }}/post/smart-retry">**Smart Rerty**</a> useful. It can automatically retry image requests for you.

# Introduction

In order to get started with RxNuke you should be familiar with the basics of RxSwift. Even if you don't you can already start taking advantage of RxNuke powerful features thanks to a number of [examples of common use cases](https://github.com/kean/RxNuke#use-cases) available in a RxNuke documentation.

Let's starts with the basics. The initial version of RxNuke adds a single new `Loading` protocol with a set of methods which returns `Singles`.

```swift
public protocol Loading {
    func loadImage(with url: URL) -> Single<Image>
    func loadImage(with urlRequest: URLRequest) -> Single<Image>
    func loadImage(with request: Nuke.Request) -> Single<Image>
}
```

> A `Single` is a variation of `Observable` that, instead of emitting a series of elements, is always guaranteed to emit either a single element or an error. The common use case of `Single` is to wrap HTTP requests. See [Traits](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Traits.md#single) for more info.

One of the Nuke's types that implement new `Loading` protocol is `Nuke.Manager`. Here's an example of how to start a request using one of the new APIs and then display an image if the request is finished successfully:

```swift
Nuke.Manager.shared.loadImage(with: url)
    .observeOn(MainScheduler.instance)
    .subscribe(onSuccess: { imageView.image = $0 })
    .disposed(by: disposeBag)
```

The first thing that `Nuke.Manager` does when you subscribe to an observable is check if the image is stored in its memory cache. If it is the manager synchronously successfully finishes the request. If the image is not cached, the manager asynchronously loads an image using an underlying image loader (see `Nuke.Loading` protocol).

This looks simple enough. Now let's see what makes this new addition to Nuke so powerful.

# Use Cases

I'm going to go throught a number of real world uses case and see how they can be implemented using `RxNuke`:

- [Going From Low to High Resolution](#huc_low_to_high) 
- [Loading the First Available Image](#huc_loading_first_avail)
- [Load Multiple Images, Display All at Once](#huc_load_multiple_display_once)
- [Showing Stale Image While Validating It](#huc_showing_stale_first)
- [Auto Retry](#huc_auto_retry)
- [Tracking Activities](#huc_activity_indicator)
- [Table or Collection View](#huc_table_collection_view)

### <a name="huc_low_to_high"></a>Going From Low to High Resolution

Suppose you want to show users a high-resolution, slow-to-download image. Rather than let them stare a placeholder for a while, you might want to quickly download a smaller thumbnail first. 

You can implement this using [`concat`](http://reactivex.io/documentation/operators/concat.html) operator which results in a **serial** execution. It would first start a thumbnail request, wait until it finishes, and only then start a request for a high-resolution image.

```swift
Observable.concat(loader.loadImage(with: lowResUrl).orEmpty,
                  loader.loadImage(with: highResUtl).orEmpty)
    .observeOn(MainScheduler.instance)
    .subscribe(onNext: { imageView.image = $0 })
    .disposed(by: disposeBag)
```

> `orEmpty` is a custom operator which dismisses errors and completes the sequence instead
> (equivalent to `func catchErrorJustComplete()` from [RxSwiftExt](https://github.com/RxSwiftCommunity/RxSwiftExt)


### <a name="huc_loading_first_avail"></a>Loading the First Available Image

Suppose you have multiple URLs for the same image. For instance, you might have uploaded an image taken from the camera. In such case, it would be beneficial to first try to get the local URL, and if that fails, try to get the network URL. It would be a shame to download the image that we may have already locally.

This use case is very similar [Going From Low to High Resolution](#huc_low_to_high), but an addition of `.take(1)` guarantees that we stop execution as soon as we receive the first result.

```swift
Observable.concat(loader.loadImage(with: localUrl).orEmpty,
                  loader.loadImage(with: networkUrl).orEmpty)
    .take(1)
    .observeOn(MainScheduler.instance)
    .subscribe(onNext: { imageView.image = $0 })
    .disposed(by: disposeBag)
```


### <a name="huc_load_multiple_display_once"></a>Load Multiple Images, Display All at Once

Suppose you want to load two icons for a button, one icon for `.normal` state and one for `.selected` state. Only when both icons are loaded you can show the button to the user. This can be done using a [`combineLatest`](http://reactivex.io/documentation/operators/combinelatest.html) operator:

```swift
Observable.combineLatest(loader.loadImage(with: iconUrl).asObservable(),
                         loader.loadImage(with: iconSelectedUrl).asObservable())
    .observeOn(MainScheduler.instance)
    .subscribe(onNext: { icon, iconSelected in
        button.isHidden = false
        button.setImage(icon, for: .normal)
        button.setImage(iconSelected, for: .selected)
    }).disposed(by: disposeBag)
```


### <a name="huc_showing_stale_first"></a>Showing Stale Image While Validating It

Suppose you want to show users a stale image stored in a disk cache (`Foundation.URLCache`) while you go to the server to validate it. This use case is actually the same as [Going From Low to High Resolution](#huc_low_to_high).

```swift
let cacheRequest = URLRequest(url: imageUrl, cachePolicy: .returnCacheDataDontLoad)
let networkRequest = URLRequest(url: imageUrl, cachePolicy: .useProtocolCachePolicy)

Observable.concat(loader.loadImage(with: cacheRequest).orEmpty,
                  loader.loadImage(with: networkRequest).orEmpty)
    .observeOn(MainScheduler.instance)
    .subscribe(onNext: { imageView.image = $0 })
    .disposed(by: disposeBag)
```

> See [**Image Caching**](https://kean.github.io/post/image-caching) to learn more about HTTP cache


### <a name="huc_auto_retry"></a>Auto Retry

Auto-retry up to 3 times with an exponentially increasing delay using a retry operator provided by [RxSwiftExt](https://github.com/RxSwiftCommunity/RxSwiftExt).

```swift
loader.loadImage(with: request).asObservable()
    .retry(.exponentialDelayed(maxCount: 3, initial: 3.0, multiplier: 1.0))
    .observeOn(MainScheduler.instance)
    .subscribe(onNext: { imageView.image = $0 })
    .disposed(by: disposeBag)
 ```

> See [A Smarter Retry with RxSwiftExt](http://rx-marin.com/post/rxswift-retry-with-delay/) for more info about auto retries


### <a name="huc_activity_indicator"></a>Tracking Activities

Suppose you want to show an activity indicator while waiting for an image to load. Here's how you can do it using `ActivityIndicator` class provided by [`RxSwiftUtilities`](https://github.com/RxSwiftCommunity/RxSwiftUtilities):

```swift
let isBusy = ActivityIndicator()

loader.loadImage(with: imageUrl)
    .observeOn(MainScheduler.instance)
    .trackActivity(isBusy)
    .subscribe(onNext: { imageView.image = $0 })
    .disposed(by: disposeBag)

isBusy.asDriver()
    .drive(activityIndicator.rx.isAnimating)
    .disposed(by: disposeBag)
```


### <a name="huc_table_collection_view"></a>Table or Collection View

Here's how you can integrate the code provided in the previous examples into your table or collection view cells:

```swift
final class ImageCell: UICollectionViewCell {

    private var imageView: UIImageView!
    private var disposeBag = DisposeBag()

    // <.. create an image view using your preferred way ..>

    func display(_ image: Single<Image>) {

        // Create a new dispose bag, previous dispose bag gets deallocated
        // and cancels all previous subscriptions.
        disposeBag = DisposeBag()

        imageView.image = nil

        // Load an image and display the result on success.
        image.subscribeOn(MainScheduler.instance)
            .subscribe(onSuccess: { [weak self] image in
                self?.imageView.image = image
            }).disposed(by: disposeBag)
    }
}
```


# Conclusion

I hope that `RxNuke` becomes a valuable addition to Nuke. It brings power to solve many common use cases which are hard to implement without Rx. `RxNuke` is still very early stage. As it evolves it's going to bring some new powerful features made possible by Rx, more examples of common use cases, and more `Nuke` extensions to give you the power to build the exact image loading pipelines that you want.

> If you have any questions, additions or corrections to the examples from the article please feel free to leave a comment below, or hit me up on [Twitter](https://twitter.com/a_grebenyuk).

# Links

- [RxSwift](https://github.com/ReactiveX/RxSwift)
- [RxNuke](https://github.com/kean/RxNuke)
