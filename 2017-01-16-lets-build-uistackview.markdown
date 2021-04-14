---
layout: post
title: "Let's Build UIStackView"
description: "Explores how UIStackView works under the hood to build its open source replacement - Arranged"
date: 2017-01-16 10:00:00 +0300
category: programming
tags: ios
permalink: /post/lets-build-uistackview
redirect_from: /blog/lets-build-uistackview
uuid: 89038ab2-a19b-4be3-9a3f-7ab2d58530d6
favorite: true
---

[`UIStackView`](https://developer.apple.com/reference/uikit/uistackview) is a great showcase of Auto Layout. It is built entirely using constraints. Which makes it fairly straightforward to build an [open source replacement](https://github.com/kean/Arranged). Even if you're not in the business of building one, you may find this article useful. It will give you a better understanding of `UIStackView`.

Here is the plan. I will create an instance of `UIStackView`, try different configurations, take screenshots, and print all of the constraints that affect the stack view layout. With this information, implementing a replacement of `UIStackView` should be as easy as translating those constraints into code. In fact, I've already done that in a library named [Arranged](https://github.com/kean/Arranged) (which I've built more than 10 months ago, this follow-up post is a bit late).

{% include ad-hor.html %}

**Prerequisites:**

- [Auto Layout Guide](https://developer.apple.com/library/content/documentation/UserExperience/Conceptual/AutolayoutPG/)
- [UIStackView Guide](https://developer.apple.com/reference/uikit/uistackview)

## Introduction

The `UIStackView` is an abstraction on top of Auto Layout for laying out a collection of views in either a column or a row.

> The stack view manages the layout of all the views in its [arrangedSubviews](https://developer.apple.com/reference/uikit/uistackview/1616232-arrangedsubviews) property. These views are arranged along the stack view’s axis, based on their order in the arrangedSubviews array. The exact layout varies depending on the stack view’s [axis](https://developer.apple.com/reference/uikit/uistackview/1616223-axis), [distribution](https://developer.apple.com/reference/uikit/uistackview/1616233-distribution), [alignment](https://developer.apple.com/reference/uikit/uistackview/1616243-alignment), [spacing](https://developer.apple.com/reference/uikit/uistackview/1616225-spacing), and other properties.

<a name="intrinsic-content-sizes"></a>

In most examples I'm going to use a single `UIStackView` with 3 arranged views with a predefined intrinsic content sizes:

<pre>
<b>Intrinsic Content Size</b>
 Subview0.width == 160 Hug:250 CompressionResistance:750
 Subview0.height == 200 Hug:250 CompressionResistance:750
 Subview1.width == 80 Hug:250 CompressionResistance:750
 Subview1.height == 100 Hug:250 CompressionResistance:750
 Subview2.width == 40 Hug:250 CompressionResistance:750
 Subview2.height == 50 Hug:250 CompressionResistance:750
</pre>

Most of the configurations are going to have a `.horizontal` axis.

## UIStackView.Distribution

The distribution determines how the stack view lays out its arranged views along its axis. Let's start with a default `UIStackView` configuration, which is also one of the easiest to implement:

<a name="UIStackViewDistribution.fill"></a>

### .fill

> A layout where the stack view resizes its arranged views so that they fill the available space along the stack view’s axis. When the arranged views do not fit within the stack view, it shrinks the views according to their compression resistance priority. If the arranged views do not fill the stack view, it stretches the views according to their hugging priority.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/01.png" class="PostStackView_Screenshot">

#### Constraints

Let's print a full list of constraints inside the stack view using this code:

```swift
constraints(for: stackView).forEach { print($0) }

func constraints(for view: UIView) -> [NSLayoutConstraint] {
    var constraints = [NSLayoutConstraint]()
    constraints.append(contentsOf: item.constraints)
    for subview in item.subviews {
        constraints.append(contentsOf: subview.constraints)
    }
    return constraints
}
```

Each constraint gets printed in a full format like this:

```swift
<NSLayoutConstraint:0x6100000922f0 'UISV-canvas-connection'
   UIStackView:0x7f892be06190.leading == Subview0.leading
   (active, names: content-view-1:0x7f892be0bb70 )>
```

As you can see stack view sets a specific `identifier` based on the role that the constraint plays in the layout. Let's convert the remaining constraints to a more concise format:

<pre>
<b>UISV-alignment:</b>
 Subview0.top == Subview1.top
 Subview0.top == Subview2.top
 Subview0.bottom == Subview1.bottom
 Subview0.bottom == Subview2.bottom

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == Subview0.top
 Subview0.bottom == StackView.bottom

<b>UISV-spacing:</b>
 H:[Subview0]-(0)-[Subview1]
 H:[Subview1]-(0)-[Subview2]
</pre>

#### Constraint Categories

The fist two categories of the constraints ensure that the subviews fill the stack view both horizontally (distribution `.fill`) and vertically (alignment `.fill`):

- <b>UISV-alignment</b> constraints align subviews horizontally by making them have equal `.bottom` and equal `.top` layout attributes (`NSLayoutAttribute`). Notice that the stack view pins all the subviews to the first one.
- <b>UISV-canvas-connection</b> constraints connect subviews to the 'canvas' which is a stack view itself. The first and the last subview gets pinned to the `.leading` and `.trailing` layout attributes of the stack view respectively. The first subview also gets pinned to the `.top` and `.bottom` attributes.

<b>UISV-spacing</b> constraints simply add the same spacing between each of the subsequent subviews. The spacing is defined by the value of the stack view `spacing` property.

#### Intrinsic Content Size

As you probably noticed we haven't defined the stack's size along neither of its axis. In this case, the stack view’s size grows freely in both dimensions, based on the its arranged views. So the other remaining constraints that affect the stack view layout are <b>intrinsic content size</b> constraints which I've <a href="#intrinsic-content-sizes">listed earlier</a>. The stack view takes the height of the `Subview0` which is the highest with `Subview0.height == 200 Hug:250 CompressionResistance:750`. The width of the stack view is equal to the combined width of all subviews + all spacings.

In the result we get a very simple but very useful layout. In terms of constraints this configuration is the simplest one to implement.




<a name="UIStackViewDistribution.fillEqually"></a>

### .fillEqually

> A layout where the stack view resizes its arranged views so that they fill the available space along the stack view’s axis. The views are resized so that they are all the same size along the stack view’s axis.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/02.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-fill-equally:</b>
 Subview1.width == Subview0.width
 Subview2.width == Subview0.width

<b>UISV-alignment:</b>
 Subview0.bottom == Subview1.bottom
 Subview0.bottom == Subview2.bottom
 Subview0.top == Subview1.top
 Subview0.top == Subview2.top

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == Subview0.top
 Subview0.bottom == StackView.bottom

<b>UISV-spacing:</b>
 H:[Subview0]-(0)-[Subview1]
 H:[Subview1]-(0)-[Subview2]
</pre>

This distribution is almost the same as <a href="#UIStackViewDistribution.fill">fill distribution</a>. The only different is the new <b>UISV-fill-equally</b> set of constraints. It forces all of the arranged views to have the same width (or height for vertical `UIStackView`).




<a name="UIStackViewDistribution.fillProportionally"></a>

### .fillProportionally

> A layout where the stack view resizes its arranged views so that they fill the available space along the stack view’s axis. Views are resized proportionally based on their intrinsic content size along the stack view’s axis.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/03.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-fill-proportionally:</b>
 Subview0.width == 0.571429*StackView.width priority:999
 Subview1.width == 0.285714*StackView.width priority:998
 Subview2.width == 0.142857*StackView.width priority:997

<b>Hardcoded Width (Custom Constraint):</b>
 StackView.width == 200

<b>UISV-alignment:</b>
 Subview0.bottom == Subview1.bottom
 Subview0.bottom == Subview2.bottom
 Subview0.top == Subview1.top
 Subview0.top == Subview2.top

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == Subview0.top
 Subview0.bottom == StackView.bottom

<b>UISV-spacing:</b>
 H:[Subview0]-(0)-[Subview1]
 H:[Subview1]-(0)-[Subview2]
</pre>

This one is also similar to the <a href="#UIStackViewDistribution.fill">fill distribution</a>, but it requires a new <b>UISV-fill-proportionally</b> category of constraints:

<pre>
<b>UISV-fill-proportionally:</b>
 Subview0.width == 0.571429*StackView.width priority:999
 Subview1.width == 0.285714*StackView.width priority:998
 Subview2.width == 0.142857*StackView.width priority:997
</pre>

The multipliers (0.571429, 0.285714, etc) are calculated as a proportion of an arranged view intrinsic content width to the combined intrinsic content width of all visible arranged views + size of all spacings.

At first this layout seems relatively simple to implement. But there are some potential issues. What happens when not all (or none) of the arranged views have an intrinsic content size? What happens when the intrinsic content size of some of the arranged views changes? Well, as to the last question, `UIStackView` uses a neat private method `_intrinsicContentSizeInvalidatedForChildView` in which it recalculate constraints' multipliers. Unfortunately we can't do that, and there doesn't seem to be any other way to implement this.

Also notice that the priority of the constraints is higher then both content hugging and compression resistance priorities which both effectively get ignored (unless you change those).

> I'm not sure why each constraint has a different priority (999, 998, etc). If you have an idea please <a href="#comment-section">leave a comment below</a>.




<a name="UIStackViewDistribution.equalSpacing"></a>

### .equalSpacing

> A layout where the stack view positions its arranged views so that they fill the available space along the stack view’s axis. When the arranged views do not fill the stack view, it pads the spacing between the views evenly. If the arranged views do not fit within the stack view, it shrinks the views according to their compression resistance priority.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/04.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-distributing-edge:</b>
 H:[Subview0]-(0)-[_UIOLAGapGuide:0x6080001dbc60]
 H:[_UIOLAGapGuide:0x6080001dbc60]-(0)-[Subview1.leading]
 H:[Subview1]-(0)-[_UIOLAGapGuide:0x6080001db8a0]
 H:[_UIOLAGapGuide:0x6080001db8a0]-(0)-[Subview2.leading]

<b>UISV-fill-equally:</b>
 _UIOLAGapGuide:0x6080001db8a0.width == _UIOLAGapGuide:0x6080001dbc60.width

<b>UISV-spacing:</b>
 H:[Subview0]-(>=10)-[Subview1]
 H:[Subview1]-(>=10)-[Subview2]

<b>UISV-alignment:</b>
 Subview0.bottom == Subview1.bottom
 Subview0.bottom == Subview2.bottom
 Subview0.top == Subview1.top
 Subview0.top == Subview2.top

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == Subview0.top
 Subview0.bottom == StackView.bottom
</pre>

This configuration requires extra spacers (`_UIOLAGapGuide`) [between subsequent subviews](http://stackoverflow.com/questions/38778692/xcode-7-constraints-equal-spacing-between-buttons). Spacers are pinned to respected subviews by <b>UISV-distributing-edge</b> constraints. All the spacers have the same size thanks to <b>UISV-fill-equally</b> constraints. The <b>UISV-spacing</b> constraints are a bit different from previous distributions too - they now use `NSLayoutRelation.greaterThanOrEqual` rather than `NSLayoutRelation.equal`.




<a name="UIStackViewDistribution.equalCentering"></a>

### .equalCentering

> A layout that attempts to position the arranged views so that they have an equal center-to-center spacing along the stack view’s axis, while maintaining the spacing property’s distance between views. If the arranged views do not fit within the stack view, it shrinks the spacing until it reaches the minimum spacing defined by its spacing property. If the views still do not fit, the stack view shrinks the arranged views according to their compression resistance priority.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/05.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-distributing-edge:</b>
 _UIOLAGapGuide:0x6000001dc5c0.leading == Subview0.centerX
 _UIOLAGapGuide:0x6000001dc5c0.trailing == Subview1.centerX
 _UIOLAGapGuide:0x6000001dc2f0.leading == Subview1.centerX
 _UIOLAGapGuide:0x6000001dc2f0.trailing == Subview2.centerX

<b>UISV-fill-equally:</b>
 _UIOLAGapGuide:0x6000001dc2f0.width ==_UIOLAGapGuide:0x6000001dc5c0.width priority:149

<b>UISV-spacing:</b>
 H:[Subview0]-(>=0)-[Subview1]
 H:[Subview1]-(>=0)-[Subview2]

<b>UISV-alignment:</b>
 Subview0.bottom == Subview1.bottom
 Subview0.bottom == Subview2.bottom
 Subview0.top == Subview1.top
 Subview0.top == Subview2.top

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == Subview0.top
 Subview0.bottom == StackView.bottom
</pre>

Equal centering distribution is similar to the [.EqualSpacing](#UIStackViewDistribution.equalSpacing) because it also uses spacers (`_UIOLAGapGuide`) between subsequent views. The main difference is how those spacers are used. Instead of pinning spacers to the respective `.leading` and `.trailing` layout attributes of the views they are now pinned to the `.centerX`.

The other difference is that **UISV-fill-equally** constraints now have a low priority of 149, which is lower than the default compression resistance, which ensures that:

> If the arranged views do not fit within the stack view, it shrinks the spacing until it reaches the minimum spacing defined by its spacing property. If the views still do not fit, the stack view shrinks the arranged views according to their compression resistance priority.

The minimum spacing is again achieved by **UISV-spacing** constraints which all have a *required* priority.



## UIStackView.Alignment


<a name="UIStackViewAlignment.fill"></a>

### .fill

> A layout where the stack view resizes its arranged views so that they fill the available space perpendicular to the stack view’s axis.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/01.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-alignment:</b>
 Subview0.bottom == Subview1.bottom
 Subview0.bottom == Subview2.bottom
 Subview0.top == Subview1.top
 Subview0.top == Subview2.top

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == Subview0.top
 Subview0.bottom == StackView.bottom

<b>UISV-spacing:</b>
 H:[Subview0]-(0)-[Subview1]
 H:[Subview1]-(0)-[Subview2]
</pre>

This configuration is exactly the same as the very first one: [fill distribution](#UIStackViewDistribution.fill).




<a name="UIStackViewAlignment.leading"></a>

### .leading

> A layout for vertical stacks where the stack view aligns the leading edge of its arranged views along its leading edge.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/06.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-alignment:</b>
 Subview0.top == Subview1.top
 Subview0.top == Subview2.top

<b>UISV-spanning-boundary:</b>
 _UILayoutSpacer.top == Subview0.top priority:999.5
 _UILayoutSpacer.bottom >= Subview0.bottom
 _UILayoutSpacer.top == Subview1.top priority:999.5
 _UILayoutSpacer.bottom >= Subview1.bottom
 _UILayoutSpacer.top == Subview2.top priority:999.5
 _UILayoutSpacer.bottom >= Subview2.bottom

<b>UISV-spanning-fit:</b>
 _UILayoutSpacer.height == 0 priority:51

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == Subview0.top
 _UILayoutSpacer.bottom == StackView.bottom

<b>UISV-ambiguity-suppression:</b>
 Subview0.height == 0 priority:25
 Subview1.height == 0 priority:25
 Subview2.height == 0 priority:25

<b>UISV-spacing:</b>
 H:[Subview0]-(0)-[Subview1]
 H:[Subview1]-(0)-[Subview2]
</pre>

That's a lot of constraints compared to the previous configurations! Let's figure out what's going on here.

Stack views adds a new auxiliary `_UILayoutSpacer` view to which it pins all of the arranged views with **UISV-spanning-boundary** constraints. Notice that all `.bottom` constraints use `.greaterThanOrEqual` relation.

Spacer's height is bounded by **UISV-spanning-fit** constraint to disambiguate its height. The height of the arranged views also gets disambiguated automatically by stack view for you (**UISV-ambiguity-suppression** constraints).

Interestingly vertical **UISV-canvas-connection** constraints pin different views to the canvas this time. The first arranged view gets pinned to the top, while the spacer (`_UILayoutSpacer`) gets pinned to the bottom.

Now why is that so complicated? Why is layout spacer necessary and is it necessary at all? Couldn't we just pin all the arranged views to the stack view itself? I think that it might be excessive, but I might be missing something. If you have an idea why it's implemented this way please [share it in the comments](#comment-section).




<a name="UIStackViewAlignment.center"></a>

### .center

> A layout where the stack view aligns the center of its arranged views with its center along its axis.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/07.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-alignment:</b>
 Subview0.centerY == Subview1.centerY
 Subview0.centerY == Subview2.centerY

<b>UISV-spanning-boundary:</b>
 _UILayoutSpacer.bottom >= Subview0.bottom
 _UILayoutSpacer.bottom >= Subview1.bottom
 _UILayoutSpacer.bottom >= Subview2.bottom
 _UILayoutSpacer.top <= Subview0.top
 _UILayoutSpacer.top <= Subview1.top
 _UILayoutSpacer.top <= Subview2.top

<b>UISV-spanning-fit:</b>
 _UILayoutSpacer.height == 0 priority:51

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == _UILayoutSpacer.top
 _UILayoutSpacer.bottom == StackView.bottom
 StackView.centerY == Subview0.centerY

<b>UISV-ambiguity-suppression:</b>
 Subview0.height == 0 priority:25
 Subview1.height == 0 priority:25
 Subview2.height == 0 priority:25

<b>UISV-spacing:</b>
 H:[Subview0]-(0)-[Subview1]
 H:[Subview1]-(0)-[Subview2]
</pre>

This and the following alignment (.trailing) is very similar to a [leading alignment](#UIStackViewAlignment.leading) so I'm not going to comment them that much. This particular configuration has an extra **UISV-canvas-connection** constraint that pins first arranged view to the `.centerY` of the stack view, and has slightly different **UISV-spanning-boundary** and **UISV-alignment**. But the idea is the same.




<a name="UIStackViewAlignment.trailing"></a>

### .trailing

> A layout for vertical stacks where the stack view aligns the trailing edge of its arranged views along its trailing edge.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/08.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-alignment:</b>
 Subview0.bottom == Subview1.bottom
 Subview0.bottom == Subview2.bottom

<b>UISV-spanning-boundary:</b>
 _UILayoutSpacer.top <= Subview0.top
 _UILayoutSpacer.top <= Subview1.top
 _UILayoutSpacer.top <= Subview2.top
 _UILayoutSpacer.bottom == Subview0.bottom priority:999.5
 _UILayoutSpacer.bottom == Subview1.bottom priority:999.5
 _UILayoutSpacer.bottom == Subview2.bottom priority:999.5

<b>UISV-spanning-fit:</b>
 _UILayoutSpacer.height == 0 priority:51

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == _UILayoutSpacer.top
 Subview0.bottom == StackView.bottom

<b>UISV-ambiguity-suppression:</b>
 Subview0.height == 0 priority:25
 Subview1.height == 0 priority:25
 Subview2.height == 0 priority:25

<b>UISV-spacing:</b>
 H:[Subview0]-(0)-[Subview1]
 H:[Subview1]-(0)-[Subview2]
</pre>



<a name="UIStackViewAlignment.firstBaseline"></a>

### .firstBaseline

> A layout where the stack view aligns its arranged views based on their first baseline. This alignment is only valid for horizontal stacks.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/09.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-alignment:</b>
 Subview0.firstBaseline == Subview1.firstBaseline
 Subview0.firstBaseline == Subview2.firstBaseline

<b>UISV-spanning-boundary:</b>
 _UILayoutSpacer.top <= Subview0.top
 _UILayoutSpacer.bottom >= Subview0.bottom
 _UILayoutSpacer.top <= Subview1.top
 _UILayoutSpacer.bottom >= Subview1.bottom
 _UILayoutSpacer.top <= Subview2.top
 _UILayoutSpacer.bottom >= Subview2.bottom

<b>UISV-spanning-fit:</b>
 _UILayoutSpacer.height == 0 priority:51

<b>UISV-canvas-fit:</b>
 StackView.height == 0 priority:49

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == _UILayoutSpacer.top
 _UILayoutSpacer.bottom == StackView.bottom

<b>UISV-text-width-disambiguation:</b>
 Subview0.width == 0.333333*StackView.width priority:760
 Subview1.width == 0.333333*StackView.width priority:760
 Subview2.width == 0.333333*StackView.width priority:760

<b>UISV-ambiguity-suppression:</b>
 Subview0.height == 0 priority:25
 Subview1.height == 0 priority:25
 Subview2.height == 0 priority:25

<b>UISV-spacing:</b>
 H:[Subview0]-(0)-[Subview1]
 H:[Subview1]-(0)-[Subview2]
</pre>



The `.firstBaseline` and the `.lastBaseline` alignments only make sense for views like `UILabel` which has some content with actual baselines (like text).

The set of constraints is similar to the previous scenario: it uses an auxiliary spacer, connects the spacer to the stack view, etc. But there is one discrepancy – **UISV-text-width-disambiguation** constraints. Without these constraints, the layout would be ambiguous – the layout system won't be able to decide which text container to prioritize. I would say that this is a user error, and it looks like Apple decided to automatically correct it to produce a layout that's not completely broken. But there is no trace of this behavior neither in the [UIStackView documentation](https://developer.apple.com/reference/uikit/uistackview) not in headers, and because of that, I decided not to implement these constraints in [Arranged](https://github.com/kean/Arranged).

<a name="UIStackViewAlignment.lastBaseline"></a>

### .lastBaseline

> A layout where the stack view aligns its arranged views based on their last baseline. This alignment is only valid for horizontal stacks.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/10.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-alignment:</b>
 Subview0.lastBaseline == Subview1.lastBaseline
 Subview0.lastBaseline == Subview2.lastBaseline

<b>UISV-spanning-boundary:</b>
 _UILayoutSpacer.top <= Subview0.top
 _UILayoutSpacer.bottom >= Subview0.bottom
 _UILayoutSpacer.top <= Subview1.top
 _UILayoutSpacer.bottom >= Subview1.bottom
 _UILayoutSpacer.top <= Subview2.top
 _UILayoutSpacer.bottom >= Subview2.bottom

<b>UISV-spanning-fit:</b>
 _UILayoutSpacer.height == 0 priority:51

<b>UISV-canvas-fit:</b>
 StackView.height == 0 priority:49

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == _UILayoutSpacer.top
 V:[Subview0]-(>=0)-|

<b>UISV-text-width-disambiguation:</b>
 Subview0.width == 0.333333*StackView.width priority:760
 Subview1.width == 0.333333*StackView.width priority:760
 Subview2.width == 0.333333*StackView.width priority:760

<b>UISV-ambiguity-suppression:</b>
 Subview0.height == 0 priority:25
 Subview1.height == 0 priority:25
 Subview2.height == 0 priority:25

<b>UISV-spacing:</b>
 H:[Subview0]-(0)-[Subview1]
 H:[Subview1]-(0)-[Subview2]
</pre>




## Misc


<a name="Misc.HidingSubviews"></a>

### Hiding Subviews

> The stack view automatically updates its layout whenever views are added, removed or inserted into the arrangedSubviews array, or whenever one of the arranged views's isHidden property changes.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/11.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-hiding:</b>
 Subview0.width == 0

<b>UISV-spacing:</b>
 H:[Subview0]-(0)-[Subview1]
 H:[Subview1]-(10)-[Subview2]

<b>UISV-alignment:</b>
 Subview0.bottom == Subview1.bottom
 Subview0.bottom == Subview2.bottom
 Subview0.top == Subview1.top
 Subview0.top == Subview2.top

<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 Subview2.trailing == StackView.trailing
 StackView.top == Subview0.top
 Subview0.bottom == StackView.bottom
</pre>

This one seems easy at first but actually is quite tricky. Hiding subviews is quite simple, `UIStackView` just adds a single **UISV-hiding** constraint and recalculates the spacings. What's more complicated is how `UIStackView` reacts to `isHidden` changes and how it perform animations.

The `UIStackView` promises to do everything automagically for you:

```swift
// Animates removing the first item in the stack.
UIView.animateWithDuration(0.25) { () -> Void in
    let firstView = stackView.arrangedSubviews[0]
    firstView.hidden = true
}
```

What happens here is `UIStackView` observes `isHidden` property of the arranged views, cancels its effect, updates constraints, and automatically calls `layoutIfNeeded()` at the end of the animation block. I'm a little concerned about what kind of trickery with private APIs is used by `UIStackView` to make this work. I made a couple of attempts to make this work but ultimately decided that this behavior was both confusing and impractical to implement. I went with a simple fully controlled API:

```swift
UIView.animateWithDuration(0.33) {
    stackView.setArrangedView(view, hidden: true)
    stackView.layoutIfNeeded()
}
```

This way some of the responsibilities gets delegated to the user of the library, but all the complexities of working with `isHidden` are gone.




<a name="Misc.MarginsRelativeLayout"></a>

### Margins Relative Layout

> A Boolean value that determines whether the stack view lays out its arranged views relative to its layout margins. If true, the stack view will layout its arranged views relative to its layout margins. If false, it lays out the arranged views relative to its bounds.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/12.png" class="PostStackView_Screenshot">

<pre>
UISV-alignment:
 Subview0.bottom == Subview1.bottom
 Subview0.bottom == Subview2.bottom
 Subview0.top == Subview1.top
 Subview0.top == Subview2.top

<b>UISV-canvas-connection:</b>
 UIViewLayoutMarginsGuide.leading == Subview0.leading
 UIViewLayoutMarginsGuide.trailing == Subview2.trailing
 UIViewLayoutMarginsGuide.top == Subview0.top
 UIViewLayoutMarginsGuide.bottom == Subview0.bottom

<b>UIView-margin-guide-constraint:</b>
 V:[UIViewLayoutMarginsGuide]-(8)-|
 H:|-(8)-[UIViewLayoutMarginsGuide]
 H:[UIViewLayoutMarginsGuide]-(8)-|
 |-(8)-[UIViewLayoutMarginsGuide]

<b>UISV-spacing:</b>
 H:[Subview0]-(0)-[Subview1]
 H:[Subview1]-(0)-[Subview2]
</pre>

**UISV-canvas-connection** constraints now pin subviews to `UIViewLayoutMarginsGuide` instead of bounds.




<a name="Misc.BaselineRelativeLayout"></a>

### Baseline Relative Layout

> A Boolean value that determines whether the vertical spacing between views is measured from their baselines. If YES, the vertical space between views are measured from the last baseline of a text-based view, to the first baseline of the view below it. Top and bottom views are also positioned so that their closest baseline is the specified distance away from the stack view’s edge. This property is only used by vertical stack views.

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/13.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-spacing:</b>
 Subview1.firstBaseline == Subview0.lastBaseline + 20
 Subview2.firstBaseline == Subview1.lastBaseline + 20

<b>UISV-alignment:</b>
 Subview0.leading == Subview1.leading
 Subview0.leading == Subview2.leading
 Subview0.trailing == Subview1.trailing
 Subview0.trailing == Subview2.trailing

<b>UISV-canvas-connection:</b>
 StackView.top == Subview0.top
 Subview2.bottom == StackView.bottom
 StackView.leading == Subview0.leading
 Subview0.trailing == StackView.trailing
</pre>

The stack modifies **UISV-spacing** constraints by using `.firstBaseline` and `.lastBaseline` layout attributes.




<a name="Misc.SingleSubviewCenterAlignment"></a>

### Single Subview, Alignment.Center

<img alt="Stack view screenshot" src="{{ site.url }}/images/stack_view/14.png" class="PostStackView_Screenshot">

<pre>
<b>UISV-canvas-connection:</b>
 StackView.leading == Subview0.leading
 H:[Subview0]-(0)-|
 StackView.top <= Subview0.top
 V:[Subview0]-(>=0)-|
 StackView.centerY == Subview0.centerY

<b>UISV-canvas-fit:</b>
 StackView.height == 0 priority:49

<b>UISV-ambiguity-suppression:</b>
 Subview0.height == 0 priority:25
</pre>

Compared to [center alignment](#UIStackViewAlignment.center) with multiple subviews this configuration optimizes constraints by removing auxiliary spacer (`_UILayoutSpacer`) which is not necessary when there is just single arranged view.


## Implementation

I'm not going to dive into too much detail as far as actual the implementation goes. You've probably already noticed that most of the configurations have something in common. All of the constraints are split into well-defined categories and follow specific rules. The only thing that remains is to translate those categories and rules into code (actually it was a bit challenging given the number of options and different scenarios).

If you'd like to check out the actual code please take a look at [Arranged](https://github.com/kean/Arranged) sources. The entire implementation takes about 500 lines of code, most of which aren't even constraint related.

### Performance

`UIStackView` has a bunch of optimizations under the hood. One thing that it does for certain is updating only the constraints that need to be changed when some of stack view options change (like, say, distribution). While this feature might be welcomed when the layout changes dynamically, I've decided to avoid those optimizations to keep code as simple and reliable as possible. Even `UIStackView` itself has a number of scenarios in which it fails to update all of the constraints when configuraiton changes

### Testing

Testing [Arranged](https://github.com/kean/Arranged) was relatively simple. All I had to do was test that `Arranged.StackView` creates a set of constraints equivalent to the reference (`UIStackView`) in all of those cases.


<a name="bottom"></a>
