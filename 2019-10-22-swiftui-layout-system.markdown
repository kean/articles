---
layout: post
title: "SwiftUI Layout System"
description: Taking a deep dive into SwiftUI layout system
date: 2019-10-22 9:00:00 +0300
category: programming
tags: programming
permalink: /post/swiftui-layout-system
uuid: 2fe0b2d5-b449-4912-8719-fee0c4ad9cb0
---

Everything about SwiftUI is new. And the layout system is no exception. SwiftUI no longer uses Auto Layout, gone all of [the cruft](https://developer.apple.com/documentation/uikit/uiview/1622572-translatesautoresizingmaskintoco) introduced over the years. SwiftUI has a completely new layout system designed from the ground up to make it easy to write adaptive cross-platform apps.

I have always been fascinated by the layout systems. I built an [open-source UIStackView replacement]({{ site.url }}/post/lets-build-uistackview), designed a [convenience API]({{ site.url }}/post/align) on top of Auto Layout (granted, many people did). I also have experience working with the layout systems on the web, including [Flexbox](https://developer.mozilla.org/en-US/docs/Learn/CSS/CSS_layout/Flexbox). I can't be more excited to dig deep into the SwiftUI layout system to see what it has to offer.

{% include ad-hor.html %}

## Layout Basics

Let's start with the most basic "Hello World" example.

<div class="SwiftUIExampleWithScreenshot Any-responsiveCard">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left">
This is the code that gets generated when you select <b>File / New File... / SwiftUI View</b> in Xcode.

{% highlight swift %}
import SwiftUI

struct ContentView: View {
    var body: some View {
        Text("Hello World")
    }
}
{% endhighlight %}
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right">
            <img src="{{ site.url }}/images/posts/swiftui-layout/case-01.png">
        </div>
    </div>
</div>

The moment you open the preview, you are already experiencing the layout system. The blue box in the preview editor indicates the bounds of the `ContentView` on screen. The bounds of the view are the same as its body, the text, which is at the bottom of the view hierarchy[^1]. And finally, the root view which in this case has the dimensions of the device minus the **safe area** insets.

<blockquote class="SwiftUIBlockquote" markdown="1">
**Safe Area**

[Safe area](https://developer.apple.com/documentation/uikit/uiview/positioning_content_relative_to_the_safe_area) helps you place your views within the visible portion of the overall interface. In the example, the safe area of an interface excludes the status bar area. In SwiftUI, you are in the safety zone by default. You can still lay views out outside the safe area using the following modifier:
```swift
Text("Hello World")
    .edgesIgnoringSafeArea(.all)
```
</blockquote>

The top layer of any custom view, like `ContentView`, is *layout neutral*. Its bounds are defined by the bounds of its body, in this case, `Text`. For the purposes of layout, you can treat the custom `ContentView` and `Text` as the same view. Now how did SwiftUI establish the bounds of the `ContentView` and why did it it position it in the center of the root view? To understand this, we need to understand how SwiftUI layout system works.

### Layout Process

There are three steps in SwiftUI layout process.

<div class="SwiftUIExampleWithScreenshot Any-responsiveCard">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left">

            <h4>1. Parent Proposes Size for Child</h4>

            <p>First, the root view offers the text a proposed size – in this case, the entire safe area of the screen, represeted by an orange rectangle.</p>
            <br/>

            <h4>2. Child Chooses its Size</h4>

            <p>Text only requires that much size to draw its content. The parent has to respect the child's choice. It doesn't stretch or compress the child.</p>
            <br/>

            <h4>3. Parent Places Child in Parent’s Coordinate Space</h4>

            <p>And now the root view has to put the child somewhere, so it puts in right in the middle.</p>

        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right">
            <img src="{{ site.url }}/images/posts/swiftui-layout/case-01-layout.png">
        </div>
    </div>
</div>

This is it. This is a simple model, but every layout in SwiftUI is calculated this way. This is a major departure from Auto Layout in many important ways.

First, it is in some ways closer to simple frame-based layout where the child had no affect on the parent's frame. This wasn't the case with Auto Layout where constraints work in both directions: in some cases, a parent would determine the size of a child, but sometimes it was the other way around. This was a major source of complexity in Auto Layout and it is gone now.

Second, as you might have noticed, we haven't explicitly said anything about the layout, but there were no "Ambiguous Layout" warnings. Unlike Auto Layout, SwiftUI *always* produces a valid layout. There is no such thing as an ambiguous or an unsatisfiable layout[^2]. The system does its best to always produce the best result and give you the control when needed.

<blockquote class="SwiftUIBlockquote" markdown="1">
**Antialiasing**

One other thing that was also mentioned on [WWDC](https://developer.apple.com/videos/play/wwdc2019/237/) is that at the final step, SwiftUI automatically rounds the edges of your views to the nearest pixels. This is important to note, but, as far as I know, it is no different from Auto Layout.
</blockquote>

Now that we've looked at the most basic example and have an idea of how SwiftUI layout process works, let's see what instruments does it offer. In Auto Layout, all the APIs were built on top of the same technology - [constraints](https://en.wikipedia.org/wiki/Cassowary_(software)). This isn't the case with SwiftUI in which everything: Stacks, Frames, Paddings, etc – is its own thing. To understand the layout system means understanding all of these instruments. Let's start with the most basic one - frames.

### Frame

First, forget everything you know about frames in UIKit or AppKit. Those have nothing to do with [`frame(width:height:alignment:)`](https://developer.apple.com/documentation/swiftui/view/3278572-frame) and other related methods in SwiftUI.

Let's take a 60x60 image and display it using SwiftUI's [`Image`](https://developer.apple.com/documentation/swiftui/image). Look what happens if I set the frame to 80x80.

<div class="SwiftUIExampleWithScreenshot Any-responsiveCard">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left3">
{% highlight swift %}
struct Frame: View {
    var body: some View {
        Image("swiftui")
            .border(Color.red)
            .frame(width: 80, height: 80)
            .border(Color.blue)
    }
}
{% endhighlight %}
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right3">
            <img src="{{ site.url }}/images/posts/swiftui-layout/frame-01.png">
        </div>
    </div>
</div>

The image has not changed its size. Why is that? A frame in SwiftUI is not a constraint. Neither it is the current frame or bounds of the view. Frame in SwiftUI is just another view which you can think of like a picture frame.

By calling `Image("swiftui").frame(width: 80, height: 80)`, SwiftUI creates a new invisible container view with the specified size and positions the image view inside it. The layout process then performs the same steps as we [just described previously]({{ site.url }}/post/swiftui-layout-system#layout-process). The new container view proposes its child, `Image`, the size 80x80. Image view responds that it is only this big – 60x60, but thank you anyway. The `Frame` needs to put the image somewhere, so it puts the image in the center – it uses `.center` alignment by default.

The `alignment` parameter specifies this view’s alignment within the frame. The default one is `.center`, but you can select any of the other [available ones](https://developer.apple.com/documentation/swiftui/alignment):

<div class="SwiftUIExampleWithScreenshot Any-responsiveCard">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left3">
<figure class="highlight"><pre><code class="language-swift" data-lang="swift"><span class="kd">struct</span> <span class="kt">Frame</span><span class="p">:</span> <span class="kt">View</span> <span class="p">{</span>
    <span class="k">var</span> <span class="nv">body</span><span class="p">:</span> <span class="n">some</span> <span class="kt">View</span> <span class="p">{</span>
        <span class="kt">Image</span><span class="p">(</span><span class="s">"swiftui"</span><span class="p">)</span>
            <span class="o">.</span><span class="nf">border</span><span class="p">(</span><span class="kt">Color</span><span class="o">.</span><span class="n">red</span><span class="p">)</span>
            <span class="o">.</span><span class="nf">frame</span><span class="p">(</span><span class="nv">width</span><span class="p">:</span> <span class="mi">80</span><span class="p">,</span> <span class="nv">height</span><span class="p">:</span> <span class="mi">80</span><span class="p">,</span>
                   <span class="SwiftUIPostHighlightedCode"><span class="nv">alignment</span><span class="p">:</span> <span class="o">.</span><span class="nf">topLeading</span><span class="p"></span>)</span>
            <span class="o">.</span><span class="nf">border</span><span class="p">(</span><span class="kt">Color</span><span class="o">.</span><span class="n">blue</span><span class="p">)</span>
    <span class="p">}</span>
<span class="p">}</span></code></pre></figure>

<!-- {% highlight swift %}
struct Frame: View {
    var body: some View {
        Image("swiftui")
            .border(Color.red)
            .frame(width: 80, height: 80,
                   alignment: .topLeading)
            .border(Color.blue)
    }
}
{% endhighlight %} -->
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right3">
            <img src="{{ site.url }}/images/posts/swiftui-layout/frame-02.png">
        </div>
    </div>
</div>

In SwiftUI, unless you mark an image as resizable, either in the asset catalog or in code, it's fixed sized. If marked resizable, frame now directly affects the size of the image view:

<div class="SwiftUIExampleWithScreenshot Any-responsiveCard">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left3">
<figure class="highlight"><pre><code class="language-swift" data-lang="swift"><span class="kd">struct</span> <span class="kt">Frame</span><span class="p">:</span> <span class="kt">View</span> <span class="p">{</span>
    <span class="k">var</span> <span class="nv">body</span><span class="p">:</span> <span class="n">some</span> <span class="kt">View</span> <span class="p">{</span>
        <span class="kt">Image</span><span class="p">(</span><span class="s">"swiftui"</span><span class="p">)</span>
            <span class="SwiftUIPostHighlightedCode"><span class="o">.</span><span class="nf">resizable</span><span class="p">()</span></span>
            <span class="o">.</span><span class="nf">border</span><span class="p">(</span><span class="kt">Color</span><span class="o">.</span><span class="n">red</span><span class="p">)</span>
            <span class="o">.</span><span class="nf">frame</span><span class="p">(</span><span class="nv">width</span><span class="p">:</span> <span class="mi">80</span><span class="p">,</span> <span class="nv">height</span><span class="p">:</span> <span class="mi">80</span><span class="p">)</span>
            <span class="o">.</span><span class="nf">border</span><span class="p">(</span><span class="kt">Color</span><span class="o">.</span><span class="n">blue</span><span class="p">)</span>
    <span class="p">}</span>
<span class="p">}</span></code></pre></figure>

<!-- {% highlight swift %}
struct Frame: View {
    var body: some View {
        Image("swiftui")
            .resizable()
            .border(Color.red)
            .frame(width: 80, height: 80)
            .border(Color.blue)
    }
}
{% endhighlight %} -->
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right3">
            <img src="{{ site.url }}/images/posts/swiftui-layout/frame-03.png">
        </div>
    </div>
</div>

All of the parameters of the [`frame(width:height:alignment:)`](https://developer.apple.com/documentation/swiftui/view/3278572-frame) method are optional. If you only specify one of the dimensions, the resulting view assumes this view’s sizing behavior in the other dimension.

Finally, let's see what happens when the frame is smaller than the size of the content.

<div class="SwiftUIExampleWithScreenshot Any-responsiveCard">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left3">
        <!-- To the left: title, subtitle, etc -->
{% highlight swift %}
struct Frame: View {
    var body: some View {
        Image("swiftui")
            .border(Color.red)
            .frame(width: 40, height: 80)
            .border(Color.blue)
    }
}
{% endhighlight %}
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right3">
            <img src="{{ site.url }}/images/posts/swiftui-layout/frame-04.png">
        </div>
    </div>
</div>

Like any other view, the child ultimately chooses its own size. It is important to understand this property of the SwiftUI layout system.

We used `alignment` to position the child inside the frame. Other instruments can be used to position the child in its parent's coordinate space, like [`position(x:y:)`](https://developer.apple.com/documentation/swiftui/view/3278632-position) and [`offset(x:y:)`](https://developer.apple.com/documentation/swiftui/view/3278608-offset). And [`frame(width:height:alignment:)`](https://developer.apple.com/documentation/swiftui/view/3278572-frame) is not the only way to specify a frame for the view. There is another variation that allows you to specify [minimum, maximum and ideal width and/or height](https://developer.apple.com/documentation/swiftui/view/3278571-frame) which I not cover in this post.

## Stacks

When creating a SwiftUI view, you describe its content in the view’s body property. However, the body property only returns a single view. You can combine[^3] and embed multiple views in stacks[^4].

- [`HStack`](https://developer.apple.com/documentation/swiftui/hstack) group views together horizontally
- [`VStack`](https://developer.apple.com/documentation/swiftui/vstack) – vertically
- [`ZStack`](https://developer.apple.com/documentation/swiftui/zstack) – back to front

<div class="SwiftUITab">
  <button class="SwiftUITabLink active" onclick="openTab(event, 'stack-page-01')">Stack</button>
  <button class="SwiftUITabLink" onclick="openTab(event, 'stack-page-02')">Alignment</button>
  <button class="SwiftUITabLink" onclick="openTab(event, 'stack-page-03')">Spacer</button>
  <button class="SwiftUITabLink" onclick="openTab(event, 'stack-page-04')">Padding</button>
</div>

<div id="stack-page-01" class="SwiftUITabContent SwiftUIExampleWithScreenshot Any-responsiveCard SwiftUIPost_ExampleWithTabs">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left2">
Let's start by adding two <code>Text</code> views to <code>VStack</code>.

{% highlight swift %}
struct ContentView: View {
    var body: some View {
        VStack() {
            Text("Title") 
                .font(.headline)
            Text("Subtitle")
                .font(.subheadline)
        }
    }
}

struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView().previewLayout(
            .fixed(width: 320, height: 70)
        )
    }
}
{% endhighlight %}

By default, stack view uses <code>.center</code> alignment. That's not quite what we want.
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right2">
            <img src="{{ site.url }}/images/posts/swiftui-layout/stack-01.png">
        </div>
    </div>
</div>

<div id="stack-page-02" class="SwiftUITabContent SwiftUIExampleWithScreenshot Any-responsiveCard SwiftUIPost_ExampleWithTabs" style="display:none">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left2">
To change the alignment, pass <code>.leading</code> to the stack view initializer.

<!-- GitHub doesn't support custom plugins -->
<figure class="highlight"><pre><code class="language-swift" data-lang="swift"><span class="kd">struct</span> <span class="kt">ContentView</span><span class="p">:</span> <span class="kt">View</span> <span class="p">{</span>
    <span class="k">var</span> <span class="nv">body</span><span class="p">:</span> <span class="n">some</span> <span class="kt">View</span> <span class="p">{</span>
        <span class="kt">VStack</span><span class="p">(</span><span class="SwiftUIPostHighlightedCode"><span class="nv">alignment</span><span class="p">:</span> <span class="o">.</span><span class="n">leading</span></span><span class="p">)</span> <span class="p">{</span>
            <span class="kt">Text</span><span class="p">(</span><span class="s">"Title"</span><span class="p">)</span>
                <span class="o">.</span><span class="nf">font</span><span class="p">(</span><span class="o">.</span><span class="n">headline</span><span class="p">)</span>
            <span class="kt">Text</span><span class="p">(</span><span class="s">"Subtitle"</span><span class="p">)</span>
                <span class="o">.</span><span class="nf">font</span><span class="p">(</span><span class="o">.</span><span class="n">subheadline</span><span class="p">)</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span></code></pre></figure>

Alright, that worked. But now we need to align the stack itself to the leading edge of the container. How do we do that?
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right2">
            <img src="{{ site.url }}/images/posts/swiftui-layout/stack-02.png">
        </div>
    </div>
</div>


<div id="stack-page-03" class="SwiftUITabContent SwiftUIExampleWithScreenshot Any-responsiveCard SwiftUIPost_ExampleWithTabs" style="display:none">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left2">
In SwiftUI, you use <code>Spacer</code> – a flexible space that expands along the major axis of its containing stack layout – to do that.

<figure class="highlight"><pre><code class="language-swift" data-lang="swift"><span class="kd">import</span> <span class="kt">SwiftUI</span>

<span class="kd">struct</span> <span class="kt">ContentView</span><span class="p">:</span> <span class="kt">View</span> <span class="p">{</span>
    <span class="k">var</span> <span class="nv">body</span><span class="p">:</span> <span class="n">some</span> <span class="kt">View</span> <span class="p">{</span>
        <span class="SwiftUIPostHighlightedCode"><span class="kt">HStack</span> <span class="p">{</span></span>
            <span class="kt">VStack</span><span class="p">(</span><span class="nv">alignment</span><span class="p">:</span> <span class="o">.</span><span class="n">leading</span><span class="p">)</span> <span class="p">{</span>
                <span class="kt">Text</span><span class="p">(</span><span class="s">"Title"</span><span class="p">)</span>
                    <span class="o">.</span><span class="nf">font</span><span class="p">(</span><span class="o">.</span><span class="n">headline</span><span class="p">)</span>
                <span class="kt">Text</span><span class="p">(</span><span class="s">"Subtitle"</span><span class="p">)</span>
                    <span class="o">.</span><span class="nf">font</span><span class="p">(</span><span class="o">.</span><span class="n">subheadline</span><span class="p">)</span>
            <span class="p">}</span>
            <span class="SwiftUIPostHighlightedCode"><span class="kt">Spacer</span><span class="p">()</span></span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span></code></pre></figure>

Spacer fills all the space on the right side of the horizontal stack view.

        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right2">
            <img src="{{ site.url }}/images/posts/swiftui-layout/stack-03.png">
        </div>
    </div>
</div>


<div id="stack-page-04" class="SwiftUITabContent SwiftUIExampleWithScreenshot Any-responsiveCard SwiftUIPost_ExampleWithTabs" style="display:none">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left2">
Now the only thing left to do is to add a bit of padding to text.

<figure class="highlight"><pre><code class="language-swift" data-lang="swift"><span class="kd">import</span> <span class="kt">SwiftUI</span>

<span class="kd">struct</span> <span class="kt">ContentView</span><span class="p">:</span> <span class="kt">View</span> <span class="p">{</span>
    <span class="k">var</span> <span class="nv">body</span><span class="p">:</span> <span class="n">some</span> <span class="kt">View</span> <span class="p">{</span>
        <span class="kt">HStack</span> <span class="p">{</span>
            <span class="kt">VStack</span><span class="p">(</span><span class="nv">alignment</span><span class="p">:</span> <span class="o">.</span><span class="n">leading</span><span class="p">)</span> <span class="p">{</span>
                <span class="kt">Text</span><span class="p">(</span><span class="s">"Title"</span><span class="p">)</span>
                    <span class="o">.</span><span class="nf">font</span><span class="p">(</span><span class="o">.</span><span class="n">headline</span><span class="p">)</span>
                <span class="kt">Text</span><span class="p">(</span><span class="s">"Subtitle"</span><span class="p">)</span>
                    <span class="o">.</span><span class="nf">font</span><span class="p">(</span><span class="o">.</span><span class="n">subheadline</span><span class="p">)</span>
            <span class="p">}</span><span class="SwiftUIPostHighlightedCode"><span class="o">.</span><span class="nf">padding</span><span class="p">()</span></span>
            <span class="kt">Spacer</span><span class="p">()</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span></code></pre></figure>

You can also customize the spacing between items by passing <code>spacing</code> to the initializer.

        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right2">
            <img src="{{ site.url }}/images/posts/swiftui-layout/stack-04.png">
        </div>
    </div>
</div>

Stacks are the primary layout instrument in SwiftUI. The vast majority of layouts can be implemented using stacks. Stacks might seem almost too simple, and they are[^5]. But don't underestimate them. You are going to use stacks *a lot* in SwiftUI. Understanding how they work is probably more important than anything.

### Stack Layout Process

There are three simple steps in the stack layout process.

> **Step 1.** The stack figures out the internal spacing and subtracts it from the size proposed by its parent view.

> **Step 2.** The stack divides the remaining space into *equal parts* for each of the *remaining* views. It then proposes one of those as the size for the *least flexible* child. Whatever size it claimed, it deducts that from the unallocated space. And then it repeats.

> **Step 3.** All children have sizes. The stack lines them up with the spacing and aligns them according to the specified alignment. By default, the alignment is – you guessed it – `.center`. Finally, the stack chooses its own size so that it exactly encloses the children.

The first and the last steps probably don't require any explanation. Step two, on the other hand, might be hard to wrap your head around. I think the best way to understand it is by going through a few examples.

Let's start with a simple example with two image views. The size of each image is 80x80 points, which is *non-negotiable* (use [`resizable`](https://developer.apple.com/documentation/swiftui/image/3269730-resizable) to allow an image to resize). Regardless of what size the stack proposes to any of the images on step two, the image always returns 80x80. The size of the stack itself, with spacing 10, is always going to be 170x80, regardless of the size of its parent.

<div class="SwiftUIExampleWithScreenshot Any-responsiveCard">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left3">
{% highlight swift %}
struct ContentView: View {
    var body: some View {
        HStack(spacing: 10) {
            Image("swiftui")
            Image("swiftui")

        }
    }
}

struct Frame_Previews: PreviewProvider {
    static var previews: some View {
        // Top: 200 x 140
        // Bottom: 140 x 140
    }
}
{% endhighlight %}
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right3">
            <img src="{{ site.url }}/images/posts/swiftui-layout/stack-uth-01.png">
            <img src="{{ site.url }}/images/posts/swiftui-layout/stack-uth-02.png">
        </div>
    </div>
</div>

This first example again shows that in SwiftUI the child ultimately chooses its size. In that regard, SwiftUI layout feels much lighter and more manageable than Auto Layout. There are no layout errors, the stack doesn't arbitrarily resize any of the images. SwiftUI always produces a well-defined result.

Let's now look at another example. This time, let's throw some more flexible views into the mix - `Text` views. The scenario where everything fits is not particularly interesting. The question is what happens if it doesn't?

<div class="SwiftUIExampleWithScreenshot Any-responsiveCard">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left3">
{% highlight swift %}
struct ContentView: View {
    var body: some View {
        HStack(spacing: 10) {
            Image("swiftui")
                .border(Color.red)
            Text("Lorem ipsum dolor sit amet, consectetur adipiscing elit. Praesent et ipsum nulla. In nec nisl nunc. Nulla lectus sem, vulputate non dolor nec, tristique pulvinar felis.")
                .border(Color.green)
            Text("Lorem ipsum dolor sit amet, consectetur adipiscing elit.")
                .border(Color.blue)
        }
    }
}

struct Frame_Previews: PreviewProvider {
    static var previews: some View {
        // Top: 230x120
        // Bottom: 340x180
    }
}
{% endhighlight %}
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right3">
            <img src="{{ site.url }}/images/posts/swiftui-layout/stack-uth-03.png">
            <img src="{{ site.url }}/images/posts/swiftui-layout/stack-uth-04.png">
        </div>
    </div>
</div>


In the first example (top screenshot) the width of the preview is 230. The spacing is 10, so the initial unallocated space is 210 (230 - 10 * 2). So, now the stack needs to calculate the size of its children.

1. The stack splits the space into three equal parts, each of the width 70.
2. It then proposes this size to the least flexible child. In our case, it's `Image`. Its size is 80x80. So that view is out of the picture. The order is irrelevant. If you move the image to another position, the layout stays the same.
3. The stack subtracts the width of the image from the remaining space, so the remaining space is 130 (210 - 80).
4. The stack again splits the space into two equal parts, each of the width 65.
5. It proposes the size 65x120 to the first text view. The text responds that, yeah, the content won't fit, but it can manage to display at least a portion of it. The same happens with the second text view. So even though the text views had a different amount of text, they both got the same width.

In the second example (bottom) we give the stack a little bit more space. So the second text now fits. But because the stack always splits the unallocated space into equal parts, the second text view occupied almost all of the available space.
 
<div class="SwiftUIExampleWithScreenshot Any-responsiveCard">
    <div class="SwiftUIExampleWithScreenshot_Flex">
        <!-- To the left: title, subtitle, etc -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Left3">
<h4>Layout Priority</h4>

<p>You can raise the <a href="https://developer.apple.com/documentation/swiftui/hstack/3269218-layoutpriority">layout priority</a> of views in a stack from the default of zero. A stack offers the children with the highest priority all the space offered to it minus the space required for all its lower-priority children.</p>
        </div>
        <!-- To the right: image -->
        <div class="SwiftUIExampleWithScreenshot_FlexItem SwiftUIExampleWithScreenshot_Right3">
            <img src="{{ site.url }}/images/posts/swiftui-layout/stack-uth-05.png">
        </div>
    </div>
</div>

## Environment

Ok, so frames and stacks are great. But what makes SwiftUI also stand out in terms of supporting adaptive cross-platform apps is the *environment*.

When you add [padding](https://developer.apple.com/documentation/swiftui/image/3269708-padding) to a view, SwiftUI automatically chooses an amount of padding that's appropriate to the platform, dynamic type size, and environment. SwiftUI also automatically sets appropriate spacings between the views in a stack. SwiftUI updates a safe area according to the device. You get the idea.

When you don't pass any parameters, you get adaptive behavior in the same way that SwiftUI adaptively styles a picker or a button depending on the context it's in. And if you to customize any of these parameters, you can do that too.

SwiftUI also makes it easy to react to the changes to the environment. Every view in SwiftUI gets access to the system [environment settings](https://developer.apple.com/documentation/swiftui/environmentvalues) like content size category, device size classes, layout direction, and more. So, for example, if you want to set custom spacings for each size category, you can do that easily with SwiftUI.

By using [@Environment](https://developer.apple.com/documentation/swiftui/environment) property wrapper, you can read the environment values and subscribe to their changes. Technically, the environment is not part of the layout system, but I think it was worth mentioning it.

```swift
struct ContentView: View {
    @Environment(\.sizeCategory) var sizeCategory
    
    var body: some View {
        // This is just an example.
        HStack(spacing: sizeCategory == .large ? 20 : 10) {
            Image("swiftui")
            Text("Lorem ipsum dolor sit amet, consectetur adipiscing elit.")
        }
    }
}
```

## Final Thoughts

> This article is largely based on the fantastic [Building Custom Views with SwiftUI](https://developer.apple.com/videos/play/wwdc2019/237/) WWDC 2019 session. I would highly recommend watching it.

With Auto Layout, Apple took a solution – a layout engine [Cassowary](https://en.wikipedia.org/wiki/Cassowary_(software)), and tried to make it fit the problem – building adaptive user interfaces. It was powerful, but it was lacking in many important areas. It had performance issues, it was complex, debugging it was hard. Apple tried to make it better by introducing more and more Auto Layout APIs over the years: anchors, stack view, safe area. But they never fixed the core problems with the technology.

In Auto Layout one rogue constraint could lead to completely unpredictable results far from the view where it was defined. SwiftUI, on the other hand, is simple and predictable. It should be always possible to understand at a glance why the layout system produces certain results.

You can feel that SwiftUI was created with a completely different mindset than Auto Layout. It is not an academic exercise to efficiently solve systems of linear equalities and inequalities. It is a pragmatic tool designed to solve the real problems that app developers face when creating adaptive cross-platform apps for Apple platforms. And it solves them in a beautiful way by providing a set of small and simple tools which are easy to combine – the [Unix way](https://en.wikipedia.org/wiki/Unix_philosophy)[^6].

Swift dominance didn't come from the server, it just might from the UI.

<div class="References" markdown="1">

## References

1. WWDC 2019, [**Building Custom Views with SwiftUI**](https://developer.apple.com/videos/play/wwdc2019/237/)
2. Apple, [**SwiftUI Tutorials**](https://developer.apple.com/tutorials/swiftui)

<div class="FootnotesSection" markdown="1">

[^1]: Unfortunately, there is no way to [examine the view hierarchy](https://developer.apple.com/library/archive/documentation/ToolsLanguages/Conceptual/Xcode_Overview/ExaminingtheViewHierarchy.html) in Xcode Previews yet. I hope this is something that will be added in the future. For now, Apple recommends you add [borders](https://developer.apple.com/documentation/swiftui/view/3288984-border) to the views for debugging purposes.

[^2]: The [Debugging Auto Layout](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/TypesofErrors.html#//apple_ref/doc/uid/TP40010853-CH17-SW1) section of the Auto Layout guide is probably longer than the rest of the guide. This says something about the system. I bet everyone had their share of moments where solving an Auto Layout problem turned out to be a blind alley.

[^3]: There are not a lot of options for combining multiple views in SwiftUI. You can no longer call `addSubview(_:)` and then layout the contents any way you want – you must use one of the existing container views. And, as far as I can tell, there is currently no way to create custom containers in SwiftUI.

[^4]: It feels that when you talk about stacks, you must mention [Flexbox](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Flexible_Box_Layout/Basic_Concepts_of_Flexbox). This is a technology that supposedly inspired [`UIStackView`](https://developer.apple.com/documentation/uikit/uistackview) and stacks in SwiftUI. I don't know if it's true, but Flexbox is indeed a similar tool. The previews that you see in this post are [powered by Flexbox](https://github.com/kean/kean.github.io). Flexbox is somewhat more powerful than stacks in SwiftUI with support for overflow, etc. However, I find Flexbox to be much more cumbersome to use that stacks in SwiftUI.

[^5]: Stacks in SwiftUI are extremely simple, especially compared to `UIStackView` which has a lot of options, some of which I doubt anyone even uses. One of the examples is `.fillProprtionally` distribution. I've personally never used it. I did [partially implement it]({{ site.url }}/post/lets-build-uistackview#fillproportionally) when building a `UIStackView` replacement. One of the limitations was the fact that `UIStackView` uses a neat private method `_intrinsicContentSize` `invalidatedForChildView` to monitor when one of its children updates its intrinsic content size. How Apple decided that this distribution needed to be built in the first is unclear.

[^6]: This Is The Way.

<!-- All of this stuff is currently specific for this invividual post. I will make it part of the intrastructure later. -->
<script type="text/javascript">
function openTab(evt, tabName) {

    console.log(evt);
  // Declare all variables
  var i, tabcontent, tablinks;

  // Get all elements with class="tabcontent" and hide them
  tabcontent = document.getElementsByClassName("SwiftUITabContent");
  for (i = 0; i < tabcontent.length; i++) {
    tabcontent[i].style.display = "none";
  }

  // Get all elements with class="tablinks" and remove the class "active"
  tablinks = document.getElementsByClassName("SwiftUITabLink");
  console.log(tablinks) 
  for (i = 0; i < tablinks.length; i++) {
    tablinks[i].className = tablinks[i].className.replace(" active", "");
  }
    console.log(tablinks) 

  // Show the current tab, and add an "active" class to the button that opened the tab
  document.getElementById(tabName).style.display = "block";
  evt.currentTarget.className += " active";
}
</script>
