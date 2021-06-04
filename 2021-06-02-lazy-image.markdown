---
layout: post
title: "LazyImage"
subtitle: Lazy image loading for SwiftUI
description: Lazy image loading for SwiftUI
date: 2021-06-02 09:00:00 -0500
category: programming
tags: programming
permalink: /post/lazy-image
uuid: 608e88a5-d9e7-40af-8957-18ad1b5bb025
---

[Nuke 10](https://github.com/kean/Nuke/releases/tag/10.0.0) is out, but this post is not about it. It's about a new package called [NukeUI](https://github.com/kean/NukeUI) that makes lazy image loading as easy as possible. It comes with two main components:

- `LazyImage` for SwiftUI
- `LazyImageView` for UIKit/AppKit

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">struct</span> <span class="kt">ContainerView</span><span class="p">:</span> <span class="kt">View</span> <span class="p">{</span>
    <span class="k">var</span> <span class="nv">body</span><span class="p">:</span> <span class="k">some</span> <span class="kt">View</span> <span class="p">{</span>
        <span class="kt">LazyImage</span><span class="p">(</span><span class="nv">source</span><span class="p">:</span> <span class="s">"https://example.com/image.jpeg"</span><span class="p">)</span>
            <span class="o">.</span><span class="kt">placeholder</span> <span class="p">{</span> <span class="kt">Image</span><span class="p">(</span><span class="s">"placeholder"</span><span class="p">)</span> <span class="p">}</span>
            <span class="o">.</span><span class="kt">transition</span><span class="p">(</span><span class="o">.</span><span class="kt">fadeIn</span><span class="p">(</span><span class="nv">duration</span><span class="p">:</span> <span class="mf">0.33</span><span class="p">))</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

`LazyImage` uses [Nuke](https://github.com/kean/Nuke) for loading images. It has many customization options. It also has GIF support powered by [Gifu](https://github.com/kaishin/Gifu). But GIF is [not the most](https://web.dev/replace-gifs-with-videos/) efficient format for animated images, so NukeUI can also play short videos, which I’ll talk about later. And it supports progressive images.

You can learn more about `LazyImage` in the [repository](https://github.com/kean/NukeUI). And in this post, I'll quickly go over some of the design decisions.

## Design

My initial approach to SwiftUI in Nuke was to provide a view model (`ObservableObject`) that you could integrate into your views – [`FetchImage`](/post/introducing-fetch-image). The reasoning was that creating views in SwiftUI is easy: you can start with a simple example and customize it precisely the way you want. It still stands, and, with [Nuke 10](https://github.com/kean/Nuke/releases/tag/10.0.0), FetchImage is part of the main library. But this approach is good only up to a point.

Firstly, there are not that many customizations you might want. It’s a good idea to provide a solution that would work for most people. Secondly, in SwiftUI, there are not that many tools available yet. For example, if you want to use [Gifu](https://github.com/kaishin/Gifu) to render animated GIFs, you need to use UIKit. Putting this all together can be a daunting task. And this is the problem I wanted to tackle with NukeUI.

### Lazy

I wanted a great name for this component that would feel at home in SwiftUI. `Image` was taken by SwiftUI. `Nuke.Image` felt clunky. I tried a couple more options befre settling on `LazyImage`.

It's lazy because it loads the image from the source only when it appears on the screen. And when it disappears (or is deallocated), the current request automatically gets canceled. When the view reappears, the download picks up where it left off, thanks to [resumable downloads](https://kean.blog/post/resumable-downloads). 

`LazyImage` also doesn't know the size of the image before it downloads it. If you look at the native `Image`, it's not resizable by default, and you have to [make it so](https://developer.apple.com/documentation/swiftui/image/resizable(capinsets:resizingmode:)). That's a good default when you display images eagerly, but not for the lazy ones. Thus, with `LazyImage`, you must specify the view size before loading the image. By default, it will resize the image to fill the available space and preserve the aspect ratio. You can change this behavior by passing a different content mode.

And if you are coming to SwiftUI from the web, the terms used by `NukeUI` will be instantly familiar: lazy image loading, image `source`.

## Implementation

I initially started working on a pure-SwiftUI version of `LazyImage`, but quickly realized there were practically no advantages of doing that, but there were limitations. I also still need to use UIKit at least partially if I wanted to use [Gifu](https://github.com/kaishin/Gifu) for animated image rendering.

So instead of the initially planned one component, I made two: `LazyImageView` for UIKit and AppKit and `LazyImage`, a thin wrapper on top of it for SwiftUI. They both have equivalent APIs, which makes it easy to learn them. And I didn't need to re-implement any of the functionality, which is great. My initial SwiftUI implementation was also using [@StateObject](https://developer.apple.com/documentation/swiftui/stateobject), making it iOS 14+ only. But with the new approach, I dropped the deployment target to iOS 13.

I also added an escape hatch similar to [SwiftUI Introspect](https://github.com/siteline/SwiftUI-Introspect). So if some features are not yet exposed in `LazyImage` or SwiftUI, there is a "legal" way to access the underlying UIKit or AppKit components.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kt">LazyImage</span><span class="p">(</span><span class="nv">source</span><span class="p">:</span> <span class="s">"https://example.com/image.jpeg"</span><span class="p">)</span>
    <span class="o">.</span><span class="kt">onCreated</span> <span class="p">{</span> <span class="n">view</span> <span class="k">in</span> 
        <span class="n">view</span><span class="o">.</span><span class="kt">isExperimentalVideoSupportEnabled</span> <span class="o">=</span> <span class="kc">true</span>
        <span class="n">view</span><span class="o">.</span><span class="kt">videoGravity</span> <span class="o">=</span> <span class="o">.</span><span class="kt">resizeAspect</span>
    <span class="p">}</span>
</code></pre></div></div>

`LazyImageView` is also just straight up a nicer and more powerful API than the existing categories for `UIImageView` that most frameworks, including Nuke, provide.

## Animation (and Video)

Some frameworks include the support for rendering animated images right in the main target. For example, there is a framework that has a copied and modified version of [Gifu](https://github.com/kaishin/Gifu) in it. That wasn't the path I wanted to take. Nuke plays well with others, and it instead makes [integration with Gifu](https://kean.blog/nuke/guides/image-formats#gif) as easy as possible. And with NukeUI, it's even easier.

Rendering animated images is not a concern of a framework that loads images – it’s the responsibility of the UI.  And that's why it fits perfectly in NukeUI where [Gifu](https://github.com/kaishin/Gifu) is added as a dependency by default, so you don't need to do anything. Gifu is a relatively small Swift library, and if you install it using SPM, it ends up in the same binary anyway, so there is barely any overhead. It also keeps [Nuke](https://github.com/kean/Nuke) lighter.

So I managed to add GIF support without reinventing it and added it to `LazyImageView`. And, of course, `LazyImage` also got it without me having to do anything. But there is a problem with GIF – it's inefficient. A common approach is to [replace GIFs with short videos](https://web.dev/replace-gifs-with-videos/) with no sound. An MP4 video file can be an order of magnitude lighter than an equivalent GIF and can also take advantage of hardware-accelerated rendering. 

> For example, check out the ["AppKit is Done"](/post/appkit-is-done) post. It's all MP4 videos that are 2 MB on average. It would've not been feasible to get this quality with GIF.
{:.info}

So I thought `LazyImage` was a great opportunity to accelerate the switch to video. With `LazyImage`, displaying a short video is as easy as providing a video URL as a source.

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/lazy-image.mp4" type="video/mp4">
</video>
</div>

It required quite a bit of digging into AVKit because there was no high-level API to play videos from memory. I ended up an asset (`AVAsset`) with a custom `AVAssetResourceLoaderDelegate`:

```swift
// This allows LazyImage to play video from memory.
final class DataAssetResourceLoader: NSObject, AVAssetResourceLoaderDelegate {
    private let data: Data
    private let contentType: String

    init(data: Data, contentType: String) {
        self.data = data
        self.contentType = contentType
    }

    func resourceLoader(_ resourceLoader: AVAssetResourceLoader, shouldWaitForLoadingOfRequestedResource loadingRequest: AVAssetResourceLoadingRequest) -> Bool {
        if let contentRequest = loadingRequest.contentInformationRequest {
            contentRequest.contentType = contentType
            contentRequest.contentLength = Int64(data.count)
            contentRequest.isByteRangeAccessSupported = true
        }

        if let dataRequest = loadingRequest.dataRequest {
            if dataRequest.requestsAllDataToEndOfResource {
                dataRequest.respond(with: data[dataRequest.requestedOffset...])
            } else {
                let range = dataRequest.requestedOffset..<(dataRequest.requestedOffset + Int64(dataRequest.requestedLength))
                dataRequest.respond(with: data[range])
            }
        }

        loadingRequest.finishLoading()

        return true
    }
}
```

Both `LazyImage` and `LazyImageView` support video playback out of the box. In practice, it means that you can replace a 20 MB GIF with a 1 MB video of comparable quality. And instead of 60% CPU usage and overheating device, you'll see 0%.

There is nothing you need to do to enable video playback. It does the right thing by default:

- It plays automatically
- It doesn't show any controls
- It loops continuously
- It's always silent
- It doesn't prevent the display from sleeping
- It displays a preview until the video is downloaded

## Pre-Release

I hope I got you interested in `LazyImage`. You can learn more in the [repository](https://github.com/kean/NukeUI).`LazyImage` 0.1.0 is out (pre-release version) and you can give it a try now. You can see it in action in the [Nuke Demo](https://github.com/kean/NukeDemo).

