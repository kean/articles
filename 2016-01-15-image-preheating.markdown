---
layout: post
title: "Preheating Images"
description: "Preheating is an effective way to improve user experience by downloading data ahead of time in anticipation of its use"
date: 2016-01-12 20:30:05 +0300
category: programming
tags: swift, ios, nuke
permalink: /post/image-preheating
redirect_from: /blog/image-preheating
uuid: 7ca6cd36-1cb2-4825-93af-3c8e291b9960
---

Preheating (prefetching/precaching) resources is an effective way to improve user experience for many apps. Prefetching is a [common term](https://en.wikipedia.org/wiki/Prefetching) that refers to *software that downloads data ahead of time in anticipation of its use*.

One of the resources that you might want to prefetch for your users is images. In this post I'm going to be talking about prefetching images in a `UICollectionView`. More specifically, dynamically prefetching images when the user is scrolling the content.

{% include ad-hor.html %}

**Warning: this post is a bit outdated because of the [Nuke](https://kean.github.io/Nuke/) and [Preheat](https://github.com/kean/Preheat) changes.** 

> ## TL;DR
- Preheating (prefetching/precaching) means loading data ahead of time in anticipation of its use
- It's possible to automate preheating in a `UICollectionView` and `UITableView`
- The actual implementation detects when the user scrolls the content, calculates the preheat window, finds out which cells are going to be displayed in it, and calls a handler that starts/stops image requests
 - [Preheat](https://github.com/kean/Preheat) & [Nuke](https://kean.github.io/Nuke/) provide a set of tools for preheating images

## Preheating in a UICollectionView

It's relatively simple to automate preheating in a collection view.

Here's an outline of what we need to do:

- Detect when the user is scrolling
- Calculate an area (*preheat window*) just outside of the viewport in the direction in which the user is scrolling
- Find out which cells are going to be displayed in it
- Signal the delegate with index paths for these cells

### Calculating Preheat Window

I'm not posting the entire implementation here, just some more interesting bits.

A `UICollectionView` is a `UIScrollView` subclass which means that we can detect when the user scrolls the content by observing changes to `contentOffset` property using KVO.

The next step is to calculate a preheat window in the current scroll direction (works for a `UICollectionViewFlowLayout` only):

```swift
extension UICollectionView {
    // `sizeRatio` is a proportion of a scroll view's width (or height for views with vertical orientation) used as a preheating window width (or height).
    func preheatingRect(isScrollingForward isScrollingForward: Bool, sizeRatio: CGFloat) -> CGRect {
        let viewport = CGRect(origin: contentOffset, size: bounds.size)
        switch (collectionViewLayout as! UICollectionViewFlowLayout).scrollDirection {
        case .Vertical:
            let height = CGRectGetHeight(viewport) * sizeRatio
            let y = isScrollingForward ? CGRectGetMaxY(viewport) : CGRectGetMinY(viewport) - height
            return CGRectIntegral(CGRect(x: 0, y: y, width: CGRectGetWidth(viewport), height: height))
        case .Horizontal:
            let width = CGRectGetWidth(viewport) * sizeRatio
            let x = isScrollingForward ? CGRectGetMaxX(viewport) : CGRectGetMinX(viewport) - width
            return CGRectIntegral(CGRect(x: x, y: 0, width: width, height: CGRectGetHeight(viewport)))
        }
    }
}
```

After we calculated a preheat window we find out which cells are going to be displayed in it by using `layoutAttributesForElementsInRect(_)` method of `UICollectionView` class. It also makes sense to sort index paths by the distance to the viewport.

### Updating Preheat Window

It becomes a little more complicated if you want to update preheat window *dynamically* when the user scrolls the content.

First, we need to keep track of the current preheat window, and update it only when the user scrolls far enough from the point where it was calculated. Otherwise, we would be recalculating it each time we observe a change in a `contentOffset` property.

Second, we might make it more convenient for the delegate by calculating a diff between previous preheat window and the current one (in terms of index paths):

```swift
// you might want to use `Set` instead
let added = newIndexPaths.filter { !self.indexPaths.contains($0) }
let removed = self.indexPaths.filter { !newIndexPaths.contains($0) }
self.indexPaths = newIndexPaths
handler?(added: added, removed: removed)
```

## Preheat & Nuke

The full implementation of preheating is available in [Preheat](https://github.com/kean/Preheat) library. Preheat consists of a single generic `PreheatController` class that supports both `UICollectionView` and `UITableView`.

You can use Preheat with any image loading library, including [Nuke](https://github.com/kean/Nuke) which it way designed for. Nuke provides a set of self-explanatory methods for precaching image which are inspired by the [PHImageManager](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHImageManager_Class/index.html) class:

```swift
func startPreheatingImages(requests: [ImageRequest])
func stopPreheatingImages(requests: [ImageRequest])
func stopPreheatingImages()
```

When you call `startPreheatingImages(_)` method, Nuke starts to load and cache images for the given requests. Nuke caches images with the exact *target size*, *content mode*, and *filters* provided in the request. At any time afterward, you can create tasks with equivalent requests - for each of the requests Nuke would either return a cached image, or add another observer to the existing preheating task.

Nuke guarantees that preheating tasks never interfere with normal (non-preheating) tasks. For instance, preheating tasks don't start executing until there are no outstanding non-preheating tasks. There is also a limit of concurrent preheating tasks.

Here is an example of how you might implement preheating in your application using Preheat & Nuke:

```swift
class PreheatDemoViewController: UICollectionViewController, PreheatControllerDelegate {
    var preheatController: PreheatController<UICollectionView>!

    override func viewDidLoad() {
        super.viewDidLoad()

        preheatController = PreheatController(view: collectionView!)
        preheatController.handler = { [weak self] in
            self?.preheatWindowChanged(addedIndexPaths: $0, removedIndexPaths: $1)
        }
    }

    func preheatWindowChanged(addedIndexPaths added: [NSIndexPath], removedIndexPaths removed: [NSIndexPath]) {
        func requestsForIndexPaths(indexPaths: [NSIndexPath]) -> [ImageRequest] {
            return indexPaths.map { ImageRequest(photos[$0.row].URL) }
        }
        Nuke.startPreheatingImages(requestsForIndexPaths(added))
        Nuke.stopPreheatingImages(requestsForIndexPaths(removed))
    }

    override func viewDidAppear(animated: Bool) {
        super.viewDidAppear(animated)

        preheatController.enabled = true
    }

    override func viewDidDisappear(animated: Bool) {
        super.viewDidDisappear(animated)

        // When you disable preheat controller it removes all preheating
        // index paths and calls its handler
        preheatController.enabled = false
    }
}
```

## Credits

The idea of automating preheating was inspired by Apple's Photos framework [sample](https://developer.apple.com/library/ios/samplecode/UsingPhotosFramework/Introduction/Intro.html).

## References

1. [Apple's Photos framework sample](https://developer.apple.com/library/ios/samplecode/UsingPhotosFramework/Introduction/Intro.html)
2. [Nuke](https://kean.github.io/Nuke/)
3. [Preheat](https://github.com/kean/Preheat)
4. [PHImageManager](https://developer.apple.com/library/prerelease/ios/documentation/Photos/Reference/PHImageManager_Class/index.html)
