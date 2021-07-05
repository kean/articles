---
layout: post
title: "Open-Sourcing Pulse"
subtitle: Open-source developer tools
description: Open-source developer tools
date: 2021-07-06 09:00:00 -0500
category: programming
tags: programming
permalink: /post/open-sourcing-pulse
uuid: 629d1d3a-3b4f-4c0c-b6a3-873bf6ea7f42
---

[Pulse](https://kean.blog/pulse/home) is now [open-source](https://github.com/kean/Pulse.git). It includes the frameworks, the document-based apps for viewing the logs, and the demo projects.

<img width="533px" class="NewScreenshot" src="{{ site.url }}/images/posts/pulse/oss.png">

There are two main reasons for open-sourcing it.

- **Transparency**. The network logging setup [ranges](https://kean.blog/pulse/guides/networking) from manual to fully automatic. Fully automated logging requires swizzling. Regardless of what you decide to use, I want it to be fully transparent what the framework does under the hood.

- **Development**. Open-source is about building things together. There are a lot of things that can be added to Pulse. If you needed to add a feature, now you can.

There are plenty of open-source apps around – which wasn't the case just a few years ago. But I'm still excited to open-source it. Firstly, unlike many other projects, it supports all four Apple platforms. And it has some interesting stuff in it. There is SwiftUI[^1], database management, also [documents](https://kean.blog/post/pulse-store) support, and more.

[^1]: I started with the SwiftUI-only approach, but eventually ended up using quite a lot of AppKit too: sometimes for performance reasons ([table view](https://kean.blog/post/not-list)), sometimes for features (e.g. rich text display).

I initially started with a closed-source approach thinking about making it a commercial tool. But ultimately, I decided to go with open-source and sponsorships. If you are using this tool, I encourage you to [sponsor it](https://github.com/sponsors/kean) (you can think of it as "pay as much as you want") to help me maintain it and make it better in the future.

It was surprisingly liberating to work as a one-person team in a closed-source codebase. I didn't have to worry about how the code looks at all, only about delivering features quickly. But open-sourcing is a crucial step to allow the tool to reach its potential.
