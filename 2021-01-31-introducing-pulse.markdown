---
layout: post
title: "Pulse: Network Inspector"
subtitle: Building a multiplatform tool using SwiftUI
description: Using SwiftUI to build a cross-platform (iOS and macOS) app.
date: 2021-01-31 10:00:00 -0500
category: programming
tags: programming
permalink: /post/pulse
uuid: a66c8ed5-555b-4755-a16a-45b7d71b9958
---

On iOS, there is this friction when it comes to debugging apps. You can't inspect anything that happens behind the scenes unless you use special tools. You can't even see network requests.

There are tools that can help: [Wireshark](https://www.wireshark.org), [Proxyman](https://proxyman.io), [Charles](http://charlesproxy.com), etc. These are all phenomenal tools, but they are all kind of complicated. For example, to start using Proxyman you need to [install a root certificate](https://proxyman.io/blog/2019/06/How-I-use-Proxyman-to-see-HTTP-requests-responses-on-my-iPhone.html) on your iPhone. It basically performs a MITM attack to see your encrypted traffic. [Charles](http://charlesproxy.com) is closer to what I want. It has an iOS app that works by creating a local VPN tunnel. But you can't inject IP packets into network interfaces on iOS, these APIs require privileged access. So it must have an entire TCP/IP stack re-implemented inside. I don't fully trust it to work and not interfere with your network connections.

What I wished iOS had is a simple analog of [Safari Web Inspector](https://developer.apple.com/safari/tools/). So I built just that. It also turned out to be a macOS app thanks to SwiftUI.

<a href="https://github.com/kean/Pulse">
<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/pulse-small.png">
</a>

## Demo

I recorded a short demo to show Pulse in practice:

<br/>

<iframe width="560" height="315" src="https://www.youtube.com/embed/17oQ9MF8Pq8" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Pulse

Pulse is not an app. It's a framework with programmatic interfaces for everything. It consists of the following parts:

- [**Pulse**](https://github.com/kean/Pulse), a logging system with structured persistent storage that implements [SwiftLog](https://github.com/apple/swift-log) protocol
- [**PulseUI**](https://github.com/kean/PulseUI), a SwiftUI-based framework with components that you can embed directly into your app, e.g. `ConsoleView`. This framework is private and is available for [**GitHub sponsors**](https://github.com/sponsors/kean). It will be open sourced when there are enough sponsors behind it.
- A macOS app based on **PulseUI**.

Integrating **PulseUI** is extremely easy. You add a package to your app and configure it to record the events and metrics provided by `URLSession`. There are no certificates or VPN tunnels involved and it only sees your app's `URLSession` traffic. You can show a `PulseUI.ConsoleView` to see the requests and other logs in real-time.

> You can bind the presentation of the `ConsoleView` to a shake gesture. This also allows you to use `Cmd+Ctrl+Z` shortcut in a simulator to display it, convenient.
{:.info}

## SwiftUI

The key to this project is SwiftUI. The speed with which I was able to develop it should not even be possible. With SwiftUI, you can program at a speed of thought (and as fast as you can type). Instant feedback is enlivening.

I used MVVM for this project. And I'm glad I did. When the time came to implement the "Share" feature to a network request, I was simply able to use the same ViewModels that I created for SwiftUI, but instead of binding them to Views, I printed all the outputs as text with no changes to ViewModels themselves. This is MVVM at its best.

## Screenshots

And here are a couple of screenshots to boot.

<img width="325px" src="/images/posts/pulse/ios-summary.png">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<img width="325px" src="/images/posts/pulse/ios-headers.png">

All these screens are built exclusively with SwiftUI. The only exception is a JSON viewer which I build using `UITextView`/`NSTextView` and attributed strings.

<img width="325px" src="/images/posts/pulse/ios-json-viewer.png">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<img width="325px" src="/images/posts/pulse/ios-network-inspector.png">

<br/>

On iPad, it just works. I made some tiny adjustments, but for the most part, it was looking good the first time I ever opened it on a bigger iPad screen.

<img src="/images/posts/pulse/ipad-charts.png">

<br/>

Thanks to SwiftUI, there is complete feature parity between platforms: iOS, tvOS, and even **macOS**.

<img class="Screenshot" src="/images/posts/pulse/macos-light.png">

And even support for Dark Mode.

<img class="Screenshot" src="/images/posts/pulse/macos-dark.png">

## Conclusion

I have a massive backlog for this project, but I'm already extremely happy with the results. It's an absolute pleasure to use. If you want to try it and support the project, please join to [GitHub sponsors](https://github.com/sponsors/kean).
