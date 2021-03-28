---
layout: post
title: "Time to Log"
subtitle: Taking a debugger with you
description: Taking a debugger with you
date: 2021-03-29 09:00:00 -0500
category: programming
tags: programming
permalink: /post/time-to-debug
uuid: de347c14-59c3-4f15-9824-1cfc1781f299
---

When I started working on Pulse, I challenged myself to push SwiftUI to the limit and not rely on the platform-specific code. I had my eyes on the prize that eventually I'll add support for _all_ Apple platforms, including watchOS and tvOS. Time to reap the benefits.

### Take Debugger with You

After designing and implementing an iOS and macOS app, watchOS was a walk in the park. SwiftUI feels the most at home on this platform. It isn’t surprising since it’s _the only_ way to develop a watchOS app. When you use SwiftUI, you know that it gives you the full power of the platform. 

Updating Pulse to work on watchOS took me just a couple of hours. I was able to reuse the existing Pulse UI components on watchOS by combining them in new ways. And it was super easy, barely an inconvenience.

<img alt="Pulse, a structured logging system built using SwiftUI" class="Screenshot Any-responsiveCard" src="{{ site.url }}/images/posts/pulse/promo-6.png">

Why bring Pulse to watchOS in the first place? Many of the watchOS apps are designed to be used outdoors and during activity. You won’t be carrying a computer with you to a gym, will you? Pulse is a perfect tool for this.

With Pulse, you can view network requests and logs right on your wrist. All logs are recorded persistently and can be shared at any time. For example, you can send and view logs on your phone. Pulse for iOS offers a great experience and a feature-set on par with a macOS version.

<div class="BlogVideo NewScreenshot">
<video controls muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/debug/share-to-phone.mp4" type="video/mp4">
</video>
</div>

Pulse can be a great tool on a watch in a situation where you can't bring your computer with you. I was as excited deploying to my Apple Watch the first time as I was many years back working on my very first iOS project. I’m glad I had my old Apple Watch lying around. Otherwise, I wouldn’t have been able to test it to a full extent. For example, you [can’t seem](https://developer.apple.com/forums/thread/127672) to be to able test parts of WatchConnectivity functionality in a simulator.

## Conclusion

On this note, I'm ready to wrap up my SwiftUI experiment tomorrow.
