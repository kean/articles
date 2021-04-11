---
layout: post
title: "Immutability and Builder Pattern"
description: "Modelling large immutable objects in Objective-C without creating telescoping initializers"
date: 2016-01-10 19:30:05 +0300
category: programming
tags: objective-c, ios, patterns
permalink: /post/immutability-and-builder-pattern
redirect_from: /blog/immutability-and-builder-pattern
uuid: 944811d2-f3f0-425c-8178-e317873e74cf
---

We are all aware of the [advantages of immutable objects](https://www.objc.io/issues/7-foundation/value-objects/). These is often a desire to make all or most model objects immutable, but they tend to have a lot of properties. How do you initialize such objects without creating a [telescoping initializer](https://stackoverflow.com/questions/11748682/telescoping-constructor)?

{% include ad-hor.html %}

## Immutable Objects in Cocoa

The first idea that comes to mind is to follow the steps of the platform. `Cocoa` objects often have an immutable and mutable counterpart. A good example of an immutable object is a `NSURLRequest` which has lots of properties. To construct it, you use its mutable counterpart `NSMutableURLRequest`. `NSMutableURLRequest` is a subclass of `NSURLRequest` which in turn implements `NSCopying` and `NSMutableCopying` protocols. We could do the same with our own classes.

Let's implement a `User` class and its mutable counterpart:

```objc
@interface User : NSObject <NSCopying, NSMutableCopying>

@property (nonnull, nonatomic, readonly) NSString *name;

- (nonnull instancetype)initWithUser:(nonnull User *)user;

@end


@interface MutableUser : User

@property (nullable, nonatomic) NSString *name;

@end
```

```objc
@interface User ()

@property (nonatomic) NSString *name;

@end

@implementation User

- (instancetype)initWithUser:(User *)user {
    if (self = [super init]) {
        _name = [user.name copy];
    }
    return self;
}

- (id)copyWithZone:(NSZone *)zone {
    return [[User alloc] initWithUser:self];
}

- (id)mutableCopyWithZone:(NSZone *)zone {
    return [[MutableUser alloc] initWithUser:self];
}

@end


@implementation MutableUser

@dynamic name;

@end
```

Now we are able to use those classes the same way we use `NSURLRequest`. However, there are several problems with this approach:

- Requires defensive copying to prevent accidental sharing of instances of `MutableUser` class where immutable `User` is expected
- We had to use four lines of code for a single `name` property
- We limited our ability to extend class hierarchy

Fortunately, there is an alternative way to create immutable objects which is an overlooked [builder pattern](https://en.wikipedia.org/wiki/Builder_pattern). It is a very simple pattern that addresses all those problems.

## Builder Pattern

Let's dive straight into implementation but this time we will start with a base class for our model objects - `Entity`.

```objc
@class EntityBuilder;

@interface Entity : NSObject

@property (nonnull, nonatomic, readonly) NSString *ID;

- (nonnull instancetype)initWithBuilder:(nonnull EntityBuilder *)builder;

@end


@interface EntityBuilder : NSObject

@property (nullable, nonatomic) NSString *ID;

- (nonnull Entity *)build;

@end
```

```objc
@implementation Entity

- (instancetype)initWithBuilder:(EntityBuilder *)builder {
    // You can also assers that validate builder here
    if (self = [super init]) {
        _ID = builder.ID;
    }
    return self;
}

@end


@implementation EntityBuilder

- (Entity *)build {
    return [[Entity alloc] initWithBuilder:self];
}

@end
```

`Entity` class has no mutable counterpart, defensive copying is no longer required. We also used just two lines of code for an `ID` property.


{% include references-start.html %}

- [objc.io: Value Objects](https://www.objc.io/issues/7-foundation/value-objects/)
- [Wikipedia: Builder Pattern](https://en.wikipedia.org/wiki/Builder_pattern)
- [Wikipedia: Immutable Objects](https://en.wikipedia.org/wiki/Immutable_object)
- [Concepts in Objective-C Programming: Object Mutability](https://developer.apple.com/library/mac/documentation/General/Conceptual/CocoaEncyclopedia/ObjectMutability/ObjectMutability.html)

{% include references-end.html %}
