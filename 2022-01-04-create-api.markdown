---
layout: post
title: "Introducing CreateAPI"
subtitle: Delightful code generation for OpenAPI specs
description: Delightful code generation for OpenAPI specs
date: 2022-01-04 09:00:00 -0500
category: programming
tags: programming
permalink: /post/create-api
uuid: 183dcd4d-27dd-4d08-924f-16b76888a5ad
---

If you've tried OpenAPI spec generators, you know how it goes. They get you about 60-80% there, but you end up having to modify the code by hand. For one of the specs ([GitHub REST API spec](https://github.com/github/rest-api-description)), a popular code generator I tried produced more than 300 compile-time errors. With [CreateAPI](https://github.com/kean/CreateAPI), I pushed well beyond just making sure the generated code compiles.

<div class="blog-new-li" markdown="1">
- **Fast**: processes specs with 100K lines of YAML in less than a second
- **Smart**: generates Swift code that looks like it's carefully written by hand
- **Reliable**: tested on 1KK lines of [publically available](https://apis.guru) OpenAPI specs producing correct code every time
- **Customizable**: offers a ton of customization options
</div>

I started with a powerful foundation: [OpenAPIKit](https://github.com/mattpolzin/OpenAPIKit) by [Mathew Polzin](https://github.com/mattpolzin). It takes care of the parsing and validation, and CreateAPI has more than 3K lines of Swift focusing purely on code generation.

## Snapshot Testing

I went to the extreme when testing CreateAPI. I wrote a simple snapshot testing utility. I then took a couple of hundred publically available OpenAPI specs from [APIs Guru](https://apis.guru) and other resources and feed them all to CreateAPI with default parameters. It then makes sure that the generated code doesn't miss any schemas, paths, or properties from the input specs, and that all of the generated code compiles successfully.

One of the main examples I used was [GitHub REST API spec](https://github.com/github/rest-api-description). It's a massive spec with more than 70K lines of YAML. For this spec, I went further than just making sure it compiles. I went through all (or almost all) paths and entities making sure they match the documentation. I also wrote a ton of unit tests feeding the mock JSON responses provided by GitHub to the framework, testing runtime correctness. I released the generated code as a separate framework – [OctoKit](https://github.com/kean/OctoKit).

## Optimizations

I have to talk about performance. CreateAPI is optimized to run on multicore processors like M1 Pro/Max. There is parallelization at every stage; even parsing is partially parallelized. And code generation saturates _all_ available code. As a result, it processes 100K lines of OpenAPI specs in less than a second.

CreateAPI was built entirely on 10-Core M1 Pro, which made it feasible to work with massive test inputs: 1KK of OpenAPI specs and 500K of generated Swift code. This beast compiles 500K lines of Swift code (heavily modularized) in less than 2 minutes – absolutely insane.

## Code Generation

And finally, let's talk about some of the optimizations it does to the produced code. Here are just a few examples:

### Swifty Booleans

<blockquote class="quotation" markdown="1">
Uses of Boolean methods and properties should read as assertions about the receiver when the use is nonmutating, e.g. x.isEmpty
 
*From https://www.swift.org/documentation/api-design-guidelines/*
</blockquote>

Unfortunately, it's uncommon for web APIs to follow this convention. This is why CreateAPI automatically generates Swifty property names for you.

```yaml
visible:
  type: boolean
```

Becomes:

```swift
public var isVisible: Bool
```

It's also smart enough not to add `is` when it's not needed:

```yaml
has_issues:
  type: boolean
```

```swift
public var hasIssues: Bool
```

> For every "smart" technique, CreateAPI has an option to either fully disable it, or add exceptions. For example, you can rename anything in generated coded: properties, types, enums:
>
>     rename:
>       # Rename properties, example:
>       #   - name: firstName
>       #   - SimpleUser.name: firstName
>       properties: {}
{:.info}

### Abbreviations

CreateAPI automatically capitalizes common abbreviations.

Before:

```swift
public var repoId: String
public var repoUrl: URL
```

After:

```swift
public var repoID: String
public var repoURL: URL
```

### Ordering Properties

Most generators ignore the original order of properties in the specs. Yet they are often designed with a particular order in mind. CreateAPI preserves the order of the properties (thanks to ordered dictionaries in [OpenAPIKit](https://github.com/mattpolzin/OpenAPIKit)). And if you want to, you can switch to alphabetical order if needed.

### Removing Noise

Readability is important. CreateAPI makes sure that it doesn't generate any unnecessary code. For example, a simpler generator might add all enum case names by default – it's just simpler to impleemnt it this way:

```swift
public enum Status: String, Codable, Equatable, CaseIterable {
    case placed = "placed"
    case approved = "approved"
    case delivered = "delivered"
}
```

CreateAPI doesn't add redudant names. It's a small thing, but all these small things add up:

```swift
public enum Status: String, Codable, CaseIterable {
    case placed
    case approved
    case delivered
}
```

### Inlining Requests and Queries

For some simple requests and queries, CreateAPI inlines the code, significantly reducing the amount of generated code.

```swift
public func post(accessToken: String) -> Request<OctoKit.Reaction>
    .post(path, body: ["accessToken": accessToken])
```

Same with query parameters:

```swift
private func makeGetQuery(_ perPage: Int, _ cursor: String) -> [(String, String?)] {
    [("per_page", String(perPage)), ("cursor", cursor))]
}
```

### Inlining Typealiases

Authors of OpenAPI specs often define arrays with items defined inline:

```yaml
search-result-text-matches:
  title: Search Result Text Matches
  type: array
  items:
    type: object
    properties:
      object_url:
        type: string
      object_type:
        nullable: true
        type: string
      ...
```

Many tools fails to produce anything useful for these specs. Here's one example:

```swift
public typealias SearchResultTextMatches = [SearchResultTextMatches]
```

t doesn't compile - the element definition is completely missing and the typealias refers to itself.

In that situation, CreateAPI uses multiple techniques that together produce the following output:

```swift
public struct SearchResultTextMatch: Codable {
    public var objectURL: String?
    public var objectType: String?
    public var property: String?
    // ...

    public init(objectURL: String? = nil, objectType: String? = nil, property: String? = nil, fragment: String? = nil, matches: [Match]? = nil) {
        // ...
    }

    private enum CodingKeys: String, CodingKey {
        case objectURL = "object_url"
        case objectType = "object_type"
        case property
        // ...
    }
}

// Usage
public struct TopicSearchResultItem: Codable {
    public var textMatches: [SearchResultTextMatch]
    // ...
```

Here is what happens under the hood:

- CreateAPI encounts a schema named "search-result-text-matches" of an "array" type.
- It checks the type of the element. In this case, it's an anonymous object defined inline. It generates a declaration for an object and names it `SearchResultTextMatch` - a singularized form of "search-result-text-matches".
- It creates a typealias, but never adds it to the output. When the "search-result-text-matches" it used anywhere else in the app, it inlines the typealias: `var textMatches: [SearchResultTextMatch]`.


### Comments

CreateAPI adds all comments and examples from the spec. But it's also smart enough to remove redundant comments. For example, in the following schema, the title matches the description, so we can keep only the title. But the title also matches the name of the entity, so the comments are completely redundant.

```yaml
validation-error:
  title: Validation Error
  description: Validation Error
  type: object
  required:
    - message
    - documentation_url
```

```swift
public struct ValidationError: Codable {
    public var message: String
    public var documentationURL: String
    // ...
```

### Edge Cases

After going through 1KK lines of OpenAPI specs, I hit a ton of edge cases. I just want to add one example. Here's what I found in Telegram's spec:

```yaml
emoji:
  default: 🎲
  type: string
  enum:
    - 🎲
    - 🎯
    - 🏀
    - ⚽
    - 🎰
```

Naturally, none of the generators I tried were able to handle it correctly. The problem is that a naive approach doesn't work because emojis can't be used as enum case names in Swift. CreateAPI uses powerful Swift APIs for dealing with Strings and generates the following code:

```swift
public enum Emoji: String, Codable, CaseIterable {
    case gameDie = "🎲"
    case directHit = "🎯"
    case basketballAndHoop = "🏀"
    case soccerBall = "⚽"
    case slotMachine = "🎰"
}
```

## Pre-Release

I pushed the first pre-release version of [CreateAPI](https://github.com/kean/CreateAPI), and it joined the rest of the frameworks for working with web APIs.

<div class="blog-new-li" markdown="1">
- [Nuke](https://github.com/kean/Nuke) - image loading an caching
- [NukeUI](https://github.com/kean/NukeUI) - UI components for image loading
- [Get](https://github.com/kean/Get) - web API client built using async/await
- [CreateAPI](https://github.com/kean/CreateAPI) - code generator for OpenAPI specs
- [Pulse](https://github.com/kean/Pulse) - network logger and inspector
- [URLQueryEncoder](https://github.com/kean/URLQueryEncoder) - URL query encoder based on Codable
- [NaiveDate](https://github.com/kean/NaiveDate) - working with dates without timezones
- [HTTPHeaders](https://github.com/kean/HTTPHeaders) - simple handling of HTTP response headers
</div>

There are still a ton of improvements in the backlog. I would also strongly appreciate any community contributions!