---
layout: post
title: "Introducing FetchImage"
subtitle: FetchImage makes it easy to download images using Nuke and display them in SwiftUI apps
description: Introducing FetchImage, a SwiftUI extension for Nuke, and image loading and caching framework for iOS
date: 2020-03-18 10:00:00 -0500
category: programming
tags: programming
permalink: /post/introducing-fetch-image
uuid: fde52716-cfda-4437-bc3f-92327d46e295
---

[`FetchImage`](https://github.com/kean/FetchImage) makes it easy to download images using [`Nuke`](https://github.com/kean/Nuke) and display them in SwiftUI apps.

## FetchImage

`FetchImage` is an observable object (`ObservableObject`) that allows you to manage the download of a single image and observe the results of the download. All of the changes to the download state are published using properties marked with `@Published` property wrapper.


```swift
public final class FetchImage: ObservableObject, Identifiable {
    /// Returns the fetched image.
    ///
    /// - note: In case pipeline has `isProgressiveDecodingEnabled` option enabled
    /// and the image being downloaded supports progressive decoding, the `image`
    /// might be updated multiple times during the download.
    @Published public private(set) var image: PlatformImage?

    /// Returns an error if the previous attempt to fetch the image failed with an error.
    /// Error is cleared out when the download is restarted.
    @Published public private(set) var error: Error?

    /// Returns `true` if the image is currently being loaded.
    @Published public private(set) var isLoading: Bool = false

    public struct Progress {
        /// The number of bytes that the task has received.
        public let completed: Int64

        /// A best-guess upper bound on the number of bytes the client expects to send.
        public let total: Int64
    }

    /// The progress of the image download.
    @Published public var progress = Progress(completed: 0, total: 0)
}
```

You can initialize the download either with a `URL` or with an `ImageRequest`, just as you would expect with `Nuke`. `FetchImage` supports everything that `Nuke` does. This includes changing request priorities, progressive image decoding, and more.

```swift
public final class FetchImage: ObservableObject, Identifiable {
    /// Initializes the fetch request and immediately start loading.
    public init(request: ImageRequest, pipeline: ImagePipeline = .shared)

    /// Initializes the fetch request and immediately start loading.
    public convenience init(url: URL, pipeline: ImagePipeline = .shared)
}
```

When the `FetchImage` object is created, it automatically starts the request. You also have an option to `cancel()` the request and restart it later using `fetch()` method. This is something that you would typically need when displaying images in a `List`. You can also use `fetch()` method to restart failed downloads.

Another little thing that `FetchImage` does for you is automatically cancelling the download when de-instantiated.

```swift
public final class FetchImage: ObservableObject, Identifiable {

    /// Updates the priority of the task, even if the task is already running.
    public var priority: ImageRequest.Priority

	/// Starts loading an image unless the download is already completed successfully.
    public func fetch()

    /// Marks the request as being cancelled.
    public func cancel()
}
```

### Low Data Mode

iOS 13 introduced a new [Low Data mode](https://support.apple.com/en-us/HT210596) and `FetchImage` offers a built-in support for it.

```swift
FetchImage(regularUrl: highQualityUrl, lowDataUrl: lowQualityUrl)
```

`FetchedImage.init(regularUrl:lowDataUrl:pipeline:)` is a convenience initializer that fetches the image with a regular URL with constrained network access disabled, and if the download fails because of the constrained network access, uses a low data URL instead. It also handles the scenarios like fetching a high quality image when unconstrained network access is restored.

## Usage

Here is an example of using `FetchImage` in a custom SwiftUI view.

```swift
public struct ImageView: View {
    @ObservedObject var image: FetchImage

    public var body: some View {
        ZStack {
            Rectangle().fill(Color.gray)
            image.view?
                .resizable()
                .aspectRatio(contentMode: .fill)
        }

        // (Optional) Animate image appearance
        .animation(.default)

        // (Optional) Cancel and restart requests during scrolling
        .onAppear(perform: image.fetch)
        .onDisappear(perform: image.cancel)
    }
}

struct ImageView_Previews: PreviewProvider {
    static var previews: some View {
    	let url = URL(string: "https://cloud.githubusercontent.com/assets/1567433/9781817/ecb16e82-57a0-11e5-9b43-6b4f52659997.jpg")!
        ImageView(image: FetchImage(url: url))
            .frame(width: 80, height: 80)
            .clipped()
    }
}
```

`FetchImage` gives you full control over how to manage the download and how to display the image. For example, one thing that you can do is to replace `onAppear` and `onDisappear` hooks to lower the priority of the requests instead of cancelling them. This might be useful if you want to continue loading and caching the images even if the user leaves the screen, but you still want the images the are currently on screen to be downloaded first.

> Nuke supports resumable HTTP downloads, so as long as your server supports [HTTP range-requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests), you won't be losing any download progress when cancelling and restarting the requests. Resumable downloads are enabled by default.

```swift
.onAppear {
    self.image.priority = .normal
    self.image.fetch() // Restart the request if previous download failed
}
.onDisappear {
    self.image.priority = .low
}
```

`FetchImage` is still in its early stages. It is currently available as a [Swift package](https://github.com/kean/FetchImage) and will eventually be integrated into the main [Nuke repository](https://github.com/kean/Nuke).

## Design

I considered a number of different options when designing `FetchImage` API. I looked at multiple existing solutions but was not particularly satisfied with any of them.

The first approach was to wrap `UIImageView` (and `NSImageView` on macOS) in a SwiftUI view using `UIViewRepresentable` and expose some of the existing Nuke APIs using that wrapper view. What I quickly realized is that it was going to be more code and more complexity than just re-implementing the existing `UIImageView` extensions. Plus, I didn't feel like relying on `UIKit` was the right choice moving forward.

The second approach was to implement a custom `ImageView` and a respective `ImageViewModel`. This turned out to be a better option, but there were still a few problems. In an attempt to make `ImageView` customizable, I ended up creating something too generic with almost no functionality of its own.

```swift
public struct ImageView: View {
    public init<ImageView: View, PlaceholderView: View, ErrorView: View>(
        viewModel: ImageViewModel,
        @ViewBuilder image: @escaping (PlatformImage) -> ImageView,
        @ViewBuilder placeholder: @escaping () -> PlaceholderView,
        @ViewBuilder error: @escaping (Error) -> ErrorView
    ) 
}
```

The problem with this approach is that the `ImageView` itself ended up doing almost nothing. If you look at the [complete implementation](https://gist.github.com/kean/06dbb043b65c3a22b21a0adb1bee25d6) it's just wrappers on top of wrappers. I didn't feel like it was the right approach. SwiftUI makes it super easy to create views. Maybe there wasn't a need to create a custom view at all.

The final solution is [`FetchImage`](https://github.com/kean/FetchImage) presented in this post. It takes care of all of the heavy-lifting, including networking, caching, state management, and gives you full control over how to manage the download and how to display the image. As you've seen in the examples, SwiftUI makes it extremely easy to create any kind of views that you want to display the images, so I felt like there was no need to provide a "built-in" view.

## Final Thoughts

Implementing [`FetchImage`](https://github.com/kean/FetchImage) was a great exercise. It confirmed to me, once again, that SwiftUI is a fantastic framework that removes a need for a lot of boilerplate and makes creating views easy and fun.

> [**ImagePublisher**](https://github.com/kean/ImagePublisher)
>
> `ImagePublisher` is another Swift package that I introduced along with `FetchImage`. It is a publisher that wraps Nuke's image tasks (`ImageTask`) and enables a variety of different use cases shown in the repository.
