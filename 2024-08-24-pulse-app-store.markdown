---
layout: post
title: Pulse is on the App Store
subtitle: Pulse is now officially available on the App Store
description: Pulse is now officially available on the App Store
date: 2024-08-28 09:00:00 -0500
category: programming
tags: programming
permalink: /post/pulse-app-store
fullscreen: true
disable-toc: true
mastodon: https://mastodon.social/@a_grebenyuk/113045277333856450
uuid: cee1039d-4379-48ae-8fb6-10a14abf6900
---

Pulse apps are now officially available on the Mac App Store (and App Store)!

<a href="https://apps.apple.com/us/app/pulse-network-logger/id6661031747">
<img src="/images/misc/download-on-mac-app-store.svg" width="220px">
</a>

## What is Pulse?

Pulse is a developer tool for logging and debugging network requests in apps built for Apple platforms. You can start using Pulse by integrating its free and [open-source SDK](https://github.com/kean/Pulse) that allows you to collect and view logs on your test devices.

<a href="https://pulselogger.com">
<img class="NewScreenshot" src="/images/posts/pulse-app-store/promo-1.png">
</a>

Pulse has multiple advantages over traditional network proxies thanks to how it integrates into your apps with an SDK.

### No Root Certificates

Unlike traditional network proxies, Pulse integrates into your app on the `URLSession` level, which means that you never have to deal with root certificates. You integrate the SDK once and never have to worry about it again on any simulators or test devices. Pulse also doesn't proxy the traffic coming from your device, so the requests are sent to the server without any changes.

### Always Recording

Have you been in a situation where you found a bug and can't reproduce it? Pulse is always recording, so you can always get back to the requests from previous sessions. If someone from your team reports a bug, they can send you the entire recorded session to help you investigate the issue.

## Pulse Apps

The companion apps for Pulse are now available on the App Store for macOS and iOS, with visionOS coming soon.

### Live Remote Logging

With the Pulse app for macOS, you can view the logs shared using its space-efficient document format (other export options are available). You can also view logs from devices or simulators on your local network live. 

<img class="full-width JustVertMargins" src="/images/posts/pulse-app-store/app-store-01.png">

The console included in the Pulse SDK also has a convenient "View on Mac" button for quickly opening a selected request on your Mac without having to share the enire recorded session.

Pulse is also a general-purpose logger with SwiftLog support, and it can show your network requests together with other logs and errors. With access to the `URLSession`, it can show the request metrics or even point to the exact place where a decoding (`Codable`) error happened, which the traditional network proxies can't do.

The app also has a handy "Issues" navigator that shows you all your network, decoding, and other errors from the logs.

<img class="full-width JustVertMargins" src="/images/posts/pulse-app-store/app-store-02.png">

One of the newest additions to the Pulse app is mocking. It allows you to take one of your network requests and turn it into a mock or create a new one. And, again, unlike the regular proxies, you can use it to mock any of the `URLSession` errors because that's the level on which Pulse operates.

There is more stuff coming in the future versions, including breakpoints. There is more that Pulse can offer thanks to its SDK-level integration.

<img class="full-width JustVertMargins" src="/images/posts/pulse-app-store/app-store-03.png">

## Pricing

Pulse SDK continues to be free and open-source, and the companion apps that are finally out of beta will use a straightforward subscription model. By subscribing, you sponsor the project's future development and get access to these excellent companion apps to help accelerate the development of your projects.

I'm grateful to the early sponsors, and if you were one of them and you'd like to get free access to the apps, please [reach out](mailto:studio@kean.blog) to me.

## Future

A lot more is coming in the future, including the upcoming Pulse SDK 5.0 release that will greatly simplify the integration experience. The app will also continue leaning into the strengths of its SDK. It will offer a better way to analyze your network traffic together with logs and other diagnostics from the device, including identifying logs from the same tasks and areas. Stay tuned!