---
layout: post
title: "Codable: Tips and Tricks"
description: "Introduced in Swift 4 to replace NSCoding APIs, Codable also features first-class JSON support"
date: 2017-11-10 18:00:00 +0300
category: programming
tags: ios
permalink: /post/codable-tips-and-tricks
uuid: 8b6a08fa-ba58-4f9a-b607-7bd4d1171f7f
cover: /images/posts/codable_cover.png
---

I've migrated our app to [Codable](https://developer.apple.com/documentation/swift/encoding_decoding_and_serialization). I'd like to share with you some of the tips and tricks that I've come up with along the way.

> <a href="{{ site.url }}/playgrounds/codable.playground.zip">Swift Playground</a> with all of the code from this article:

<img src="{{ site.url }}/images/posts/codable_screen_01.png" class="Screenshot">

{% include ad-hor.html %}

`Codable` was introduces in Swift 4 with a [motivation](https://github.com/apple/swift-evolution/blob/master/proposals/0167-swift-encoders.md#motivation) to replace old `NSCoding` APIs. Unlike `NSCoding` it has a first class JSON support which makes it a promising option for consuming JSON APIs.

`Codable` is great for its intended purpose of `NSCoding` replacement. If you just need to encode and decode some local data which you have full control over you might even be able to take advantage of [automatic encoding and decoding](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types).

In the real world though things get very *complicated* very *quickly*. Trying to build a fault tolerant system which deals with all the quirks of the external JSON and models all the product requirements is a challenge.

One of the major downsides of `Codable` is that as soon as you need custom decoding logic - even for a single key - you have to provide custom everything: manually define all the *coding keys*, and implementing an entire `init(from decoder: Decoder) throws` initializer by hand. This isn't ideal. But it is at least as good (or bad) as third-party JSON libraries in Swift. Having one built into the standard library is definitely a win.

So if you'd like to start using `Codable` in your app (and you are already familiar with all the basics) here are some tips and tricks that you may find helpful:

* TOC
{:toc}

## 1. Safely Decoding Arrays

Let's say you want to load and display a collection of posts (`Post`) in your app. Each `Post` has an `id` (required), `title` (required), and `subtitle` (optional).


```swift
final class Post: Decodable {
    let id: Id<Post> // More about this type later.
    let title: String
    let subtitle: String?
}
```

The `Post` class nicely models the requirements. It already adopts `Decodable` protocol so we are ready to decode some data:

```json
[
    {
        "id": "pos_1",
        "title": "Codable: Tips and Tricks"
    },
    {
        "id": "pos_2"
    }
]
```

```swift
do {
    let posts = try JSONDecoder().decode([Post].self, from: json.data(using: .utf8)!)
} catch {
    print(error)
    // prints "No value associated with key title (\"title\")."
}
```

As you might have expected we received a [`.keyNotFound`](https://developer.apple.com/documentation/swift/decodingerror/2893451-keynotfound) error because the second post object doesn't have a `title`.

> When the data doesn't match an expected format (e.g. it might be a result of miscommunication, regression, or unexpected user input) the system should automatically report an error to give the developers a chance to fix it. Swift provides a thorough error report in a form of [`DecodingError`](https://developer.apple.com/documentation/swift/decodingerror) any time the decoding fails which is extremely useful.

In most cases, you wouldn't want a single corrupted post to prevent you from displaying an entire page of other perfectly valid ones. To prevent this from happening I use a special `Safe<T>` type which allows me to safely decode an object. If it encounters an error during decoding it fails safely and [sends a report](https://sentry.io/welcome/):

```swift
public struct Safe<Base: Decodable>: Decodable {
    public let value: Base?

    public init(from decoder: Decoder) throws {
        do {
            let container = try decoder.singleValueContainer()
            self.value = try container.decode(Base.self)
        } catch {
            assertionFailure("ERROR: \(error)")
            // TODO: automatically send a report about a corrupted data
            self.value = nil
        }
    }
}
```

Now when I decode an array I can indicate that I don't want to stop decoding in case of a single corrupted element:

```swift
do {
    let posts = try JSONDecoder().decode([Safe<Post>].self, from: json.data(using: .utf8)!)
    print(posts[0].value!.title)    // prints "Codable: Tips and Tricks"
    print(posts[1].value)           // prints "nil"
} catch {
    print(error)
}
```

> Keep in mind that `decode([Safe<Post>].self, from:...` call is going to throw an error if the data doesn't contain an array. In general errors like that should be caught on a higher level. The common API contract is to always return an empty array if there are no elements to return.

## 2. Id Type and a Single Value Container

In the previous example, I've used a special `Id<Post>` type. The `Id` type gets parametrized with a generic parameter `Entity` which isn't actually used by the `Id` itself but is used by the compiler when comparing different types of `Id`s. This way the compiler ensures that I can't accidentally pass `Id<Media>` where `Id<Image>` is expected.

> Another place where I used *phantom types* for type safety is my <a href="{{ site.url }}/post/api-client">**API Client in Swift**</a> post.

The `Id` type itself is very simple, it's just a wrapper on top of a raw `String`:

```swift
public struct Id<Entity>: Hashable {
    public let raw: String
    public init(_ raw: String) {
        self.raw = raw
    }
        
    public var hashValue: Int {
        return raw.hashValue
    }
    
    public static func ==(lhs: Id, rhs: Id) -> Bool {
        return lhs.raw == rhs.raw
    }
}
```

Adding `Codable` conformance to it is a bit tricky. It requires a special [`SingleValueEncodingContainer`](https://developer.apple.com/documentation/swift/singlevalueencodingcontainer) type:

> A container that can support the storage and direct encoding of a single non-keyed value.

```swift
extension Id: Codable {
    public init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        let raw = try container.decode(String.self)
        if raw.isEmpty {
            throw DecodingError.dataCorruptedError(
                in: container,
                debugDescription: "Cannot initialize Id from an empty string"
            )
        }
        self.init(raw)
    }

    public func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(raw)
    }
}
```

As you can see from the code above `Id` also has a special rule that prevents it from being initialized from an empty `String`.


## 3. Safely Decoding Enums

Swift has a great support for decoding (and encoding) enums. In many cases, all you need to do is to just declare a `Decodable` conformance which gets synthesized automatically by a compiler (the enum raw type must be either `String` or `Int`).

Suppose you're building a system that displays all your devices on a map. A device has a `location` (required) and `system` (required) which it's running.

```swift
enum System: String, Decodable {
    case ios, macos, tvos, watchos
}

struct Location: Decodable {
    let latitude: Double
    let longitude: Double
}

final class Device: Decodable {
    let location: Location
    let system: System
}
```

Now here is the question. What if more systems are added in the future? The product decision might be to still display those devices but somehow indicate that the system is "unknown". Now how should you go about modeling this in the app?

By default Swift would throw a [`.dataCorrupted`](https://developer.apple.com/documentation/swift/decodingerror/2893182-datacorrupted?changes=lates_1) error if it encounters unknown enum value:

```json
{
    "location": {
        "latitude": 37.3317,
        "longitude": 122.0302
    },
    "system": "caros"
}
```

```swift
do {
    let device = try JSONDecoder().decode(Device.self, from: json.data(using: .utf8)!)
} catch {
    print(error)
    // Prints "Cannot initialize System from invalid String value caros"
}
```

How can `system` be modeled and be decoded in a safe way? One way is to make `system` property optional which would mean "unknown". And the most straightforward way to decode `system` safely is by implementing a custom `init(from decoder: Decoder) throws` initializer:

```swift
final class Device: Decodable {
    let location: Location
    let system: System?

    init(from decoder: Decoder) throws {
        let map = try decoder.container(keyedBy: CodingKeys.self)
        self.location = try map.decode(Location.self, forKey: .location)
        self.system = try? map.decode(System.self, forKey: .system)
    }

    private enum CodingKeys: CodingKey {
        case location
        case system
    }
} 
```

Keep in mind that this version simply ignores *all* the potential problems with `system` value. This means that even "corrupted" data (e.g. missing key `system`, a number `123`, `null`, empty object `{}` - depending on what the API contract is) gets decoded to `nil` ("unknown"). A more precise way to say "decode unknown strings as nil" would be:

```swift
self.system = System(rawValue: try map.decode(String.self, forKey: .system))
```


## 4. Less Verbose Manual Decoding

In the previous example, we had to implement a custom initializer `init(from decoder: Decoder) throws` which turned out [pretty verbose](https://bugs.swift.org/browse/SR-6063). Fortunately, there are a few ways to make it terser.

### 4.1. Getting Rid of Explicit Type Parameters

One option is to get rid of explicit type parameters:

```swift
extension KeyedDecodingContainer {
    public func decode<T: Decodable>(_ key: Key, as type: T.Type = T.self) throws -> T {
        return try self.decode(T.self, forKey: key)
    }

    public func decodeIfPresent<T: Decodable>(_ key: KeyedDecodingContainer.Key) throws -> T? {
        return try decodeIfPresent(T.self, forKey: key)
    }
}
```

Let's go back to our `Post` example and extend it with `webURL` property (optional). If we try to decode the data posted below we'll get a [`.dataCorrupted`](https://developer.apple.com/documentation/swift/decodingerror/2893182-datacorrupted?changes=latest_minor) error with an underlying error: `"Invalid URL string."`.

```swift
{
    "id": "pos_1",
    "title": "Codable: Tips and Tricks",
    "webURL": "http://google.com/🤬"
}
```

To decode this data safely we could implement a custom `init(from decoder: Decoder) throws` initializer but this time take advantage of our new `decode(...)` methods which don't require an explicit type parameter:

```swift
final class Post: Decodable {
    let id: Id<Post>
    let title: String
    let webURL: URL?

    init(from decoder: Decoder) throws {
        let map = try decoder.container(keyedBy: CodingKeys.self)
        self.id = try map.decode(.id)
        self.title = try map.decode(.title)
        self.webURL = try? map.decode(.webURL)
    }

    private enum CodingKeys: CodingKey {
        case id
        case title
        case webURL
    }
}
```

> As one of the Swift developers points out in [a comment to SR-6063](https://bugs.swift.org/browse/SR-6063?focusedCommentId=29267&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-29267) there are places when explicitly specifying the generic type is necessary.

This is great, however this way we simply ignore all the errors. What we'd really like to do is automatically send a report about a corrupted URL like that.

In order to deal with the second problem I currently use a special family of `decodeSafely...` methods. This is however still a work in progress:

```swift
extension KeyedDecodingContainer {
    public func decodeSafely<T: Decodable>(_ key: KeyedDecodingContainer.Key) -> T? {
        return self.decodeSafely(T.self, forKey: key)
    }

    public func decodeSafely<T: Decodable>(_ type: T.Type, forKey key: KeyedDecodingContainer.Key) -> T? {
        let decoded = try? decode(Safe<T>.self, forKey: key)
        return decoded?.value
    }

    public func decodeSafelyIfPresent<T: Decodable>(_ key: KeyedDecodingContainer.Key) -> T? {
        return self.decodeSafelyIfPresent(T.self, forKey: key)
    }

    public func decodeSafelyIfPresent<T: Decodable>(_ type: T.Type, forKey key: KeyedDecodingContainer.Key) -> T? {
        let decoded = try? decodeIfPresent(Safe<T>.self, forKey: key)
        return decoded??.value
    }
}
```

Here's how to use it:

```swift
self.webURL = map.decodeSafelyIfPresent(.webURL)
```

It would try to decode a URL (only in case it's present). If a URL isn't valid it would fail safely and send a report.

Another "safe" functions that would be nice to have is the one for safely decoding array of elements:

```swift
extension KeyedDecodingContainer {
    public func decodeSafelyArray<T: Decodable>(of type: T.Type, forKey key: KeyedDecodingContainer.Key) -> [T] {
        let array = decodeSafely([Safe<T>].self, forKey: key)
        return array?.flatMap { $0.raw } ?? []
    }
}

extension JSONDecoder {
    public func decodeSafelyArray<T: Decodable>(of type: T.Type, from data: Data) -> [T] {
        guard let array = try? decode([Decoded<T>].self, from: data) else { return [] }
        return array.flatMap { $0.raw }
    }
}


// Usage:

let posts: [Post] = JSONDecoder().decodeSafelyArray(of: Post.self, from: data)

init(from decoder: Decoder) throws {
    let map = try decoder.container(keyedBy: CodingKeys.self)
    self.posts = map.decodeSafelyArray(of: Post.self, forKey: .posts)
}
```


### 4.2. Using Separate Decoding Scheme

Another approach that I find promising is to define a separate `<#Type#>Scheme` type which takes advantage of automatic decoding, but uses some helper types (like `Safe`) to customize the decoding process:

```swift
final class Post: Decodable {
    let id: Id<Post>
    let title: String
    let webURL: URL?

    init(from decoder: Decoder) throws {
        let map = try PostScheme(from: decoder)
        self.id = map.id
        self.title = map.title
        self.webURL = map.webURL?.value
    }

    final class PostScheme: Decodable {
        let id: Id<Post>
        let title: String
        let webURL: Safe<URL>?
    }
}
```

The upside of this approach is that we take advantage of automatic decoding. The scheme is also arguably easier to read then a wall of manual `decode(...)` calls. The scheme describes an API contract: which keys are required, which are optional, and which can be "corrupted" without breaking an entire app (you can progressively disable some of the features instead).

There are plenty of downsides too, so I'm reluctant to recommend this approach.


## 5. Encoding Patch Parameters

A common `PATCH` request in our app follows this set of rules:
- The keys not present in the request are ignored
- If the key is present the value entry gets updated (`null` deletes a value)

One way to implement this using `Codable` involves another new custom `Parameter` type. Here's how it looks like:


```swift
public enum Parameter<Base: Swift.Codable>: Swift.Encodable {
    case null // parameter set to `null`
    case value(Base)

    public init(_ value: Base?) {
        self = value.map(Parameter.value) ?? .null
    }

    public func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        switch self {
        case .null: try container.encodeNil()
        case let .value(value): try container.encode(value)
        }
    }
}
```

Here's how it work on a simple example:

```swift
struct PatchParameters: Swift.Encodable {
    let name: Parameter<String>?
}

func encoded(_ params: PatchParameters) -> String {
    let data = try! JSONEncoder().encode(params)
    return String(data: data, encoding: .utf8)!
}

encoded(PatchParameters(name: nil))
// prints "{}"

encoded(PatchParameters(name: .null))
//print "{"name":null}"

encoded(PatchParameters(name: .value("Alex")))
//print "{"name":"Alex"}"
```
