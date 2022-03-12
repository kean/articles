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

There are tools that can help: [Wireshark](https://www.wireshark.org), [Proxyman](https://proxyman.io), [Charles](http://charlesproxy.com), etc. These are all phenomenal tools, but they are all kind of complicated. For example, to start using Proxyman you need to [install a root certificate](https://proxyman.io/blog/2019/06/How-I-use-Proxyman-to-see-HTTP-requests-responses-on-my-iPhone.html) on your iPhone. It basically performs a MITM attack to see your encrypted traffic. [Charles](http://charlesproxy.com) is closer to what I want. My understanding is that it does it by using Packet Tunnel Provider with a custom HTTP proxy.

*Update: Proxyman released a [Proxyman iOS Beta](https://twitter.com/proxyman_app/status/1357314980471656450?s=20) right after this post, a great Charles alternative.*

What I wished iOS had is a simple analog of [Safari Web Inspector](https://developer.apple.com/safari/tools/). So I built just that. It also turned out to be a macOS app thanks to SwiftUI.

<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot kb-legacy-card" src="{{ site.url }}/images/posts/pulse/pulse-02.png">
<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot kb-legacy-card" src="{{ site.url }}/images/posts/pulse/pulse-01.png">
<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot kb-legacy-card" src="{{ site.url }}/images/posts/pulse/pulse-01-console.png">
<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot kb-legacy-card" src="{{ site.url }}/images/posts/pulse/pulse-02-inspector.png">
<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot kb-legacy-card" src="{{ site.url }}/images/posts/pulse/pulse-03-share.png">


## Pulse

Pulse is not an app. It's a framework with programmatic interfaces for everything. It consists of the following parts:

- [**Pulse**](https://github.com/kean/Pulse), a logging system with structured persistent storage that implements [SwiftLog](https://github.com/apple/swift-log) protocol
- [**PulseUI**](https://github.com/kean/PulseUI), a SwiftUI-based framework with components that you can embed directly into your app, e.g. `ConsoleView`. This framework is open source.
- A macOS app based on **PulseUI**.

Integrating **PulseUI** is easy. You add a package to your app and configure it to record the events and metrics provided by `URLSession`. There are no certificates or VPN tunnels involved and it only sees your app's `URLSession` traffic. You can show a `PulseUI.ConsoleView` to see the requests and other logs in real-time.

> You can bind the presentation of the `ConsoleView` to a shake gesture. This also allows you to use `Cmd+Ctrl+Z` shortcut in a simulator to display it, convenient.
{:.info}

The main advantage of Pulse is that it is always there and recording: logs, network events, metrics. And you don't have to be in front of a computer to use it. If a team member encounters an issue in one of your builds, all the information is recorded and can be shared using any native mechanism, such as AirDrop. It's super easy.

I think the best way to think about Pulse is not as a network debugging tool but as a comprehensive logging system.

## SwiftUI

The key to this project is SwiftUI. The speed with which I was able to develop it should not even be possible. With SwiftUI, you can program at a speed of thought (and as fast as you can type). Instant feedback is enlivening.

I also used MVVM. And I'm glad I did. When the time came to implement the "Share" feature, I was simply able to use the same ViewModels that I created for SwiftUI views, but instead of binding them to Views, I was able to simply print all the outputs as text. No changes to the ViewModels themselves. This is MVVM at its best.

## Conclusion

I have a massive backlog for this project, but I'm already extremely happy with the results.