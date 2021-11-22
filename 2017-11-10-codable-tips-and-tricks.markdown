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
favorite: true
---

<div class="UpdatesSections" markdown="1">
**Updates**

- Nov 21, 2021. Add references to the new `Codable` features and rework the existing tips.
</div>

I've just finished migrating our app to [Codable](https://developer.apple.com/documentation/swift/encoding_decoding_and_serialization) and I'd like to share some of the tips and tricks that I've come up with along the way.

> You can [download](/playgrounds/codable.playground.zip) a Swift Playground with all of the code from this article.
{:.info}

<img alt="Xcode screenshot showing Codable example" src="{{ site.url }}/images/posts/codable_screen_01.png" class="Screenshot">

{% include ad-hor.html %}

Apple introduced `Codable` in Swift 4 with a [motivation](https://github.com/apple/swift-evolution/blob/master/proposals/0167-swift-encoders.md#motivation) to replace old `NSCoding` APIs. Unlike `NSCoding` it has first-class JSON support making it a great option not just for persisting data but also for decoding JSON responses from web APIs.

`Codable` is great for its intended purpose of `NSCoding` replacement. If you just need to encode and decode some local data that you have full control over you might even be able to take advantage of [automatic encoding and decoding](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types).

In the real world though things get complicated *very quickly*. Building a fault tolerant system capable of dealing with all the quirks of the external JSON is a challenge.

One of the major downsides of `Codable` is that as soon as you need custom decoding logic - even for a single key - you have to provide custom everything: manually define all the *coding keys*, and implementing an entire `init(from decoder: Decoder) throws` initializer by hand. This, of course, isn't ideal. But fortunately, there are a few tricks that can make working with `Codable` a bit easier.

## 1. Safely Decoding Arrays

Let's say you need to load and display a list of posts in your app. Every `Post` has an `id` (required), `title` (required), and `subtitle` (optional).


<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">struct</span> <span class="kc">Post</span><span class="p">:</span> <span class="xc">Decodable</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">id</span><span class="p">:</span> <span class="kc">Id</span><span class="o">&lt;</span><span class="kc">Post</span><span class="o">&gt;</span> <span class="c1">// More about this type later.</span>
    <span class="k">let</span> <span class="nv">title</span><span class="p">:</span> <span class="xc">String</span>
    <span class="k">let</span> <span class="nv">subtitle</span><span class="p">:</span> <span class="xc">String</span><span class="p">?</span>
<span class="p">}</span>
</code></pre></div></div>

Swift compiler automatically generated `Decodable` implementation for this struct, taking into account what types are optional. Let's try to decode an array of posts.

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

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">do</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">posts</span> <span class="o">=</span> <span class="k">try</span> <span class="xc">JSONDecoder</span><span class="p">()</span><span class="o">.</span><span class="xv">decode</span><span class="p">([</span><span class="kc">Post</span><span class="p">]</span><span class="o">.</span><span class="k">self</span><span class="p">,</span> <span class="xv">from</span><span class="p">:</span> <span class="n">json</span><span class="o">.</span><span class="xv">data</span><span class="p">(</span><span class="xv">using</span><span class="p">:</span> <span class="o">.</span><span class="xv">utf8</span><span class="p">)</span><span class="o">!</span><span class="p">)</span>
<span class="p">}</span> <span class="k">catch</span> <span class="p">{</span>
    <span class="xv">print</span><span class="p">(</span><span class="n">error</span><span class="p">)</span>
    <span class="c1">// prints "No value associated with key title (\"title\")."</span>
<span class="p">}</span>
</code></pre></div></div>

It throws an error: [`.keyNotFound`](https://developer.apple.com/documentation/swift/decodingerror/2893451-keynotfound). The second post object is missing a required `title` field, so that makes sense. Swift provides a complete error report using [`DecodingError`](https://developer.apple.com/documentation/swift/decodingerror) which is extremely useful.

But what if you don't want a single corrupted post to prevent you from displaying an entire page of otherwise perfectly valid posts. I use a special `Safe<T>` type that allows me to safely decode an object. If it encounters an error during decoding, it fails safely and [sends a report](https://sentry.io/welcome/) to the development team:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">public</span> <span class="kd">struct</span> <span class="kc">Safe</span><span class="o">&lt;</span><span class="o">Base</span><span class="p">:</span> <span class="xc">Decodable</span><span class="o">&gt;</span><span class="p">:</span> <span class="xc">Decodable</span> <span class="p">{</span>
    <span class="kd">public</span> <span class="k">let</span> <span class="nv">value</span><span class="p">:</span> <span class="o">Base</span><span class="p">?</span>

    <span class="kd">public</span> <span class="k">init</span><span class="p">(</span><span class="n">from</span> <span class="nv">decoder</span><span class="p">:</span> <span class="xc">Decoder</span><span class="p">)</span> <span class="k">throws</span> <span class="p">{</span>
        <span class="k">do</span> <span class="p">{</span>
            <span class="k">let</span> <span class="nv">container</span> <span class="o">=</span> <span class="k">try</span> <span class="n">decoder</span><span class="o">.</span><span class="xv">singleValueContainer</span><span class="p">()</span>
            <span class="k">self</span><span class="o">.</span><span class="n">value</span> <span class="o">=</span> <span class="k">try</span> <span class="n">container</span><span class="o">.</span><span class="xv">decode</span><span class="p">(</span><span class="o">Base</span><span class="o">.</span><span class="k">self</span><span class="p">)</span>
        <span class="p">}</span> <span class="k">catch</span> <span class="p">{</span>
            <span class="nf">assertionFailure</span><span class="p">(</span><span class="s">"ERROR: </span><span class="se">\(</span><span class="n">error</span><span class="se">)</span><span class="s">"</span><span class="p">)</span>
            <span class="c1">// TODO: automatically send a report about a corrupted data</span>
            <span class="k">self</span><span class="o">.</span><span class="n">value</span> <span class="o">=</span> <span class="k">nil</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

Now I can indicate that I don't want to stop decoding in case of a single corrupted element:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">do</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">posts</span> <span class="o">=</span> <span class="k">try</span> <span class="xc">JSONDecoder</span><span class="p">()</span><span class="o">.</span><span class="xv">decode</span><span class="p">([</span><span class="kc">Safe</span><span class="o">&lt;</span><span class="kc">Post</span><span class="o">&gt;</span><span class="p">]</span><span class="o">.</span><span class="k">self</span><span class="p">,</span> <span class="xv">from</span><span class="p">:</span> <span class="n">json</span><span class="o">.</span><span class="xv">data</span><span class="p">(</span><span class="xv">using</span><span class="p">:</span> <span class="o">.</span><span class="xv">utf8</span><span class="p">)</span><span class="o">!</span><span class="p">)</span>
    <span class="nf">print</span><span class="p">(</span><span class="n">posts</span><span class="p">[</span><span class="mi">0</span><span class="p">]</span><span class="o">.</span><span class="kt">value</span><span class="o">!.</span><span class="kt">title</span><span class="p">)</span>    <span class="c1">// prints "Codable: Tips and Tricks"</span>
    <span class="nf">print</span><span class="p">(</span><span class="n">posts</span><span class="p">[</span><span class="mi">1</span><span class="p">]</span><span class="o">.</span><span class="kt">value</span><span class="p">)</span>           <span class="c1">// prints "nil"</span>
<span class="p">}</span> <span class="k">catch</span> <span class="p">{</span>
    <span class="nf">print</span><span class="p">(</span><span class="n">error</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

Keep in mind that `decode([Safe<Post>].self, from:...` is still going to throw an error if the data doesn't contain an array of contain something other than an array.

> Ignoring errors is not always the best strategy. For example, if you are displaying bank accounts and the backend suddenly starts sending invalid data for one of them, it’s might be better to show an error than cause a heart attack!
{:.warning}

## 2. Id Type and a Single Value Container

In the previous example, I used a special `Id<Post>` type. It gets parametrized with a generic parameter `Entity` which isn't used by the `Id` itself but is used by the compiler when comparing different `Id`. This way it can ensure that you can't accidentally pass `Id<Media>` where `Id<Image>` is expected.

> Generic types like this are sometimes referred to as *phantom* types. You can learn more about them in [Three Use-Cases of Phantom Types](https://kean.blog/post/phantom-types).
{:.info}

The `Id` type itself is very simple, it's just a wrapper for a raw `String`:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">public</span> <span class="kd">struct</span> <span class="kc">Id</span><span class="o">&lt;</span><span class="o">Entity</span><span class="o">&gt;</span><span class="p">:</span> <span class="xc">Hashable</span> <span class="p">{</span>
    <span class="kd">public</span> <span class="k">let</span> <span class="nv">raw</span><span class="p">:</span> <span class="xc">String</span>
    <span class="kd">public</span> <span class="k">init</span><span class="p">(</span><span class="n">_</span> <span class="nv">raw</span><span class="p">:</span> <span class="xc">String</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">raw</span> <span class="o">=</span> <span class="n">raw</span>
    <span class="p">}</span>
        
    <span class="kd">public</span> <span class="k">var</span> <span class="nv">hashValue</span><span class="p">:</span> <span class="xc">Int</span> <span class="p">{</span>
         <span class="kt">raw</span><span class="o">.</span><span class="xv">hashValue</span>
    <span class="p">}</span>
    
    <span class="kd">public</span> <span class="kd">static</span> <span class="kd">func</span> <span class="o">==</span><span class="p">(</span><span class="nv">lhs</span><span class="p">:</span> <span class="kc">Id</span><span class="p">,</span> <span class="nv">rhs</span><span class="p">:</span> <span class="kc">Id</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="xc">Bool</span> <span class="p">{</span>
        <span class="k">return</span> <span class="n">lhs</span><span class="o">.</span><span class="kt">raw</span> <span class="o">==</span> <span class="n">rhs</span><span class="o">.</span><span class="kt">raw</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

Adding `Codable` conformance to it is a bit more tricky. It requires a special [`SingleValueEncodingContainer`](https://developer.apple.com/documentation/swift/singlevalueencodingcontainer) type – a container that can support the storage and direct encoding of a single non-keyed value.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">extension</span> <span class="kc">Id</span><span class="p">:</span> <span class="xc">Codable</span> <span class="p">{</span>
    <span class="kd">public</span> <span class="k">init</span><span class="p">(</span><span class="n">from</span> <span class="nv">decoder</span><span class="p">:</span> <span class="xc">Decoder</span><span class="p">)</span> <span class="k">throws</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">container</span> <span class="o">=</span> <span class="k">try</span> <span class="n">decoder</span><span class="o">.</span><span class="xv">singleValueContainer</span><span class="p">()</span>
        <span class="k">let</span> <span class="nv">raw</span> <span class="o">=</span> <span class="k">try</span> <span class="n">container</span><span class="o">.</span><span class="xv">decode</span><span class="p">(</span><span class="xc">String</span><span class="o">.</span><span class="k">self</span><span class="p">)</span>
        <span class="k">if</span> <span class="n">raw</span><span class="o">.</span><span class="xv">isEmpty</span> <span class="p">{</span>
            <span class="k">throw</span> <span class="xc">DecodingError</span><span class="o">.</span><span class="xv">dataCorruptedError</span><span class="p">(</span>
                <span class="nv">in</span><span class="p">:</span> <span class="n">container</span><span class="p">,</span>
                <span class="nv">debugDescription</span><span class="p">:</span> <span class="s">"Cannot initialize Id from an empty string"</span>
            <span class="p">)</span>
        <span class="p">}</span>
        <span class="k">self</span><span class="o">.</span><span class="k">init</span><span class="p">(</span><span class="n">raw</span><span class="p">)</span>
    <span class="p">}</span>

    <span class="kd">public</span> <span class="kd">func</span> <span class="nf">encode</span><span class="p">(</span><span class="n">to</span> <span class="nv">encoder</span><span class="p">:</span> <span class="xc">Encoder</span><span class="p">)</span> <span class="k">throws</span> <span class="p">{</span>
        <span class="k">var</span> <span class="nv">container</span> <span class="o">=</span> <span class="n">encoder</span><span class="o">.</span><span class="xv">singleValueContainer</span><span class="p">()</span>
        <span class="k">try</span> <span class="n">container</span><span class="o">.</span><span class="xv">encode</span><span class="p">(</span><span class="kt">raw</span><span class="p">)</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

This gets the job done also ensuring that the ID has a non-empty value.

## 3. Safely Decoding Enums

Swift has great support for decoding (and encoding) enums and Swift 5.5, it even [generates conformances](https://github.com/apple/swift-evolution/blob/main/proposals/0295-codable-synthesis-for-enums-with-associated-values.md) for enums with associated types. Often all you need to do is declare a `Decodable` conformance synthesized automatically by a compiler (the enum raw type must be either `String` or `Int`).

Suppose you're building a system that displays all your devices on a map and you modelled your types the following way:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">enum</span> <span class="kc">System</span><span class="p">:</span> <span class="xc">String</span><span class="p">,</span> <span class="xc">Decodable</span> <span class="p">{</span>
    <span class="k">case</span> <span class="n">ios</span><span class="p">,</span> <span class="n">macos</span><span class="p">,</span> <span class="n">tvos</span><span class="p">,</span> <span class="n">watchos</span>
<span class="p">}</span>

<span class="kd">struct</span> <span class="kc">Location</span><span class="p">:</span> <span class="xc">Decodable</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">latitude</span><span class="p">:</span> <span class="xc">Double</span>
    <span class="k">let</span> <span class="nv">longitude</span><span class="p">:</span> <span class="xc">Double</span>
<span class="p">}</span>

<span class="kd">final</span> <span class="kd">class</span> <span class="kc">Device</span><span class="p">:</span> <span class="xc">Decodable</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">location</span><span class="p">:</span> <span class="kc">Location</span>
    <span class="k">let</span> <span class="nv">system</span><span class="p">:</span> <span class="kc">System</span>
<span class="p">}</span>
</code></pre></div></div>

Now, what if more systems are added in the future? The product decision might be to display the "unknown" devices on a map but indicate that the app update is needed. But how should you go about modeling this in the app? By default Swift throws a [`.dataCorrupted`](https://developer.apple.com/documentation/swift/decodingerror/2893182-datacorrupted?changes=lates_1) error if it encounters an unknown enum value:

```json
{
    "location": {
        "latitude": 37.3317,
        "longitude": 122.0302
    },
    "system": "caros"
}
```

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">do</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">device</span> <span class="o">=</span> <span class="k">try</span> <span class="xc">JSONDecoder</span><span class="p">()</span><span class="o">.</span><span class="xv">decode</span><span class="p">(</span><span class="kc">Device</span><span class="o">.</span><span class="k">self</span><span class="p">,</span> <span class="xv">from</span><span class="p">:</span> <span class="n">json</span><span class="o">.</span><span class="xv">data</span><span class="p">(</span><span class="xv">using</span><span class="p">:</span> <span class="o">.</span><span class="xv">utf8</span><span class="p">)</span><span class="o">!</span><span class="p">)</span>
<span class="p">}</span> <span class="k">catch</span> <span class="p">{</span>
    <span class="nf">print</span><span class="p">(</span><span class="n">error</span><span class="p">)</span>
    <span class="c1">// Prints "Cannot initialize System from invalid String value caros"</span>
<span class="p">}</span>
</code></pre></div></div>

You can make a system optional or add an explicit `.unknown` type. And then the most straightforward way to decode `system` safely is by implementing a custom `init(from decoder: Decoder) throws` initializer:

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

You could also use the `Safe` type introduced in one of the previous sections. But both of these options are not ideal because they ignore *all* the potential problems with the `system` value. This means that even corrupted data – e.g. a missing key, a number `123`, `null`, an empty object – gets decoded to `nil` (or `.unknown`). A more precise way to say "decode unknown strings as nil" would be:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">self</span><span class="o">.</span><span class="n">system</span> <span class="o">=</span> <span class="kc">System</span><span class="p">(</span><span class="kt">rawValue</span><span class="p">:</span> <span class="k">try</span> <span class="n">map</span><span class="o">.</span><span class="xv">decode</span><span class="p">(</span><span class="xc">String</span><span class="o">.</span><span class="k">self</span><span class="p">,</span> <span class="xv">forKey</span><span class="p">:</span> <span class="o">.</span><span class="kt">system</span><span class="p">))</span>
</code></pre></div></div>

## 4. Less Verbose Decoding

In the previous example, I used a custom initializer and it turned out [pretty verbose](https://bugs.swift.org/browse/SR-6063). Fortunately, there are a few ways to make it more concise.

### 4.1. Implitic Type Parameters

The first obvious thing is to get rid of the explicit type parameters.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">extension</span> <span class="xc">KeyedDecodingContainer</span> <span class="p">{</span>
    <span class="kd">public</span> <span class="kd">func</span> <span class="n">decode</span><span class="o">&lt;</span><span class="o">T</span><span class="p">:</span> <span class="xc">Decodable</span><span class="o">&gt;</span><span class="p">(</span><span class="n">_</span> <span class="nv">key</span><span class="p">:</span> <span class="xc">Key</span><span class="p">,</span> <span class="o">as</span> <span class="nv">type</span><span class="p">:</span> <span class="o">T</span><span class="o">.</span><span class="k">Type</span> <span class="o">=</span> <span class="o">T</span><span class="o">.</span><span class="k">self</span><span class="p">)</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="o">T</span> <span class="p">{</span>
        <span class="k">try</span> <span class="k">self</span><span class="o">.</span><span class="xv">decode</span><span class="p">(</span><span class="o">T</span><span class="o">.</span><span class="k">self</span><span class="p">,</span> <span class="xv">forKey</span><span class="p">:</span> <span class="n">key</span><span class="p">)</span>
    <span class="p">}</span>

    <span class="kd">public</span> <span class="kd">func</span> <span class="n">decodeIfPresent</span><span class="o">&lt;</span><span class="o">T</span><span class="p">:</span> <span class="xc">Decodable</span><span class="o">&gt;</span><span class="p">(</span><span class="n">_</span> <span class="nv">key</span><span class="p">:</span> <span class="xc">KeyedDecodingContainer</span><span class="o">.</span><span class="xc">Key</span><span class="p">)</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="o">T</span><span class="p">?</span> <span class="p">{</span>
        <span class="k">try</span> <span class="xv">decodeIfPresent</span><span class="p">(</span><span class="o">T</span><span class="o">.</span><span class="k">self</span><span class="p">,</span> <span class="xv">forKey</span><span class="p">:</span> <span class="n">key</span><span class="p">)</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

Let's go back to our `Post` example and extend it with an optional `webURL` property. If you try to decode the data posted below, you'll get a [`.dataCorrupted`](https://developer.apple.com/documentation/swift/decodingerror/2893182-datacorrupted?changes=latest_minor) error with an underlying error: "Invalid URL string.".

```swift
{
    "id": "pos_1",
    "title": "Codable: Tips and Tricks",
    "webURL": "http://google.com/🤬"
}
```

Let's implement a custom initializer to ignore this error and also take the new convenience methods for a spin.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">final</span>  <span class="kc">Post</span><span class="p">:</span> <span class="xc">Decodable</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">id</span><span class="p">:</span> <span class="kc">Id</span><span class="o">&lt;</span><span class="kc">Post</span><span class="o">&gt;</span>
    <span class="k">let</span> <span class="nv">title</span><span class="p">:</span> <span class="xc">String</span>
    <span class="k">let</span> <span class="nv">webURL</span><span class="p">:</span> <span class="xc">URL</span><span class="p">?</span>

    <span class="k">init</span><span class="p">(</span><span class="n">from</span> <span class="nv">decoder</span><span class="p">:</span> <span class="xc">Decoder</span><span class="p">)</span> <span class="k">throws</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">map</span> <span class="o">=</span> <span class="k">try</span> <span class="n">decoder</span><span class="o">.</span><span class="xv">container</span><span class="p">(</span><span class="xv">keyedBy</span><span class="p">:</span> <span class="kc">CodingKeys</span><span class="o">.</span><span class="k">self</span><span class="p">)</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">id</span> <span class="o">=</span> <span class="k">try</span> <span class="n">map</span><span class="o">.</span><span class="xv">decode</span><span class="p">(</span><span class="o">.</span><span class="kt">id</span><span class="p">)</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">title</span> <span class="o">=</span> <span class="k">try</span> <span class="n">map</span><span class="o">.</span><span class="xv">decode</span><span class="p">(</span><span class="o">.</span><span class="kt">title</span><span class="p">)</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">webURL</span> <span class="o">=</span> <span class="k">try</span><span class="p">?</span> <span class="n">map</span><span class="o">.</span><span class="xv">decode</span><span class="p">(</span><span class="o">.</span><span class="kt">webURL</span><span class="p">)</span>
    <span class="p">}</span>

    <span class="kd">private</span> <span class="kd">enum</span> <span class="kt">CodingKeys</span><span class="p">:</span> <span class="kt">CodingKey</span> <span class="p">{</span>
        <span class="k">case</span> <span class="n">id</span>
        <span class="k">case</span> <span class="n">title</span>
        <span class="k">case</span> <span class="n">webURL</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

This works. But as one of the Swift developers points out in [a comment to SR-6063](https://bugs.swift.org/browse/SR-6063?focusedCommentId=29267&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-29267) there are a few places when explicitly specifying the generic type is necessary. Well, you can always fallback to the built-in methods.

This new approach is great, but this way all the errors are ignored again. What would be ideal is to automatically send a report to the server about the corrupted data. You can do that pretty easily with a few more convenience methods for decoding values.

Before I wrap up this section, let me also share one more extension that I added:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">extension</span> <span class="xc">JSONDecoder</span> <span class="p">{</span>
    <span class="kd">public</span> <span class="kd">func</span> <span class="n">decodeSafelyArray</span><span class="o">&lt;</span><span class="o">T</span><span class="p">:</span> <span class="xc">Decodable</span><span class="o">&gt;</span><span class="p">(</span><span class="n">of</span> <span class="nv">type</span><span class="p">:</span> <span class="o">T</span><span class="o">.</span><span class="k">Type</span><span class="p">,</span> <span class="n">from</span> <span class="nv">data</span><span class="p">:</span> <span class="xc">Data</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="p">[</span><span class="o">T</span><span class="p">]</span> <span class="p">{</span>
        <span class="k">guard</span> <span class="k">let</span> <span class="nv">array</span> <span class="o">=</span> <span class="k">try</span><span class="p">?</span> <span class="xv">decode</span><span class="p">([</span><span class="kc">Decoded</span><span class="o">&lt;</span><span class="o">T</span><span class="o">&gt;</span><span class="p">]</span><span class="o">.</span><span class="k">self</span><span class="p">,</span> <span class="xv">from</span><span class="p">:</span> <span class="n">data</span><span class="p">)</span> <span class="k">else</span> <span class="p">{</span> <span class="k">return</span> <span class="p">[]</span> <span class="p">}</span>
        <span class="k">return</span> <span class="n">array</span><span class="o">.</span><span class="xv">flatMap</span> <span class="p">{</span> <span class="nv">$0</span><span class="o">.</span><span class="kt">raw</span> <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>

<span class="c1">// Usage (creates a plain array of posts – [Post])</span>
<span class="k">let</span> <span class="nv">posts</span> <span class="o">=</span> <span class="xc">JSONDecoder</span><span class="p">()</span><span class="o">.</span><span class="kt">decodeSafelyArray</span><span class="p">(</span><span class="kt">of</span><span class="p">:</span> <span class="kc">Post</span><span class="o">.</span><span class="k">self</span><span class="p">,</span> <span class="kt">from</span><span class="p">:</span> <span class="n">data</span><span class="p">)</span>
</code></pre></div></div>

### 4.2. Separate JSON Scheme

Another approach that I really like is defining a separate type to take advantage of automatic decoding, and then map it to your entity.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">final</span> <span class="kd">struct</span> <span class="kc">Post</span><span class="p">:</span> <span class="xc">Decodable</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">id</span><span class="p">:</span> <span class="kc">Id</span><span class="o">&lt;</span><span class="kc">Post</span><span class="o">&gt;</span>
    <span class="k">let</span> <span class="nv">title</span><span class="p">:</span> <span class="xc">String</span>
    <span class="k">let</span> <span class="nv">webURL</span><span class="p">:</span> <span class="xc">URL</span><span class="p">?</span>

    <span class="k">init</span><span class="p">(</span><span class="n">from</span> <span class="nv">decoder</span><span class="p">:</span> <span class="xc">Decoder</span><span class="p">)</span> <span class="k">throws</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">map</span> <span class="o">=</span> <span class="k">try</span> <span class="kc">PostScheme</span><span class="p">(</span><span class="xv">from</span><span class="p">:</span> <span class="n">decoder</span><span class="p">)</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">id</span> <span class="o">=</span> <span class="n">map</span><span class="o">.</span><span class="kt">postIdentifier</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">title</span> <span class="o">=</span> <span class="n">map</span><span class="o">.</span><span class="kt">title</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">webURL</span> <span class="o">=</span> <span class="n">map</span><span class="o">.</span><span class="kt">webURL</span><span class="p">?</span><span class="o">.</span><span class="kt">value</span>
    <span class="p">}</span>

    <span class="kd">private</span> <span class="kd">struct</span> <span class="kc">PostJSON</span><span class="p">:</span> <span class="xc">Decodable</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">postIdentifier</span><span class="p">:</span> <span class="kc">Id</span><span class="o">&lt;</span><span class="kc">Post</span><span class="o">&gt;</span>
        <span class="k">let</span> <span class="nv">title</span><span class="p">:</span> <span class="xc">String</span>
        <span class="k">let</span> <span class="nv">webURL</span><span class="p">:</span> <span class="kc">Safe</span><span class="o">&lt;</span><span class="xc">URL</span><span class="o">&gt;</span><span class="p">?</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

What I like about this approach is that it's declarative. It still uses automatic decoding while giving you a ton of flexibility. It's a great way to map the data to the format more suitable for your app.

## 5. Encoding Patch Parameters

And for the final type - the PATCH HTTP method. In REST, it is used to send a set of changes described to be applied to the entity.

- The keys not present in the request are ignored
- If the key is present, the value gets updated (`null` deletes a value)

The way I implemented this in the app is with a new `Parameter` type. Here's how it looks like:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">public</span> <span class="kd">enum</span> <span class="kc">Parameter</span><span class="o">&lt;</span><span class="o">Base</span><span class="p">:</span> <span class="xc">Codable</span><span class="o">&gt;</span><span class="p">:</span> <span class="xc">Encodable</span> <span class="p">{</span>
    <span class="k">case</span> <span class="n">null</span> <span class="c1">// parameter set to `null`</span>
    <span class="k">case</span> <span class="nf">value</span><span class="p">(</span><span class="o">Base</span><span class="p">)</span>

    <span class="kd">public</span> <span class="k">init</span><span class="p">(</span><span class="n">_</span> <span class="nv">value</span><span class="p">:</span> <span class="o">Base</span><span class="p">?)</span> <span class="p">{</span>
        <span class="k">self</span> <span class="o">=</span> <span class="n">value</span><span class="o">.</span><span class="xv">map</span><span class="p">(</span><span class="kc">Parameter</span><span class="o">.</span><span class="kt">value</span><span class="p">)</span> <span class="p">??</span> <span class="o">.</span><span class="kt">null</span>
    <span class="p">}</span>

    <span class="kd">public</span> <span class="kd">func</span> <span class="nf">encode</span><span class="p">(</span><span class="n">to</span> <span class="nv">encoder</span><span class="p">:</span> <span class="xc">Encoder</span><span class="p">)</span> <span class="k">throws</span> <span class="p">{</span>
        <span class="k">var</span> <span class="nv">container</span> <span class="o">=</span> <span class="n">encoder</span><span class="o">.</span><span class="xv">singleValueContainer</span><span class="p">()</span>
        <span class="k">switch</span> <span class="k">self</span> <span class="p">{</span>
        <span class="k">case</span> <span class="o">.</span><span class="nv">null</span><span class="p">:</span> <span class="k">try</span> <span class="n">container</span><span class="o">.</span><span class="xv">encodeNil</span><span class="p">()</span>
        <span class="k">case</span> <span class="kd">let</span> <span class="o">.</span><span class="nf">value</span><span class="p">(</span><span class="n">value</span><span class="p">):</span> <span class="k">try</span> <span class="n">container</span><span class="o">.</span><span class="xv">encode</span><span class="p">(</span><span class="n">value</span><span class="p">)</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

And the example usage:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">struct</span> <span class="kc">PatchParameters</span><span class="p">:</span> <span class="xc">Encodable</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">name</span><span class="p">:</span> <span class="kc">Parameter</span><span class="o">&lt;</span><span class="xc">String</span><span class="o">&gt;</span><span class="p">?</span>
<span class="p">}</span>

<span class="kd">func</span> <span class="nf">encoded</span><span class="p">(</span><span class="n">_</span> <span class="nv">params</span><span class="p">:</span> <span class="kc">PatchParameters</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="xc">String</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">data</span> <span class="o">=</span> <span class="k">try!</span> <span class="xc">JSONEncoder</span><span class="p">()</span><span class="o">.</span><span class="xv">encode</span><span class="p">(</span><span class="n">params</span><span class="p">)</span>
    <span class="k">return</span> <span class="xc">String</span><span class="p">(</span><span class="o">data</span><span class="p">:</span> <span class="n">data</span><span class="p">,</span> <span class="o">encoding</span><span class="p">:</span> <span class="o">.</span><span class="xv">utf8</span><span class="p">)</span><span class="o">!</span>
<span class="p">}</span>

<span class="nf">encoded</span><span class="p">(</span><span class="kc">PatchParameters</span><span class="p">(</span><span class="nv">name</span><span class="p">:</span> <span class="k">nil</span><span class="p">))</span>
<span class="c1">// prints "{}"</span>

<span class="nf">encoded</span><span class="p">(</span><span class="kc">PatchParameters</span><span class="p">(</span><span class="nv">name</span><span class="p">:</span> <span class="o">.</span><span class="kt">null</span><span class="p">))</span>
<span class="c1">//print "{"name":null}"</span>

<span class="nf">encoded</span><span class="p">(</span><span class="kc">PatchParameters</span><span class="p">(</span><span class="nv">name</span><span class="p">:</span> <span class="o">.</span><span class="kt">value</span><span class="p">(</span><span class="s">"Alex"</span><span class="p">)))</span>
<span class="c1">//print "{"name":"Alex"}"</span>
</code></pre></div></div>

Sweet, exactly what I wanted.

## Final Thoughts

Apple got a lot of things right with `Codable`, and I hope to see more improvements in the future!
