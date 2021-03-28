---
layout: post
title: "SwiftUI Experiment"
subtitle: Restrospecive on using SwiftUI to build an app targeting all Apple platforms
description: Restrospecive on using SwiftUI to build an app targeting all Apple platforms
date: 2021-03-30 09:00:00 -0500
category: programming
tags: programming
permalink: /post/swiftui-experiment
uuid: 4559025a-577f-495f-93d8-dfc0c49f5b35
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

I was so excited after building the macOS version, that I wrote an optimistic take on how [AppKit was done]({{ site.url }}/post/appkit-is-done), only to [correct myself]({{ site.url }}/post/not-list) later after I had spent more time on optimization. I encountered massive issues with [`List`](https://developer.apple.com/documentation/swiftui/list) performance and had to replace it with an AppKit version, including AppKit-based cells and context menus. Fortunately, I was able to get it to with the SwiftUI navigation system. I think it was worth it. The optimized version is fast even when working with hundreds of thousands of messages.

The rest of the macOS apps is written using SwiftUI, including the more challenging aspects of macOS: commands [`Commands`](https://developer.apple.com/documentation/swiftui/commands), window management ([`WindowGroup`](https://developer.apple.com/documentation/swiftui/windowgroup), [`handlesExternalEvents`](https://developer.apple.com/documentation/swiftui/group/handlesexternalevents(matching:)), focus management, etc.

> At the core of the macOS is a triple-column navigation setup. It was hard to find accurate information about it in SwiftUI, so I covered it in ["Triple Trouble"]({{ site.url }}/post/triple-trouble).
{:.info}

### watchOS

After designing and implementing a macOS app, watchOS was a walk in the park. SwiftUI feels the most at home on this platform. It isn’t surprising since it’s the only way to develop a watchOS app. When you use SwiftUI, you know that it gives you the full power of the platform. You can learn more about Pulse for watchOS in ["Time to Debug"]({{ site.url }}/post/time-to-debug). Pulse can be a great tool on a watch in situation where you can't bring you computer with you.

<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/pulse/promo-6.png">

### tvOS

I'm not entirely sure if Pulse is needed on tvOS, so I haven't prioritized this platform. But since I already covered the rest of the platforms and had the energy to continue, I decided to give it a go, mostly for fun. And it was!

I was surprised to learn just how powerful the API for this platform is. It basically runs UIKit. There is a bit of mismatch between the APIs and the typical design of the apps. It's counterintutive, but the apps for the biggest screen are limited in a similar way that the watchOS apps are. I ended up reusing a lot of the elements from watchOS.

<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/pulse/promo-7.png">

Additive. I want to add some more command options, network inspector should be able to display pending tasks, remote debugging. These feature have nothing to do with SwiftUI, I already pushed it to its limits and used pretty much every SwiftUI feature available (maybe except for alignment guides, ugh this API is a mess). AutoLaotut is clearly better for some use cases than the SwiftUI simplified layout system. I’m also a bit uncomfortable with it even after writing about it (list), and studying. There are some areas that I haven’t build a mental modal around.

Old issues (e.g. almost impossible to get it to crash), but new issues, e.g. “type-safety hard”. Example with & instead of $. Maybe it’s just me an it then produces weird unrelated error messages in some other place. Hello, C++ templates.

TODO: “Failed to produce a diagnostic”
For better or worse, It’s pushing to generics system farther than it’s capable of.

TODO: “Selection” not inferred
This is a doom of me

## Went Well

## Needs Improvement


About 20 minutes realizing why focusable() doesn’t work. You need to pass “true”. This is fucking stupid.

With the current limitation, I can safely say that generics were a mistake.

It’s so damn hard to work with List selection or navigation. I want to kill myself.

I think they done the best job they possible could. I’m a bit skeptical on generics (TODO: why do we need them). And about view identity and @State, @StateObject - too cumbersome to use. I prefer not to this about view identity, that’s the whole point.

It’s one thing when you watch WWDC videos and everything is nice and smooth. But when you are building with it, you feel like your are trying to tackle an F1 bolid. It’s fast, especially on a small project, but one wrong move and you are in quievet [video where iOS app navigation is wrong]. It’s unforgiving.

Add screenshot of self-destruct.

At first being able to playing all roles: product owner, designer, engineer seems liked a blessing. I was able to iterate so quickly. But then it became a bit of a burden. Turned out coming up with features and designed for them is hard! Who knew!

The hardest parts of design, be it product of frameworks, is determining what not to build. Too often I see features, especially on framework, that are clearly built because it was easy, but with no clear use case. 

My view about SwiftUI is largely unchanged. It’s a fantastic tool that allow you to iterate much quicker than UIKit. but it is also pretty hard to learn. My least favorite aspect is still view identity and nuances associated with state management (StateObject vs ObservedOBject). I don’t like it because in this case it feels like I’m serving framework neeess, not my needs. My needs are simply - I just want to associate a view model with a view. I should need to care about this details. But I have to because StateObject is designed as autoclosure. It is limited in what you can do and never know when you are going to hit a wall.

- Since I shared the initial pre-alpha version
- A macOS version
- Numerous performance optimizations, especially on a Mac. I just can’t make a framework, and not make it an F1 car
- A watchOS version
- tvOS support
- An iOS app with documents support
- A custom document type


## Pulse Beta

It’s freeing to work with closed source code-base. With open-source, you have to put yourself out there. It makes me spend more on clear code, sensible commits, tests that are not as much of a priority to write, better code names, better documentation.

- It’s a super solid beta, optimized and ready to run

## Further Reading

TODO: from archive

- ["XCFrameworks"]({{ site.url }}/post/xcframeworks-caveats)
- ["What's a Document"]({{ site.url }}/post/pulse-store)
- ["AppKit is Done"]({{ site.url }}/post/appkit-is-done)
- ["...But not NSTableView"]({{ site.url }}post/not-list)
- ["Triple Trouble"]({{ site.url }}/post/triple-trouble)
- ["Time to Debug]({{ site.url }}/post/time-to-debug)
- ["SwiftUI Layout System"]({{ site.url }}/post/swiftui-layout-system)
- ["SwiftUI Data Flow]({{ site.url }}/post/swiftui-data-flow)



## Conclusion

I coud probably write a few posts just explaining how to do certain things in SwiftUI, but I’m don’t want to add to documentation and maintain them later.

All that within a month in my free time.
