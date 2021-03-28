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

There is always this friction when it comes to debugging native apps: you can’t inspect anything that happens behind the scenes unless you use special tools, not even network requests.  That's not the case on the web with tools like Safari [Web Inspector](https://developer.apple.com/library/archive/documentation/AppleApplications/Conceptual/Safari_Developer_Guide/Introduction/Introduction.html). I wanted to bring something similar to native apps.

## Project




I was always stuckprimarily focusing on one platform - iOS. What is the best time to try your hand at other platforms?

There are two major advantages of SwiftUI
- Faster iterations
- Target multiple Apple platforms at the same time

So I came up with an idea of a project that would take advantage of both – Pulse.

I coud probably write a few posts just explaining how to do certain things in SwiftUI, but I’m don’t want to add to documentation and maintain them later. 

Additive. I want to add some more command options, network inspector should be able to display pending tasks, remote debugging. These feature have nothing to do with SwiftUI, I already pushed it to its limits and used pretty much every SwiftUI feature available (maybe except for alignment guides, ugh this API is a mess). AutoLaotut is clearly better for some use cases than the SwiftUI simplified layout system. I’m also a bit uncomfortable with it even after writing about it (list), and studying. There are some areas that I haven’t build a mental modal around.

Old issues (e.g. almost impossible to get it to crash), but new issues, e.g. “type-safety hard”. Example with & instead of $. Maybe it’s just me an it then produces weird unrelated error messages in some other place. Hello, C++ templates.

TODO: “Failed to produce a diagnostic”
For better or worse, It’s pushing to generics system farther than it’s capable of.

TODO: “Selection” not inferred
This is a doom of me

# Restrospective
Good
Bad

About 20 minutes realizing why focusable() doesn’t work. You need to pass “true”. This is fucking stupid.

With the current limitation, I can safely say that generics were a mistake.

It’s so damn hard to work with List selection or navigation. I want to kill myself.

I think they done the best job they possible could. I’m a bit skeptical on generics (TODO: why do we need them). And about view identity and @State, @StateObject - too cumbersome to use. I prefer not to this about view identity, that’s the whole point.

It’s one thing when you watch WWDC videos and everything is nice and smooth. But when you are building with it, you feel like your are trying to tackle an F1 bolid. It’s fast, especially on a small project, but one wrong move and you are in quievet [video where iOS app navigation is wrong]. It’s unforgiving.

Add screenshot of self-destruct.

At first being able to playing all roles: product owner, designer, engineer seems liked a blessing. I was able to iterate so quickly. But then it became a bit of a burden. Turned out coming up with features and designed for them is hard! Who knew!

The hardest parts of design, be it product of frameworks, is determining what not to build. Too often I see features, especially on framework, that are clearly built because it was easy, but with no clear use case. 

My view about SwiftUI is largely unchanged. It’s a fantastic tool that allow you to iterate much quicker than UIKit. but it is also pretty hard to learn. My least favorite aspect is still view identity and nuances associated with state management (StateObject vs ObservedOBject). I don’t like it because in this case it feels like I’m serving framework neeess, not my needs. My needs are simply - I just want to associate a view model with a view. I should need to care about this details. But I have to because StateObject is designed as autoclosure. It is limited in what you can do and never know when you are going to hit a wall.

- Story of developments
- It’s a super solid beta, optimized and ready to run

- Since I shared the initial pre-alpha version
- A macOS version
- Numerous performance optimizations, especially on a Mac. I just can’t make a framework, and not make it an F1 car
- A watchOS version
- tvOS support
- An iOS app with documents support
- A custom document type

It’s freeing to work with closed source code-base. With open-source, you have to put yourself out there. It makes me spend more on clear code, sensible commits, tests that are not as much of a priority to write, better code names, better documentation.

All that within a month in my free time.


<img  class="NewScreenshot" src="{{ site.url }}/images/posts/time-to-debug/01.jpg">


<img width="320px" class="NewScreenshot" src="{{ site.url }}/images/posts/time-to-debug/w1.png"><img width="320px" class="NewScreenshot" src="{{ site.url }}/images/posts/time-to-debug/w2.png">
<img width="320px" class="NewScreenshot" src="{{ site.url }}/images/posts/time-to-debug/w3.png"><img width="320px" class="NewScreenshot" src="{{ site.url }}/images/posts/time-to-debug/w4.png">


<img width="375px" class="NewScreenshot" src="{{ site.url }}/images/posts/time-to-debug/i2.png">

<img width="375px" class="NewScreenshot" src="{{ site.url }}/images/posts/time-to-debug/i1.png">


<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/debug/details.mp4" type="video/mp4">
</video>
</div>

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/debug/share-to-phone.mp4" type="video/mp4">
</video>
</div>
