---
layout: post
title: "Are Optionals Any Good?"
description: "Taking a step back to see what is good and bad about Optional type in Swift"
date: 2018-06-14 18:00:00 +0300
category: programming
tags: programming
permalink: /post/optionals
uuid: d2dad45e-0c95-4f8e-a9bd-78aab52ad7ab
---

When Swift was first announced, one of its defining features was [optionals](https://developer.apple.com/documentation/swift/optional/). This concept is not new, it is wide-spread in [functional programming languages](https://en.wikipedia.org/wiki/Option_type). But are Swift optionals actually good and do they solve the problems that we had in Objective-C _effectively_?

> Swift is a great language (#pleaseImproveTooling) and I love using it - you can probably tell by my [GitHub page](https://github.com/kean). But optionals have always been something that many people don't seem to be happy about. I'd like to take a step back and look at why that might be the case.

{% include ad-hor.html %}

## Benefits

The best thing about optionals is when there aren't any. If a parameter or a property isn't optional, you have a guarantee that the value is always going to be there. In Objective-C you never had this guarantee. Though in many cases this wasn't a big deal - in Objective-C, famously, you could send messages to `nil`. But sometimes you did have to write asserts here and there, sometimes your app would crash.

> In Swift your app would also crash if you force-unwrap an optional. Just like with any other tool there is a way to misuse it.

Swift type system is also advanced enough to allow optionals to be used not just with reference types, but with value types. This eliminates a whole class of common bugs in Objective-C - remember `NSNotFound`?

## Challenges

It's great when there are no optionals. But very often this is not the case, and working with them comes with a few challenges.

### Learning

There are a lot of concepts to be learned in order to work with optionals effectively. To name a few:

- Optional Binding
- Optional Chaining
- Nil-Coalescing Operator
- Unconditional (Force) Unwrapping

These aren't the most difficult concepts in Swift, but it still takes time to learn them.

### Optional Closures Are Escaping

All closures are non-escaping by default. Would you expect this closure to also be non-escaping?

```swift
func foo(_ closure: (() -> Void)?) {
    closure()
}
```

It definitely doesn't escape the context. But is it non-escaping? Turns out it's not. Its formal type is `Optional<() -> Void>` and the closure is treated like it is [stored inside another value](https://www.jessesquires.com/blog/why-optional-swift-closures-are-escaping/) which makes it escaping.

Some of the Core Team members also feel that this behavior might [need to be changed](https://twitter.com/slava_pestov/status/998629531698118658). It would require another special-case for `Optional` in the Swift compiler.

### Double (Triple, Etc) Optionals

If you're carefully designing your APIs, you're not going to see many double (triple, etc) optionals. But if you do, working with them can be very annoying.

If you have two types `Optional<Optional<T>>` and `Optional<T>` chances are that you want to treat them the same - `.some(.none)` is the same as just `.none`. You end up feeling that you're not "using" optionals but trying to "get rid" of them.

As an exercise to the reader, can you tell what the value of `v` is going to be in each case?

```swift
let v1: String?? = "Str"
let v2: String?? = .some("Str")
let v3: String?? = .some(.some("Str"))
let v4: String?? = nil
let v5: String?? = .some(nil)
```

### Combining Optional Chaining and Try?

```swift
func foo() throws -> String? {
    return "1"
}
let count = try? foo()?.count
// let count = (try? foo())??.count
```

This one is a bit tricky. Because of the optional chaining it _looks_ like the result of `try? foo()` is `Optional<String>` (if it were a double optional you would have to use `??`). But the resulting `count` is actually a double optional (`Optional<Optional<Int>>`).

This is confusing and I couldn't figure out why it works this way (there is probably a logical explanation though). There is a [recent thread](https://forums.swift.org/t/make-try-optional-chain-flattening-work-together/7415) on Swift Evolution Forms which discusses this issue.

### String Interpolation

```
String interpolation produces a debug description for an optional value; did you mean to make this explicit?
```

You've probably seen this warning before. It happens when you're trying to use string interpolation with optionals:

```swift
let value: Int? = 2
print("Received value: \(value)")
```

What would you expect this call to print? I would say "2". Nope, it prints "Optional(2)" and produces a warning implemented in [SR-1882](https://bugs.swift.org/browse/SR-1882). This warning seems well-intentioned, but it seems to have created another problem. In most cases, you should really not be printing optionals. But you do for logging. People are going to be [complaining forever](https://forums.developer.apple.com/thread/74890) about it.

But how would you make it print "2"? Here's one way to do it:

```swift
print("Received value: \(value?.description ?? "null")")
```

### Just an Enum or Maybe Not?

The Swift compiler is very good at hiding the fact that optionals are implemented as a regular enum.

> `Optional` isn't actually a regular enum. There is a lot of special-casing in the compiler (e.g. syntax sugar, weak refences, optional bindings, etc) and there is probably going to be even more.

At some point every Swift developer is going to learn this.

```swift
public enum Optional<Wrapped> : ExpressibleByNilLiteral {
    case none
    case some(Wrapped)
}
```

And it raises questions of whether you need to use it as an enum and if so then when. One of the potential ways to use its cases explicitly is in `switch` statements - I personally find explicit cases easier to read and understand:

```swift
enum Foo {
    case bar(Int)
}

let foo: Foo? = Foo.bar(1)

// "implicit"
switch foo {
case let .bar(value)?: print(value)
default: break
}

// vs "explicit" case
switch foo {
case let .some(.bar(value)): print(value)
default: break
}
```

Sometimes this syntax might be a bit confusing.

### To Map or Not to Map

Option types are wide-spread in [functional programming](https://wiki.haskell.org/Maybe) languages. Here's `Maybe` type in Haskell for example:

```haskell
 data Maybe a = Just a | Nothing
     deriving (Eq, Ord)
```

The difference between Swift and functional programming languages is that the latter are very good at working with wrapped types. There seems to be three key components to this:

- [Monads, functors, applicatives](http://www.mokacoding.com/blog/functor-applicative-monads-in-pictures/)
- Convenience operators to work with them
- Everything is a function

At first glance Swift also has `map` (functor) and `flatMap` (monad) methods defined for `Optional`. But the other two components are missing. Swift in its current form is all about _methods_ and _properties_, not functions.

I think every Swift developer is familiar with these "functional" methods and is comfortable using them on types like `Sequence` and, well, I don't really have any other examples. But using these methods on `Optional` sometimes _feels_ weird (and even might be prohibited by your team's style guide).

Which version would you prefer?

```swift
func int(from string: String?) -> Int? {
    return string.flatMap(Int.init)

    // Or something like this?
    // return string >>= Int.init
}

func int(from string: String?) -> Int? {
    guard let string = string else { return nil }
    return Int(string)

    // Or maybe Kotlin way (`if` is an exression)?
    // return if string != nil ? Int(string) : nil
}
```

The first function is terser, but the second one isn't bad either - there is a clear `guard` which validates input parameters at the top and only after validation they are used. If `string` were used more than once, unwrapping it first makes sense.

And another example, this time we're using not a function (init), but a property:

```swift
func count(from string: String?) -> Int? {
    // Imagine that count was a function (which it isn't):
    return string.map(count)
    // return string <$> count
}

func count(from string: String?) -> Int? {
    return string?.count
}
```

See, in this case Swift gives us a solution to the problem (optional binding), which is even nicer than the functional way.

These are just the basic examples, they don't take into account things like currying (which Swift doesn't have) etc. The point of these examples was to show that these two ways of working with optionals are at odds in Swift.

### Weak References

Ever wondered how weak references work in Swift? They look like regular optionals, but they aren't, they are [special-cased in the compilter](https://github.com/apple/swift/blob/master/docs/weak.rst).

> This is not a "challenge" per se, but just pointing out how optionals aren't regular enums in Swift.

### Conditional Conformances

Until the introduction of [conditional conformances](https://swift.org/blog/conditional-conformance/) in Swift 4.1, `Optional<T>` where `T` was `Equatable` wasn't itself `Equatable`. The same was true for other protocols. Fortunately, this challenge is now behind us.

## Conclusion

So, are optionals in Swift any good and do they solve the problem in an efficient way? I don't have a clear answer.

As someone who prefers [New Jersey style](https://en.wikipedia.org/wiki/Worse_is_better), I'd love to say _no_. Objective-C had a simple and effective way of managing `nil` references.

But I really like Swift's type system and especially how it handles value types. I can't imagine modern programming without option types now. So even taking all these challenges into account, I'm leaning towards _yes_.

Will Swift lean even more towards functional programming in the future? Is it going to overcome these challenges and make optionals even more ergonomic? I can't wait to see.
