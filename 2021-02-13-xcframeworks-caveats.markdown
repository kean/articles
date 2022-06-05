---
layout: post
title: "XCFrameworks"
subtitle: Caveats of using XCFrameworks with Swift Package Manager
description: Caveats of using XCFrameworks
date: 2021-02-13 10:00:00 -0500
category: programming
tags: programming
permalink: /post/xcframeworks-caveats
uuid: d99799b5-aa4b-4403-8b65-aa639db7dc10
---

This post is about how one bad assumption about XCFrameworks turned into multiple hours of needless effort. I wanted to quickly share my experience so others could avoid falling into the same pitfall. In retrospect, the problem seems obvious, but it wasn't when I just encountered it. 

## Problem Statement

Last week I started working on a binary distribution for [Pulse](https://github.com/kean/Pulse). My initial dependency graph, before knowing anything about XCFrameworks, was this:

<img height="200px" alt="Pulse, a structured logging system built using SwiftUI" class="NewScreenshot" src="{{ site.url }}/images/posts/xcframeworks/01.png">

`PulseUI` and `Pulse` were Swift packages, each in its own repo. My initial goal was to distribute only `PulseUI` as a binary package.

## Initial Approach

My initial naive approach was to simply build `PulseUI` as an XCFramework.

<img height="200px" alt="Pulse, a structured logging system built using SwiftUI" class="NewScreenshot" src="{{ site.url }}/images/posts/xcframeworks/02.png">

This was based on an assumption that XCFrameworks can have dependencies on Swift Packages.

### Building XCFramework

To build an XCFramework you use `xcodebuild`. It's a multistep process that requires you to have an Xcode project which I didn't have. Fortunately, [swift-create-xcframework](https://github.com/marketplace/actions/swift-create-xcframework) came to rescue. It's a fantastic script and I can't recommend it enough.

> There are currently two great ways to build XCFrameworks:
>
> - [swift-create-xcframework](https://github.com/marketplace/actions/swift-create-xcframework), a script that can also be installed as GitHub Action
> - [create_xcframework](https://github.com/bielikb/fastlane-plugin-create_xcframework), a plugin for Fastlane
{:.info}

### Distributing XCFramework

Swift Package Manager supports XCFrameworks, so I went ahead and updated `PulseUI.package`:

```swift
// swift-tools-version:5.3
import PackageDescription

let package = Package(
    name: "PulseUI",
    platforms: [
        .iOS(.v13)
    ],
    products: [
        .library(name: "PulseUI", targets: ["PulseUI"])
    ],
    dependencies: [
        .package(url: "https://github.com/kean/pulse", from: "0.6.0")
    ],
    targets: [
        .binaryTarget(
            name: "PulseUI",
            url: "https://example.com/PulseUI-0.9.2.zip",
            checksum: "bdda7b5b5fa314a4035e048249b0dee557d65c161d9b2a4ddcbaf062317a5707"
        )
    ]
)
```

This is where I encountered the first red flag: there was no way to specify a dependency for a `.binaryTarget`. I went to Google and what I wished I found was this:

<img alt="Pulse, a structured logging system built using SwiftUI" class="NewScreenshot" src="{{ site.url }}/images/posts/xcframeworks/04.png">

But unfortunately, this picture is from the very end (`38:16`) of the [WWDC video from 2019](https://developer.apple.com/videos/play/wwdc2019/416/) which reduced its chances of me finding it on Google quite dramatically. What I found instead was [this thread](https://forums.swift.org/t/swiftpm-binary-target-with-sub-dependencies/40197) on Swift Forums which explains a workaround on how to add sub-dependencies for binary targets using fake targets which put me on a sidetrack[^1]. I thought OK, so you are saying there is a chance.

I reworked the package manifest, now was the time to try and integrate it into the app. Xcode installed the package without any complaints and I was able to run the app. The run failed.

```
dyld: Library not loaded: @rpath/Pulse.framework
  Reason: image not found
```

This is fine. XCFramework links to its dependencies dynamically and Xcode defaults to static linkage for Swift packages. As a package user, you can't change whether to use static or dynamic linkage, but you can do that as a package author. Knowing that I can't change `apple/swift-log` manifest, I still went ahead and changed `Pulse` library type to `.dynamic`. This also failed but with a different error this time:

```
dyld: Symbol not found: _$s9PulseCore18NetworkLoggerEventO12taskDidStartyA2C04TaskgH0VcACmFWC
  Referenced from:
```

### Library Evolution

My understanding, and I might be wrong here, is that the problem has to do with [Library Evolution](https://swift.org/blog/library-evolution/) (or the lack thereof it). The way library evolution work is by generating a `.swiftmodule` folder for your framework.

<img alt="Pulse, a structured logging system built using SwiftUI" class="NewScreenshot" src="{{ site.url }}/images/posts/xcframeworks/05.png">

One of the key parts of `.swiftmodule` is a textual `.swiftinterface` file which is a representation of the public interfaces of your framework.

```swift
// swift-interface-format-version: 1.0
// swift-compiler-version: Apple Swift version 5.3.2 (swiftlang-1200.0.45 clang-1200.0.32.28)
// swift-module-flags: -target arm64-apple-ios11.0 -enable-objc-interop \
// -enable-library-evolution -swift-version 5 -enforce-exclusivity=checked \
// -O -module-name PulseUI
import Combine
import CommonCrypto
import CoreData
import Foundation
import PulseCore
import Swift
import SwiftUI
import UIKit
@available(iOS 13.0, *)
public struct ConsoleView : SwiftUI.View {
  public init(messageStore: PulseCore.LoggerMessageStore = .default,
  	          blobStore: PulseCore.BlobStore = .default)
  public var body: some SwiftUI.View {
    get
  }
  public typealias Body = @_opaqueReturnTypeOf("$s7PulseUI11ConsoleViewV4bodyQrvp", 0) 🦸
}
...
```

These are a lot of intricacies when it comes to `.swiftinterface` but the main takeaways are:

- Xcode doesn't generate `.swiftinterface` for Swift packages, even linked dynamically as frameworks
- Without library evolution, an XCFramework can't reference symbols from a framework built from a source Swift package

But it looks like we are almost there with the tooling. For the sake of the experiment, I even tried combining different source-compatible versions of `Pulse.xcframework` and `PulseUI.xcframework` built at different times in the test app and it worked perfectly. I'm not entirely sure what's stopping Xcode from realizing that a binary dependency has a source dependency and compiling the source dependency with library evolution support enabled.

## Final Solution

What I ended up doing eventually is rewriting the framework so that there are no dependencies from binary frameworks to Swift packages.

**Before**

<img height="200px" alt="Pulse, a structured logging system built using SwiftUI" class="NewScreenshot" src="{{ site.url }}/images/posts/xcframeworks/02.png">

**After**

<img height="200px" alt="Pulse, a structured logging system built using SwiftUI" class="NewScreenshot" src="{{ site.url }}/images/posts/xcframeworks/03.png">

With this approach, I was finally able to ship my frameworks[^2]. As a final test, I uploaded the app with the frameworks to TestFlight.

If I were more diligent, I would've learned about the limitations before starting my work. I watched the WWDC video that I referenced earlier. I don't know how the crucial information about package dependencies exited my mind. Maybe I just haven't watched until the very end. At least it was before credits. Anyway, I didn't feel the need to rewatch it, because 90% of the video was trivial.

I'm now thinking, could Xcode do something to at least communicate the problem early? It couldn't when I was writing a manifest, initially the package dependency was in a different repo. But it could probably throw a more user-friendly error during dependency resolution saying that the configuration isn't supported.

The main thing is that with a few hiccups but I was able to achieve what I needed and Pulse could now be distributed.

{% include references-start.html %}

- [**Pulse**](https://github.com/kean/Pulse), a structured logging system
- [**swift-create-xcframework**](https://github.com/marketplace/actions/swift-create-xcframework) (GitHub Action)
- [**create_xcframework**](https://github.com/bielikb/fastlane-plugin-create_xcframework) (Fastlane plugin)
- [**WWDC 2019: Binary Frameworks in Swift**](https://developer.apple.com/videos/play/wwdc2019/416/)
- [**Library Evolution**](https://swift.org/blog/library-evolution/)

{% include references-end.html %}

[^1]: This solution works fine if your dependencies are also distributed as Swift binary frameworks, or Objective-C frameworks for that matter.
[^2]: Unfortunately, there was another small hiccup after I made this (massive) change. When I added `PulseUI` and `Pulse` to my app, I assumed that `PulseCore` would be automatically added as a transitive dependency. It turned out not to be the case. It worked during deployment, my guess is that all the binaries produced from Swift packages are copied to the bundle during the deployment. But it failed during Archive. And it failed when I tried dragging an product to a simulator.
