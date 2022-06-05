---
layout: post
title: "Three Use Cases of Phantom Types"
description: "Adding additional type safety using phantom types on the examples"
date: 2017-11-24 18:00:00 +0300
category: programming
tags: ios
permalink: /post/phantom-types
uuid: f6bfea67-880f-462f-a08c-5b065a70f573
---

Phantom types are used to parameterize generic types but never actually appear in their implementation. Why are they useful? For additional type safety.

With phantom types, you can add extra information to your types and use it to restrict the code to make invalid situations impossible. Having more restrictions means less room for programming error, better, self-documenting code, and fewer tests. In practice, those restrictions can be quite complex which I would demonstrate on the three examples from my most recent projects:

- [Id Type](#id-type)
- [Authentication Scopes](#authentication-scopes)
- [Layout Anchors](#layout-anchors)

{% include ad-hor.html %}

## Id Type

The `Id` type represents an identifier of a model entity. It gets parametrized with an `Entity` which is used by the compiler when comparing different types of `Id`s. This way the compiler prevents you from accidentally mixing up different ids. Essentially each entity has its own type of id (e.g. `Id<User>`, `Id<Image>`, etc).

```swift
struct Id<Entity>: Hashable {
    let raw: String
        
    var hashValue: Int {
        return raw.hashValue
    }
    
    static func ==(lhs: Id, rhs: Id) -> Bool {
        return lhs.raw == rhs.raw
    }
}
```

Usage:

```swift
final class Activity: Swift.Decodable {
    let id: Id<Activity>
    let name: String
}

extension API.Activity {
    static func get(activityId: Id<Activity>) -> Endpoint<Activity>
}

```

> See [**Codable: Id Type and Single Value Container**](https://kean.github.io/post/codable-tips-and-tricks#2-id-type-and-a-single-value-container) to learn how to add `Codable` conformance to `Id` type.

## Authentication Scopes

Another use case is the <a href="{{ site.url }}/post/api-client">API Client</a> where phantom types represent authentication scopes:

```swift
enum Scope { // enums used as namespaces
    enum Guest {}
    enum Customer {}
}

struct AuthorizedEndpoint<Authorization, Response> {
    let raw: Endpoint<Response>
}

struct AuthorizedClient<Authorization> {
    let raw: ClientProtocol
}
```

Extensions with [generic where clauses](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Generics.html#//apple_ref/doc/uid/TP40014097-CH26-ID553) are used to represent permissions in `AuthorizedClient`:

```swift
/// API client with a `Guest` authorization can only perform requests
/// with a lowest (`Guest`) authorization scope.
extension AuthorizedClient where Authorization == Scope.Guest {
    func request<Response>(_ endpoint: AuthorizedEndpoint<Scope.Guest, Response>) -> Single<Response> {
        return raw.request(endpoint.raw)
    }
}

/// API client with a `Customer` authorization can perform requests
/// with both `Guest` and `Customer` authorization scopes.
extension AuthorizedClient where Authorization == Scope.Customer {
    func request<Response>(_ endpoint: AuthorizedEndpoint<Scope.Guest, Response>) -> Single<Response> {
        return raw.request(endpoint.raw)
    }

    func request<Response>(_ endpoint: AuthorizedEndpoint<Scope.Customer, Response>) -> Single<Response> {
        return raw.request(endpoint.raw)
    }
}
```

> For more info about the API client see <a href="{{ site.url }}/post/api-client">**API Client in Swift**</a>.


## Layout Anchors

The most recent and the most complex use case of phantom types for me was in [Align](https://github.com/kean/Align), a small Auto Layout library which implements custom layout anchors. I think this is most interesting one, so I'm going to focus on it a bit more. Similar to [`NSLayoutAnchor`](https://developer.apple.com/documentation/uikit/nslayoutanchor) each anchor in Align represents a layout attribute of a view. Unlike `NSLayoutAnchor` each kind of anchor has its own special set of methods which is part of Align's fluent API.

First, let's take a quick look at how `NSLayoutAnchor` represents different kinds of anchors. There is a base class `NSLayoutAnchor` and there are three subclasses:

> You never use the `NSLayoutAnchor` class directly. Instead, use one of its subclasses, based on the type of constraint you wish to create.
> - Use `NSLayoutXAxisAnchor` to create horizontal constraints.
> - Use `NSLayoutYAxisAnchor` to create vertical constraints.
> - Use `NSLayoutDimension` to create constraints that affect the view’s height or width.

Their subclasses provide additional type checking, preventing you from creating invalid constraints. For example, you can't create a constraint between `left` and `top` anchors.

Align needed more information about the kind of the attribute that the anchor was wrapping. Here's an anchor "taxonomy" that I've come up with:

```swift
// Each anchor is parameterized with a type and an axis.
struct Anchor<Type, Axis> {}

// There are four types of anchors:
final class AnchorTypeDimension {}
final class AnchorTypeCenter: AnchorTypeAlignment {}
final class AnchorTypeEdge: AnchorTypeAlignment {}
final class AnchorTypeBaseline: AnchorTypeAlignment {}

/// Alignments include `center`, `edge` and `baselines` anchors.
protocol AnchorTypeAlignment {}

// And there are two phantom types to represent axis:
final class AnchorAxisHorizontal {}
final class AnchorAxisVertical {}
```

A combination of `type` and `axis` is used to represent different anchors:

```swift
var top: Anchor<AnchorTypeEdge, AnchorAxisVertical>
var left: Anchor<AnchorTypeEdge, AnchorAxisHorizontal>
var centerX: Anchor<AnchorTypeCenter, AnchorAxisHorizontal>
var firstBaseline: Anchor<AnchorTypeBaseline, AnchorAxisVertical>
var width: Anchor<AnchorTypeDimension, AnchorAxisHorizontal>
```

With all that extra information available at compile time (type and axis), I could add methods tailored for each specific type of anchor. Here are a few examples.

Each anchor which defines view's alignment (`center`, `edge` and `baseline`) can be `aligned` with another alignment anchor, but only if the other anchor has the same axis (prevents users from creating invalid constraints!):

```swift
extension Anchor where Type: AnchorTypeAlignment {
    func align<Type: AnchorTypeAlignment>(with anchor: Anchor<Type, Axis>, offset: CGFloat = 0) -> NSLayoutConstraint
}
```

Each dimension anchor can `match` the other dimensions, no matter the axis.

```swift
extension Anchor where Type: AnchorTypeDimension {
    func match<Axis>(_ anchor: Anchor<AnchorTypeDimension, Axis>, offset: CGFloat = 0) -> NSLayoutConstraint

    // Or you can just set a constant size.
    func set(_ constant: CGFloat) -> NSLayoutConstraint
}
```

Edges can be `pinned` to a superview with insets (which is different from an offset!):

```swift
extension Anchor where Type: AnchorTypeEdge {
    func pinToSuperview(inset: CGFloat = 0) -> NSLayoutConstraint
}
```

With just a few phantom types I was able to add all that extra type information without having to subclass `Anchor` type (it is actually a simple `struct`). If I were to use the approach similar to `NSLayoutAnchor` it would lead to a class explosion (think `AnchorEdgeVertical`, `AnchorCenterVertical`, etc). More importantly, with generics I was able to target specific groups of anchors (e.g. `Anchor<*, Vertical>`, or `Anchor<Dimension, *>)>`).

## Resources

There are more examples of phantom types in Swift available online:

- [Functional Snippet #13: Phantom Types](https://www.objc.io/blog/2014/12/29/functional-snippet-13-phantom-types/)
- [Swift: Money with Phantom Types](https://www.natashatherobot.com/swift-money-phantom-types/)
- [Measurements and Units with Phantom Types](https://oleb.net/blog/2016/08/measurements-and-units-with-phantom-types/)
