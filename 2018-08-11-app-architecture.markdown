---
layout: post
title: "Let's Talk Architecture"
description: "MVC – The Past. MVVM with RxSwift or ReactiveCocoa, MVP – The Present. Functional Architectures – The Future."
date: 2018-08-11 18:00:00 +0300
category: programming
tags: programming
permalink: /post/app-architecture
uuid: e0b6da16-7dc6-49d0-8ea5-e68c3500b7d2
---

> <b>MVC</b> – The Past. <b>MVVM with RxSwift or ReactiveCocoa, MVP</b> – The Present. <b>Functional Architectures</b> – The Future.

From the day when iOS SDK was introduced, we've seen many changes in the way we develop software for iPhones. One of the hot topics is always app architecture. We've seen quite a few over the years. I think it's a good time to put them in some context and try to see what the future of iOS development might look like.

{% include ad-hor.html %}

## The Past

Let's go back to the biblical times, the year 2008. A guy named Scott Forstall goes on stage in Apple Town Hall, Cupertino and [unveils iOS SDK](https://www.youtube.com/watch?v=2dODtKBl8L4) to the world. One of the public classes introduced is `UIViewController` and the official app architecture to be used is MVC.

### MVC

MVC is the term that has been around for [more than 40 years](http://heim.ifi.uio.no/~trygver/themes/mvc/mvc-index.html). It was one of the first attempts to create an architecture that would allow developing increasingly more complex GUIs – I wasn't around back then, but I do read [Martin Fowler](https://www.martinfowler.com/eaaDev/uiArchs.html).

Apple's [infamous interpretation of MVC](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html) is different from the classic one. In the original, the View is active and reads data directly from the Model. The diagram that Apple has in their documentation looks a lot more like MVP than MVC.

Apple also provides a class named `UIViewController` which does ... [many things](https://github.com/nst/iOS-Runtime-Headers/blob/master/Frameworks/UIKit.framework/UIViewController.h). When you have just taken up iOS development, it's easy to fall into the trap and start treating the words `Controller` and `UIViewController` as synonyms. If you start putting most of the View and Controller logic in your `UIViewController` subclasses, you end up with a *two-tier architecture* where all you have is a View and a Model – `UIViewController` is very closely tied to a View and is often considered to be part of it.

A two-tier architecture is neither good nor bad, it's just a trade-off to be made. It has many advantages: it has a lower entry point, it's often easier to read and understand, and it's faster to write. But it isn't without problems: reusing components and testing them becomes awkward. It doesn't mean that you can't ship perfectly good apps with it, or that you can't write tests – you most definitely can. There is nothing preventing you from testing classes in your Model layer – this is what I would consider the first priority for unit test coverage anyway.

Back in Objective-C days, when pretty much all iOS developers used some form of MVC, we would still often incorporate approaches from other architectures. For example, it was a common practice to have an equivalent of ViewModels written using KVO – these bad boys were just as good if not better than the ones based on reactive frameworks today.

As far as I remember, Apple never told us that `UIViewController` was the only Controller you would ever need. In fact, Apple ships a bunch of [controllers](https://developer.apple.com/documentation/mediaplayer/mpmusicplayercontroller) that are just [plain NSObjects](https://developer.apple.com/documentation/appkit/nsarraycontroller). Many developers understood that and never had too many problems with MVC.

## The Present

Fast forward to 2015. iOS development has matured, people are doing CI, TDD, BDD and are using all sorts tools and practices – many of them are often not even mentioned once in Apple's guidelines. The developer community thinks that Apple is doing them a disservice by not updating any of their app architecture recommendations and starts looking outside of the Apple ecosystem for solutions. An infamous term [Massive View Controller](https://www.objc.io/issues/13-architecture/mvvm/) is born.

May 2015 – [RxSwift 1.0 is released](https://github.com/ReactiveX/RxSwift/releases/tag/1.0) taking the spotlight from ReactiveCocoa.

### MVVM

MVVM was one of the first "alternative" patterns that were wildly adopted by iOS developers. Every newsletter in 2015 would have at least one article about MVVM and RxSwift or ReactiveCocoa.

This pattern was [originally introduced](https://blogs.msdn.microsoft.com/johngossman/2005/10/08/introduction-to-modelviewviewmodel-pattern-for-building-wpf-apps/) by Microsoft under the name MVVM back in 2005, but most of the ideas behind it were shared earilier by Martin Fowler in his article named [Presentation Model](https://martinfowler.com/eaaDev/PresentationModel.html) from 2004.

MVVM is a three-tier architecture. You have View which is active, ViewModel and Model. An _active_ View means that the View knows about the ViewModel and that it itself binds to the output of the ViewModel.

Many people liked this pattern, some thought [it was "exceptionally OK"](https://ashfurrow.com/blog/mvvm-is-exceptionally-ok/) and some were complaining about [its poor naming](http://khanlou.com/2015/12/mvvm-is-not-very-good/).

I used MVVM+RxSwift in one of my projects at work and had spent more than a year with it. Back then, both of these technologies were new to me, and I ended up quite liking them.

The great thing about MVVM (and MVP, but more on it later) is that it was the first pattern where I was able to finally embrace unit testing. Now, I was writing unit tests not just for a Model layer, but for most of my ViewModels as well. What I like most about MVVM is that you don't need a View to test a ViewModel. All you need is to inject a Model test double, trigger some events and [record the outputs](https://kean.github.io/post/rxswift-testing) of the ViewModel. This stuff is fantastic.

What I didn't like much was – surprisingly to me – bindings. It's really tempting to use bindings because of how nice they seem on the surface, but they have a few major problems:

- They aren't very useful. In most cases, bindings are too granular. You often want to bind the entire output of a ViewModel to a View instead, not each property individually. This means that in most cases you end up writing your own custom binding targets or just not using bindings at all.
- It's hard to do animations.
- They are hard to debug when things go wrong – can't put breakpoints.

I also quite liked RxSwift at first, but what I found not very helpful was this relentless idea in RxSwift community that everything should be a [pure transformation of input sequences to output sequences](https://github.com/ReactiveX/RxSwift/blob/master/RxExample/RxExample/Examples/GitHubSignup/UsingDriver/GithubSignupViewModel2.swift). It looks great in demos, but it doesn't always work in practice. For example, most of the code samples require you to create a ViewModel when you already have an instance of a view. This is not how you use MVVM in practice. What you do is create a ViewModel first and then at some point later bind it to a view, this is especially true for table cells.

RxSwift fits perfectly in a Model layer, but it is [too low-level](http://elm-lang.org/blog/farewell-to-frp) and too awkward to use in the UI. There are some attempts, like [RxFeedback](https://github.com/NoTests/RxFeedback.swift), that try to prove that not everything is lost and that it doesn't have to be painful to write UIs using RxSwift, but to me personally, they seem more like a bunch of clever tricks than a holistic architecture approach.

> It would be unfair not to mention RxSwift's fantastic [traits system](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Traits.md) which is an attempt to add semantics to observable sequences by introducing higher level concepts like `Single`, `Driver`, `ControlEvent`, `ControlProperty` etc. I think systems written using them are much easier to read and understand than the ones based solely on raw ReactiveCocoa `Signal` and `SignalProducer` types – ReactiveCocoa doesn't have the high-level concepts equivalent to traits.

I tried hard to make myself think of button taps as sequences of `Void` values, but it just never clicked for me.

### MVP

MVP is a de-facto pattern for many Android developers, but it's not very popular in the iOS community – or at least it doesn't seem to be.

MVP appeared [during the 90's](http://www.wildcrest.com/Potel/Portfolio/mvp.pdf), so it's a bit older than MVVM.

MVP is also a three-tier architecture. You have View which is passive, Presenter and Model. A _passive_ View means that unlike MVVM it doesn't update itself, but rather it waits for Presenter to do so.

MVP is as testable as MVVM is with the only difference that you have to write tons of View mocks. Fortunately, you can easily automate that using [Sourcery](https://github.com/krzysztofzablocki/Sourcery).

Another major difference is that MVP is arguably much easier to wrap your head around than MVVM. Just like with Apple's MVC, you get a direct access to a view from a presenter, which means that you can easily do animations and all sorts of other things.

I hope to see MVP being used more in the future. It's testable, it's easy to understand and in my view, it's very under-appreciated in the iOS developer community.

## The Future

Present, 2018. There is an architecture pattern [for every combination of three letters](https://iosarchitecture.top). Functional-ish programming is mainstream. But to be serious, we seem to be on the break of the next wave of new architecture patterns – the ones that embrace immutability and functional programming.

### The Elm Architecture

[The Elm Architecture](https://guide.elm-lang.org/architecture/) and the ones inspired by it push forward the ideas of functional and declarative programming in the UI layer. It's an architecture used with Elm programming language to develop web apps, but there are many attempts to adopt it or its derivatives to the Apple platforms.

If you'd like to see how it could look like in Swift, check out [samples](https://github.com/objcio/app-architecture) from [App Architecture](https://www.objc.io/books/app-architecture/) book and [tea-in-swift](https://github.com/chriseidhof/tea-in-swift) repo made by Chris Eidhof.

There is Model – the state of the app, Update – messages to update the state, and View. The model reacts to the user actions by updating its state and producing a new view. A view is a function from a model – this is the functional part. On every new state, you produce a new instance of a view.

To make this all run smoothly, you don’t actually produce any of the real instances of UIView or UIViewController subclasses when creating a view. What you do instead is create their virtual representations. The framework then automatically figures out what has changed and updates the UI accordingly. React.js is also based on the same principle. The so-called ["Reconciliation"](https://reactjs.org/docs/reconciliation.html) operates on a virtual DOM and uses a diffing algorithm to figure out what to change in the real DOM.

Now, why is this all so great? First, you have a simple and reliable way to update the UI where you no longer need to "manually" mutate each individual control – you write a pure function instead. If you like functional programming, you’ll love it too. Second, you get all sorts of cool features, like [time traveling debuggers](http://debug.elm-lang.org).

There are some open questions, at least in my head. For instance, if all I have as an output from the Model is an instance of a virtual representation, then how do I test it? It contains too much information that I don't necessarily need, like the order of controls. I could use techniques similar to the ones found in UI automation testing to write test expectations, but it doesn't seem optimal. I don't mind being proven wrong, but that's one thing that comes to mind. It's clear that snapshot testing would become much more fun. If you trust your framework that implements virtual views, you're no longer going to need screenshot to do snapshot testing, a virtual representation can be your snapshot.

If functional programming has any future in UI development, this is it. It doesn’t have to be Elm or React Native or some other specific framework – they are all based on the same principles. We’ve already tried using functional reactive programming in UI, but it ended up being too low-level and too cumbersome to use. We need a simple solution that anyone can understand and use.

If you want to try something close to the Elm Architecture today, but without the need for a virtual view implementation, I would recommend looking into immutable ViewModels. They feel very similar, but don't require a ton of infrastructure to make them play nicely with UIKit.

## Conclusion

App architecture is a topic that might seem intimidating. There are so many acronyms which stand for things that seem so abstract. But don’t get discouraged. Most of the architectures are actually very similar and are all there to help you achieve the same thing – write great software.
