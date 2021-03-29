---
layout: post
title: "The SwiftUI Experiment"
subtitle: Restrospecive on using SwiftUI to build an app targeting all Apple platforms
description: Restrospecive on using SwiftUI to build an app targeting all Apple platforms
date: 2021-03-30 09:00:00 -0500
category: programming
tags: programming
permalink: /post/swiftui-experiment
uuid: 4559025a-577f-495f-93d8-dfc0c49f5b35
published: false
---

A couple of months ago, I started an experiment to answer one question: is SwiftUI ready for building non-trivial apps targeting all Apple platforms? It was an incredibly fun and sometimes challenging experience, but part of the journey is the end. This post is about all the things that I built and learned along the way.

## Beginning

I've been working on iOS for many years now. I was getting a bit bored. When Apple shipped SwiftUI, I saw it as a perfect opportunity to explore other Apple platforms. 

The best way to learn something is to build a project. SwiftUI has two main advantages: iteration speed and code reuse across Apple platforms. I had an idea for a project that would take advantage of both.

There is always this friction when it comes to debugging native apps: you can’t inspect anything that happens behind the scenes unless you use special tools, not even network requests.  That's not the case on the web with tools like Safari [Web Inspector](https://developer.apple.com/library/archive/documentation/AppleApplications/Conceptual/Safari_Developer_Guide/Introduction/Introduction.html). I wanted to bring something similar to native apps. This is how [Pulse](https://github.com/kean/Pulse) was started.

## Project

What is Pulse? It's a structured persistent logger with a network inspector consisting of:

- **PulseCore.framework** (iOS, macOS, watchOS, tvOS) with a persistent logger itself and a network proxy for automatically capturing network requests 
- **PulseUI.framework** (iOS, macOS, watchOS, tvOS) containing all the UI components you see on the screenshots
- **Pulse apps** (iOS, macOS), document-based apps to view logs shared from other devices

As a developer, you integrate the frameworks into your app, configure them to capture logs[^1] and network traffic, and add a way to display a Pulse console to view logs right on the device (can be a shake gesture, for example).

[^1]: If you are using [SwiftLog](https://github.com/apple/swift-log), you can also install `Pulse.framework` that adds `PersistentLogHandler` class that can be used as SwiftLog backend.

> Pulse frameworks are closed-source and are distributed as binary frameworks using XCFramework. You can learn more in a ["XCFrameworks"]({{ site.url }}/post/xcframeworks-caveats).
{:.info}

As a QA engineer testing the app, you can view the network requests right on your iOS device without installing any special tools. And when you encounter a defect, you can send the logs to developers using any native sharing mechanism. A developer can view the shared files using a dedicated Pulse app available both on macOS and iOS.

> Implementing a custom document format was a story on its own, so I covered it in ["What's a Document"]({{ site.url }}/post/pulse-store).
{:.info}

## Pulse Beta

TBD
TODO: nice flame animation
TODO: it's a pretty solid beta. people who wanted to make try to make it crash, I challenge them to ahead and try do that

## Development

### iOS

I started with the platform I felt the most comfortable with – iOS – with the initial goal to truly learn SwiftUI by putting it through its faces. After all, before building the app, I had to design it. I wanted to avoid the distraction of having to learn a new UX paradigm.

<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/pulse/pulse-01-console.png">
<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/pulse/pulse-02-inspector.png">
<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/pulse/pulse-03-share.png">

I built the entire app using SwiftU with a couple of minor exceptions: `UITextView`, `UISearchBar`, and `UIActivityViewController`. But it was a simple matter of wrapping them using [`UIViewRepresentable`](https://developer.apple.com/documentation/swiftui/uiviewrepresentable). The rest of the systems: [layout]({{ site.url }}/post/swiftui-layout-system), [data flow]({{ site.url }}/post/swiftui-data-flow), and the existing SwiftUI components, worked perfectly.

### macOS

The next milestone was macOS. When the time came to work on it, I was anxious. I had no idea what to expect from SwiftUI on a Mac. But thanks to the [new](https://developer.apple.com/videos/play/wwdc2020/10041/) SwiftUI APIs and the Big Sur design changes, everything just made sense!

<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/pulse/pulse-01.png">

I was so excited after building the macOS version, that I wrote an optimistic take on how [AppKit was done]({{ site.url }}/post/appkit-is-done), only to [correct myself]({{ site.url }}/post/not-list) later after I had spent more time on optimization. I encountered massive issues with [`List`](https://developer.apple.com/documentation/swiftui/list) performance and had to replace it with an AppKit version, including AppKit-based cells and context menus. Fortunately, I was able to get it to with the SwiftUI navigation system. It was worth it. The optimized version is fast even when working with hundreds of thousands of messages.

The rest of the macOS apps is written using SwiftUI, including the more challenging aspects of macOS: commands [`Commands`](https://developer.apple.com/documentation/swiftui/commands), window management ([`WindowGroup`](https://developer.apple.com/documentation/swiftui/windowgroup), [`handlesExternalEvents`](https://developer.apple.com/documentation/swiftui/group/handlesexternalevents(matching:)), focus management, etc.

> At the core of the macOS is a triple-column navigation setup. It was hard to find accurate information about it in SwiftUI, so I covered it in ["Triple Trouble"]({{ site.url }}/post/triple-trouble).
{:.info}

### watchOS

When I started working on Pulse, I challenged myself to push SwiftUI to the limit and not rely on the platform-specific code. I had my eyes on the prize that eventually I’ll add support for all Apple platforms, including watchOS and tvOS. It was time to reap the benefits.

After designing and implementing a macOS app, watchOS was a walk in the park. SwiftUI feels the most at home on this platform. It isn’t surprising since it’s _the only_ way to develop a watchOS app. When you use SwiftUI, you know that it gives you the full power of the platform.

Why bring Pulse to watchOS in the first place? Many of the watchOS apps are designed to be used outdoors and during activity. You won't be carrying a computer with you to a gym, will you? Pulse is a perfect tool for this. I was as excited deploying to my Apple Watch the first time as I was many years back working on my very first iOS project.

> You can learn more about Pulse for watchOS in ["Time to Log"]({{ site.url }}/post/time-to-debug).
{:.info}

<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/pulse/promo-6.png">

### tvOS

I'm not entirely sure if Pulse is needed on tvOS, so I haven't prioritized this platform. But since I already covered the rest of the platforms and had the energy to continue, I decided to give it a go, mostly for fun. And it was!

I was surprised to learn just how powerful the API for this platform is. It basically runs UIKit. There is a bit of a mismatch between the APIs and the typical design of the apps. It's counterintuitive, but the apps for the biggest screen are limited in a similar way that the watchOS apps are. I ended up reusing a lot of the elements from watchOS.

<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/pulse/promo-7.png">

## Restospective

Everyone wants to know the answer to the question: is SwiftUI ready for production? It's more nuanced than that. For Pulse, SwiftUI was undoubtedly the right tool. But there are some aspects of SwiftUI that I'm not crazy about.

### Went Well

- **Iteration Speed**

 Saying that developing with SwiftUI is fast is an understatement. It feels orders of magnitude faster than UIKit/AppKit[^2]. Canvas, layout system, data flow – all designed for maximum productivity.

[^2]: To clarify, it's mostly Xcode Canvas that accelerates the development. You can use it with UIKit/AppKit just as well as with SwiftUI. If you wrote declarative on top of UIKit/AppKit, do you really need SwiftUI?

- **Layout System (Good Parts)**

People love to praise [Flexbox](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Flexible_Box_Layout/Basic_Concepts_of_Flexbox). I use it on this website too and I'm not crazy about. I very much prefer SwiftUI stacks, grids, and spacers.
 
- **Code Reuse**

Pulse consists of **~10000** lines of code. PulseCore is **3000** lines (store and network proxy). The rest (**7000** lines) takes PulseUI. **85%** of the UI code is shared between platforms. That's a lot. It's astonishing how much UI you can fit in 7000 lines. SwiftUI, it gets the job done!

### Needs Improvement

SwiftUI increases iteration speed and enables code reuse. But is it because of its design? A lot of speed up comes from Xcode Canvas. And UIKit and AppKit have more in common than they are given credit for. The layout system – the same. A lot of components are one typealias away from reuse. Was UIKit so horrible that it needed a complete overhaul?

- **Generics**

I can't count how much time I wasted fighting the SwiftUI generics. Code completion is bad, error messages are bad, generated documentation is bad. It's a legitimate issue. Does it need to be this clever?

<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/swiftui-exp/gen01.png">
<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/swiftui-exp/gen02.png">

My most common mistake is using `&` instead of `$` and then wasting 20 minutes figuring out why there is an unrelated type-system error 10 lines from where I used `$`.

- **Layout System (Bad Parts)**

Some basic things that are super easy to do with Auto Layout are hard in SwiftUI. For example, matching views size or aligning views. SwiftUI layout system loses in power to Auto Layout. The good things about it (stacks, grids, spacers) are easy to add to Auto Layout. And I still can’t build a complete mental model around the SwiftUI layout system, while Auto Layout makes total sense – it’s math.

- **View Builders**

I'm used to it now, but it's still too clever to my taste. But I wouldn't bet against DSLs.

- **View Identity**

We are all used to memory management in UIKit. You create a ViewModel, you create a View, View holds a strong reference to a ViewModel - easy and clear. SwiftUI makes this basic setup unnecessarily awkward.

[`ObservedObject`](https://developer.apple.com/documentation/swiftui/observedobject) makes no guarantees about the object lifetime, so you can't use it. Managing ViewModel graph manually is a pain. [`StateObject`](https://developer.apple.com/documentation/swiftui/stateobject) uses autoclosure making initializer injection pain to use. It doesn't make sense in the first place because it can give you a wrong impression that you can inject new parameters without changing the identity of the view.

I figured data flow out, but I feel [like this]({{ site.url }}/images/posts/swiftui-exp/web.webp) every time I explain it to anyone. It just doesn't come from the use case. It comes from serving the needs of the framework, more specifically that the views are constantly recreated structs but at the same time maintain identity.

- **Conditionals**

It's harder to add platform-specific code (or any conditional code really) in SwiftUI than it is in UIKit/AppKit.

<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/swiftui-exp/cond01.png">

- **API Quirks**

```swift
.disableAutocorrection(true) // disable autocorrection
.disableAutocorrection(false) // don't disable autocorrection?
.disableAutocorrection(nil) // ???
.disableAutocorrection() // ???
```

I tried to [make sense](https://twitter.com/a_grebenyuk/status/1362644512959586304?s=20) of it, I still don't fully understand the logic. I never use the `nil` part. Regardless of the use-case, the naming is objectively bad.

- **Telescoping APIs**

UIKit/AppKit delegate-based approach is well-positioned to scale to any level of complexity. I’m interested to see how SwiftUI, which currently barely offers any customization, will tackle this problem.

## Conclusion

It’s one thing to watch WWDC videos on SwiftUI where everything is nice and smooth. But only when you use it extensively, you truly get a feel for it. I used pretty much every SwiftUI component throughout this experiment. SwiftUI is as fast as it is unforgiving. You structure the code in one predefined way, or you suffer. But when you figure it out, you can be extremely productive.

The hardest part of the design is determining what not to build. This applies both to apps and frameworks. I’m pretty happy with the Pulse feature-set, focused on my personal use-cases. I hope you’ll enjoy using it as much as I do.

## Further Reading

Working on Pulse was such an amazing experience. I if you want to read more, boy, I have posts for you.

<br/>

<div>
{% for post in site.posts  %}
    {% if post.uuid == "de347c14-59c3-4f15-9824-1cfc1781f299" or post.uuid == "424b2065-a294-4311-9bc7-f85ed82d1290" or post.uuid == "29cb612c-2e35-406d-a27a-a9c1b9f9c122" or post.uuid == "ef480460-ee95-4504-b4b1-ec133703b6b3" or post.uuid == "0526798f-a074-4365-8970-cb470579c358" or post.uuid == "d99799b5-aa4b-4403-8b65-aa639db7dc10" or post.uuid == "c5358288-8c59-41e0-a790-521b52f89921" or post.uuid == "2fe0b2d5-b449-4912-8719-fee0c4ad9cb0" %}
	{% include archive-post-list-item.html %}
	{% endif %}
{% endfor %}
</div>

<br/>

This wraps up the series on Pulse. I already have a new project in the works...

<div class="FootnotesSection" markdown="1">
</div>