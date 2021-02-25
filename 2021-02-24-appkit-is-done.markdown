---
layout: post
title: "AppKit is Done"
subtitle: Learn how to build delightful macOS apps using only SwiftUI
description: Learn how to build delightful macOS apps using only SwiftUI
date: 2021-02-24 10:00:00 -0500
category: programming
tags: programming
permalink: /post/appkit-is-done
uuid: ef480460-ee95-4504-b4b1-ec133703b6b3
image:
  path: /images/posts/macos/cover-macos.png
  height: 1280
  width: 640
---

Well, not like Carbon. Don't be [so dramatic](https://www.youtube.com/watch?v=Cl7xQ8i3fc0)!

More like [Core Foundation](https://developer.apple.com/documentation/corefoundation). It's still there behind the scenes, but programmers use high-level Objective-C and Swift wrappers from Foundation. If something is missing, you can call an underlying C API. The relation between SwiftUI and AppKit is similar, for now[^6].

<img class="NewScreenshot" src="{{ site.url }}/images/posts/macos/cover-macos.png">
<!-- <img class="NewScreenshot" src="{{ site.url }}/images/posts/macos/03-macos-final.png"> -->

This is a native [macOS app](https://github.com/kean/Pulse) written entirely in SwiftUI, from `@main` to bottom. Not a prototype, not a toy. A full-featured app. The intention is to deliver the best macOS experience possible.

The two main reasons this app is possible are SwiftUI and Big Sur. I have extensive experience with iOS, so I started with it. When the time came to adopt it for macOS, I was anxious. I had no idea what to expect from SwiftUI on a Mac. But thanks to the [new](https://developer.apple.com/videos/play/wwdc2020/10041/) SwiftUI APIs and the Big Sur design changes, everything just made sense!

Total Lines: **7124**<br>
iOS: **593 (8%)**<br>
macOS: **513 (7%)**<br>
Shared: **6018 (85%)**[^1]<br>

App Size: **2.5 MB**

Want to learn how to build macOS apps or see some videos with rad UI interactions?<br> This post is for you. 

## Navigation

<!-- <img alt="Triple column navigation on macOS using SwiftUI" class="NewScreenshot" src="{{ site.url }}/images/posts/macos/00-columns.png"> -->

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/pulse/00-triple-2.mp4" type="video/mp4">
</video>
</div>

The [first step](https://twitter.com/a_grebenyuk/status/1363882900136095745?s=20) was to get the navigation right. I knew I wanted to use triple-column navigation[^4]. SwiftUI makes this setup ease.

```swift
struct ContentView: View {
    var body: some View {
        NavigationView {
            SidebarView()
            Text("No Sidebar Selection") // You won't see this in practice (default selection)
            Text("No Message Selection") // You will see this
        }
    }
}
```

The key is the type system. What you used to configure in code, e.g. by passing a number of columns to `UISplitViewController`, you now configure using generic parameters. In my case, the type of the view is `NavigationView<TupleView<(Sidebar, Text, Text)>>`.

Use navigation links as usual. To make the first tab active by default, pass [`isActive`](https://developer.apple.com/documentation/swiftui/navigationlink/init(_:destination:isactive:)-3v44) in the [`NavigationLink`](https://developer.apple.com/documentation/swiftui/navigationlink) initializer with a value of `true`.

```swift
struct SidebarView: View {
    @State private var isDefaultItemActive = true

    var body: some View {
        List {
            NavigationLink(destination: ConsoleView(), isActive: $isDefaultItemActive) {
                Label("Console", systemImage: "message")
            }
            // ...
        }.listStyle(SidebarListStyle()) // Gives you this sweet sidebar look
    }
}
```

By default, the toolbar displays a title in the main panel. To disable it, I used [`UnifiedWindowToolbarStyle(showsTitle: false)`](https://developer.apple.com/documentation/swiftui/scene/windowtoolbarstyle(_:)).

> To learn more about triple-column navigation, see [**Triple Trouble**](https://kean.blog/post/triple-trouble).
{:.info}

> Don't use the List [`selection`](https://developer.apple.com/documentation/swiftui/list#topics) API for navigation. It only exists to support single or multiple item selection of items in *edit* mode.
{:.warning}

## Sidebar

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/pulse/01-sidebar-icon.mp4" type="video/mp4">
</video>
</div>

The "Toggle Sidebar" button does not appear by default. To add it, use [`.toolbar`](https://developer.apple.com/documentation/swiftui/rotatedshape/toolbar(content:)-53znx?changes=latest_mino_8) modifier.

<!-- ```swift
SidebarView(model: model)
    .toolbar {
        Button(action: toggleSidebar) {
            Image(systemName: "sidebar.left")
                .help("Toggle Sidebar")
            }
        }
    }
}
``` -->

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kt">SidebarView</span><span class="p">(</span><span class="nv">model</span><span class="p">:</span> <span class="n">model</span><span class="p">)</span>
    <span class="o">.</span><a href="https://developer.apple.com/documentation/swiftui/rotatedshape/toolbar(content:)-53znx?changes=latest_mino_8"><span class="SwiftUIPostHighlightedCode"><span class="n">toolbar</span></span></a> <span class="p">{</span>
        <span class="kt">Button</span><span class="p">(</span><span class="nv">action</span><span class="p">:</span> <span class="n">toggleSidebar</span><span class="p">)</span> <span class="p">{</span>
            <span class="kt">Image</span><span class="p">(</span><span class="nv">systemName</span><span class="p">:</span> <span class="s">"sidebar.left"</span><span class="p">)</span>
                <span class="o">.</span><span class="nf">help</span><span class="p">(</span><span class="s">"Toggle Sidebar"</span><span class="p">)</span>
            <span class="p">}</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

Unfortunately, there doesn't seem to be a way to hide the sidebar programmatically using `NavigationView`. I had to use [`NSApp`](https://developer.apple.com/documentation/appkit/nsapp) APIs. One of the few AppKit usages in the app.

```swift
private func toggleSidebar() {
    NSApp.keyWindow?.firstResponder?
        .tryToPerform(#selector(NSSplitViewController.toggleSidebar(_:)), with: nil)
}
```

> The relationship between SwiftUI and AppKit are not documented and not guaranteed to be supported. This workaround is useful for now, but might stop working in the future.
{:.warning}

To add a "Toggle Sidebar" shortcut, use [`SidebarCommands`](https://developer.apple.com/documentation/swiftui/sidebarcommands).

<img class="NewScreenshot" src="{{ site.url }}/images/posts/macos/01-menu-items.png">

> If you have a group of items, put them in [`ToolbarItemGroup`](https://developer.apple.com/documentation/swiftui/toolbaritemgroup). This will ensure proper positioning and spacing.
{:.info}

> Customize the positioning on buttons with [`ToolbarItemPlacement`](https://developer.apple.com/documentation/swiftui/toolbaritemplacement). The positioning that you can see on the video is [the default option](https://developer.apple.com/documentation/swiftui/toolbaritemplacement/automatic), no customization is needed.
{:.info}

## Outline Groups

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/pulse/02-outline-group.mp4" type="video/mp4">
</video>
</div>

To create a collapsing section, use [`DisclosureGroup`](https://developer.apple.com/documentation/swiftui/disclosuregroup). You can even nest sections. To open a section programmatically, use [`isExpanded`](https://developer.apple.com/documentation/swiftui/disclosuregroup/init(isexpanded:content:label:)) binding.

```swift
DisclosureGroup(content: {
    Toggle("Current Session", isOn: $model.isCurrentSession)
    // ...
}, label: {
    Label("Log Level", systemImage: "flag")
})
```

> To learn more about outlines, see [WWDC 2020: Stacks, Grids, and Outlines in SwiftUI](https://developer.apple.com/videos/play/wwdc2020/10031/)
{:.info}


## Windows

With the new [`WindowGroup`](https://developer.apple.com/documentation/swiftui/windowgroup) API introduces in Big Sur, SwiftUI takes care of certain platform behaviors. For example, users can open more than one window from the group simultaneously. When the user double-clicks on a list item in the main panel, it opens the details in a separate window. This happens automatically.

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/pulse/07-double-tap.mp4" type="video/mp4">
</video>
</div>

It also comes with a standard set of shortcuts: `Command-Tilde` to switch between open windows, `Command+W` to close, `Command+M` to minimize, etc.

You can also open a new window group using `Command+N`. In my case, the app remembers the database you selected. This way you can have multiple views on the same set of logs with different filters. Very useful when investigating complex issues!

```swift
@main
struct PulseApp: App {
    @StateObject var model = AppViewModel()

    var body: some Scene {
        WindowGroup {
            contents
        }
        .windowToolbarStyle(UnifiedWindowToolbarStyle(showsTitle: false))
        .commands {
            // Adds `Command+Option+S` shortcut to toggle the sidebar
            SidebarCommands()
            // ...
        }
    }

    @ViewBuilder
    private var contents: some View {
        if let store = model.selectedStore {
            MainView(messageStore: store)
        } else {
            WelcomeView()
        }
    }
}
```

The way `WindowGroup` works is clever: it's just struct copying. Every window created from the group maintains independent state. For example, for each new window created from the group the system allocates new storage for any `State` or `StateObject` variables instantiated by the scene’s view hierarchy.

> To learn more about data flow in SwiftUI, see [SwiftUI Data Flow](https://kean.blog/post/swiftui-data-flow).
{:.info}


## List Search

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/pulse/04-search-4.mp4" type="video/mp4">
</video>
</div>

Search is probably the most complex UI feature I built. It felt like I was pushing it.

The search field, the search toolbar, and how the searches are performed is not particularly interesting – same as in AppKit. What is, is how do you scroll to the next item and activate the navigation link each time the user hits the Return (↩) key?

For programmatic scrolling, I used [`ScrollViewReader`](https://developer.apple.com/documentation/swiftui/scrollviewreader).

```swift
struct ConsoleContentView: View {
    @ObservedObject private(set) var model: ConsoleViewModel

    var body: some View {
        VStack(spacing: 0) {
            if !model.isSearchBarHidden {
                searchToolbar
            }
            ScrollViewReader { proxy in
                ConsoleMessageListView(model: model)
                    .onReceive(model.$selectedObjectId) { objectId in
                        guard let objectId = objectId else { return }
                        proxy.scrollTo(objectId)
                    }

            }
            filterToolbar
        }
    }
}
```

When the user hits Return (↩), `ConsoleViewModel` updates the [published](https://developer.apple.com/documentation/combine/published) `selectedObjectId` property value observed by the view.

> Every declarative framework is imperative if you try hard enough.

Now, what about activating the navigation links? We already know how to do that using [`isActive`](https://developer.apple.com/documentation/swiftui/navigationlink/init(_:destination:isactive:)-3v44). But what do you do in a dynamic list? The solution I came up with is to create an array of bindings and grow it every time the number of items increases.

```swift
final class ConsoleViewModel: ObservableObject {
    @Published private(set) var messages: [MessageEntity]
    @Published var isLinkActive: [Bool]

    private func refresh() {
        // ....
        messages = /* new array */
        if isLinkActive.count < messages.count {
            isLinkActive += Array(repeating: false, count: messages.count - isLinkActive.count)
        }
    }

    private func scrollToMatch(_ match: Match) {
        selectedObjectId = match.objectID
        isLinkActive[match.index] = true
    }
}
```

```swift
struct ConsoleMessageListView: View {
    var body: some View {
        List(messages, rowContent: makeListItem)
    }

    private func makeListItem(message: MessageEntity) -> some View {
        NavigationLink(destination: DetailsView(message),
                       isActive: $model.isLinkActive[row.index]) {
            ConsoleMessageView(message)
        }
    }
}
```

By the way, this is not the only way to navigate a list with a keyboard. By default, on macOS, List can also be navigated with "Up" and "Down" arrows!

## Text View

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/pulse/03-search-text-view.mp4" type="video/mp4">
</video>
</div>

A similar search is also available on a details screen. This is the only place (except for `NSSearchField`) where I had to use `AppKit` components directly. SwiftUI doesn't yet have a wrapper for `NSTextView` (except for [`TextEditor`](https://developer.apple.com/documentation/swiftui/texteditor), but it doesn't support attributed strings that I needed for syntax highlighting). So I had to create one.

Now here is the question. Does it mean that SwiftUI is somewhat incomplete? Maybe. After all, text view is a common component. But SwiftUI is built on top of AppKit, it is not going anywhere. Just because I had to wrap two components doesn’t mean that SwiftUI was in the way. Quite the opposite! The AppKit integration works great.

## Deep Links

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/pulse/05-deep-links.mp4" type="video/mp4">
</video>
</div>

Despite the rumors of SwiftUI's lack of programmatic navigation, it exists.

One of the cool Pulse features is the ability to _pin_ important messages. Imagine you are investigating a bug report, you found some related messages, but you are not sure. You can pin them and get back to them later in the "Pins" tab.

> You can also pin a selected item using `Command+P` shortcut.

But this isn't enough, is it? What you want to do is the ability to open pinned messages in the list. Pulse allows you to just that.

To implement it, `PinsViewModel` sends a message to `MainViewModel`: "hey, open a message with this ID". `MainViewModel` activates the console navigation link and asks `ConsoleViewModel` to scroll to the given link. The console uses the same implementation from [search](#list-search). Simple.

## Context Menus

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/pulse/10-context-menus.mp4" type="video/mp4">
</video>
</div>

To add a context menu to the view, use [`.contextMenu`](https://developer.apple.com/documentation/swiftui/view/contextmenu(menuitems:)) modifier. It's relatively easy to use and there are plenty of code samples online, including the official documentation.

The context menus look different on different platforms. On iOS, you invoke a context menu using a long-press. On macOS, it's a right-click. You can add context menu to pretty much any views.

[`Menu`](https://developer.apple.com/documentation/swiftui/menu) is a related component. Unlike `.contextMenu`, a menu has a `label` (can be a button) to be displayed on the screen. To invoke a menu, you press the button. On macOS the menu also automatically adds an arrow to the control.

Menus can have pickers, this is how you create hierarchy.

```swift
Menu(content: {
    Picker(searchOptions.isCaseSensitive ? "Case Sensitive" : "Case Insensitive",
           selection: $searchOptions.isCaseSensitive) {
        Text("Case Sensitive").tag(true)
        Text("Case Insensitive").tag(false)
    }
    // ...
}
```

> The menu from the video can be improved even further. When there are only a couple of items, there is no need for a hierarchal menu. To flatten the hierarchy, use [InlinePickerStyle](https://developer.apple.com/documentation/swiftui/inlinepickerstyle).
{:.info}

## Share

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/pulse/09-share-2.mp4" type="video/mp4">
</video>
</div>

Unlike iOS, there are multiple ways to implement sharing on macOS. A common approach is to fetch the list of sharing services available for the current item and put them in a context menu. My initial implementation looked like this:

```swift
Menu {
    ForEach(NSSharingService.sharingServices(forItems: preview), id: \.title) { service in
        Button(action: { service.perform(withItems: self.items()) }) {
            Image(nsImage: service.image)
            Text(service.title)
        }
    }
}
```

Now, here is a problem, the first time you call `NSSharingService.sharingServices` it is _slow_. And when you create a menu, its content is evaluated eagerly. When I added the menu to the details screen, the first time I open it, it would be unacceptably slow.

My first attempt to fix it was to load services asynchronously when the menu appears while showing a placeholder "Loading..." button. This didn't work. The menu just wouldn't reload when the request is done.

What I ended up doing – and I'm not sure I'm happy about – is pre-fetching the list of services when the app starts and just hoping that the user doesn't open the share menu before that.

## Lightning Round

If that wasn't enough, here are 13 more tips.

<div class="NewList" markdown="1">

- Use [`.help`](https://developer.apple.com/documentation/swiftui/view/help(_:)-9lm7l) to configure the view's accessibility hint and its tooltip (“help tag”)
- Use [`.keyboardShortcut`](https://developer.apple.com/documentation/swiftui/keyboardshortcut) to add shortcuts for common actions that are _not_ in the app menu. For example, `.keyboardShortcut("f", modifiers: [.shift, .option])`.
- Use [`.keyboardShortcut(.defaultAction)`](https://developer.apple.com/documentation/swiftui/keyboardshortcut/defaultaction) on a button to make it a default button (will use accent color) and register it for Return (↩) keyboard shortcut. Use [`.cancelAction`](https://developer.apple.com/documentation/swiftui/keyboardshortcut/cancelaction) for Escape.
- Use [`FocusedValues`](https://developer.apple.com/documentation/swiftui/focusedvalues) to add custom menu commands that operate on selected items. Apple has a good guide on [the subject](https://developer.apple.com/tutorials/swiftui/creating-a-macos-app#Add-a-Custom-Menu-Command).
- To change your app's accent color, add a new color to your assets catalog and set it as a "Global Accent Color Name" in the build settings. On Big Sur, the system uses your app's accent color by default, but the user can change the accent color to one of the predefined colors. Make sure your app looks great with all of them.
- Use [`GeometryReader`](https://developer.apple.com/documentation/swiftui/geometryreader) for layouts where SwiftUI layout system doesn't cut it. I used it for [metrics chart]({{ site.url }}/images/posts/macos/02-metrics.png).
- When adding commands, use [CommandGroupPlacement](https://developer.apple.com/documentation/swiftui/commandgroupplacement) to place items in one of the standard locations. For example, for "Open" command, use [`.newItem`](https://developer.apple.com/documentation/swiftui/commandgroupplacement/newitem).
- SwiftUI has several built-in command groups, e.g. [`SidebarCommands`](https://developer.apple.com/documentation/swiftui/sidebarcommands), [`ToolbarCommands`](https://developer.apple.com/documentation/swiftui/toolbarcommands), [`TextEditingCommands`](https://developer.apple.com/documentation/swiftui/texteditingcommands), [`TextFormattingCommands`](https://developer.apple.com/documentation/swiftui/textformattingcommands).
- SwiftUI [layout system](https://kean.blog/post/swiftui-layout-system) and [data flow](https://kean.blog/post/swiftui-data-flow) are the same on macOS and iOS, see the linked posts to learn more. Gotta learn these first.
- Use [`Settings`](https://developer.apple.com/documentation/swiftui/settings) to add a settings screen, don't add it anywhere in the app itself
- Instead of `navigationBarTitle`, use [`navigationTitle`](https://developer.apple.com/documentation/swiftui/view/navigationtitle(_:)-43srq). macOS also supports [subtitles](https://developer.apple.com/documentation/swiftui/view/navigationsubtitle(_:)-262n7).
- Use [`Alert`](https://developer.apple.com/documentation/swiftui/alert) to show alerts, same API as iOS
- Prioritize watching WWDC and reading official documentation, there is now a lot of outdated information about SwiftUI online

</div>

## Conclusion

Is SwiftUI perfect?

No. I had to compromise in a few places. But I don't have a lot of bugs to report[^3].  Maybe I'm just getting better at avoiding things that don't quite work as expected. There are some limitations, but the AppKit integration is always there for me.

One of the criticisms I hear a lot is: "I started using SwiftUI and it took me T amount of time to build X, it would've taken me a fraction of time to do that in AppKit". I voiced this criticism. SwiftUI is complex and is not magic. It is almost nothing like AppKit which I think is a good thing. But it means that you need to learn a lot before you can use it efficiently, even if you are already familiar with the core principles behind it.

Is SwiftUI a game-changer for macOS?

It's economics. Choosing web technologies used to be almost a no-brainer for many companies. But the things are changing. On one hand, there is M1 which is finally powerful and energy-efficient enough to run Slack, rejoice! On the other hand, the calculation has changed. The main impediment, AppKit, is gone. There are millions of iOS engineers on the market. The path to delivering great native experience on Apple platforms has never been clearer[^5]. I heard you can even put Swift on a server, but was not able to reach Tim Cook for comment[^2].

<div class="FootnotesSection" markdown="1">

[^1]: I'm positive that I can increase the amount of the shared code to at least 90%. It wasn't a priority, my main focus was on the delivery.
[^2]: I love Swift on a server. If you haven't checked it out, go see [Vapor](https://vapor.codes). It's a vast ecosystem that keeps expanding. With coroutines in Swift 6, this is going to be insanely great.
[^3]: I see many Catalyst defects reported on Twitter which can sometimes be confused with SwiftUI defects. I'm a bit surprised this configuration (SwiftUI Catalyst) is even supported. But of course, UIKit apps that chose to support Catalyst need a path forward to adopt SwiftUI, so it has to be there. For new apps, if you are writing SwiftUI, there should be no reason to use Catalyst, you should compile for macOS natively.
[^4]: The iPhone app was developed first and it uses the simplest `TabView` and `NavigationView` configuration possible. I then jumped straight into developing a macOS app skipping iPad. The reason I did it this way is that iPhone and Mac are simply higher priorities for me. The iPad app currently looks very similar to iPhone (it has `TabView`), but it uses double-column navigation provided by `NavigationView` by default.
[^5]: Let's not talk about Catalyst, shall we?
[^6]: Many parts of SwiftUI don't use AppKit. The relationship between SwiftUI and AppKit are not documented and not guaranteed to be supported. AppKit might be completely removed in the future, but we don't know that.