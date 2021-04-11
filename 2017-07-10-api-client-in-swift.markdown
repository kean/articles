---
layout: post
title: "API Client in Swift"
description: "How to implement a declarative and powerful API client using Alamofire, RxSwift, and Codable"
date: 2017-07-10 9:00:00 +0300
category: programming
tags: ios
permalink: /post/api-client
uuid: 543b6221-e3ec-4636-be57-6e6eb501e1f6
---

Consuming a web service API was a major part of almost all the projects that I worked on. Each one of them had a different approach to networking.

- My very first iOS project was built using [ASIHTTPRequest](https://github.com/pokeb/asi-http-request). `AFNetworking` was yet to be released. The XML responses were parsed using [TouchXML](https://github.com/TouchCode/TouchXML).
- The next major project that I've worked on used [AFNetworking](https://github.com/AFNetworking/AFNetworking) which was just recently introduced. We've later wrapped the API calls in our in-house [Promises/A+](https://promisesaplus.com) to make things like chaining requests easier. This was well before [PromiseKit](https://github.com/mxcl/PromiseKit) first appeared on GitHub. The JSON responses were parsed manually with a help of a simple [safe_cast](https://gist.github.com/kean/3600ad35c818a6b28caa3e0fa026d478) macro.
- The latest project that I've worked on had a custom API layer written on top of `NSURLSession`. It supported things like [batch HTTP requests](https://tools.ietf.org/id/draft-snell-http-batch-00.html#http-batch). The JSON responses were parsed using our in-house Objective-C JSON wrapper which was similar in terms of features to one of the first Swift JSON wrappers [SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON).

{% include ad-hor.html %}

Each of those had its own good and bad parts. I'm not going to do a full retrospect, but I'd like to stress out that the majority of the problems in those different implementations *were not caused but some intrinsic flaws* of the technologies being used. It's always about how we were using those tools.

In this article, I'd like to share my latest networking stack. It has a minimalistic yet powerful API, type-safe authorization scopes, endpoints are modeled in a type-safe, declarative and concise way, support for OAuth 2 "refresh access token" dance. And it takes full advantage of the open source frameworks to achieve all of those powerful features 

> All of the code from the post is [available here](https://gist.github.com/kean/64b9fc0963fd430594fdb3eb848bccf3) (requires Swift 4).

* TOC
{:toc}

## Dependencies

Let's start with dependencies. As Swift ecosystem grows it becomes increasingly more complicated to select the right tools for the job, just because of the sheer number of options. Especially for someone who likes to go through an entire library code base before making a final choice.

### 1. Alamofire

[Alamofire](https://github.com/Alamofire/Alamofire) is a workhorse of many iOS projects. It has a lot of convenient features:

- Dispatches from `Foundation.URLSession(Task/DataTask)Delegate` methods to individual requests
- Parameter encoding including URL encoding and JSON encoding
- Response validation
- Default `Accept-Encoding`, `Accept-Language`, and `User-Agent` HTTP headers
- Tools for implementing automatic OAuth 2 "refresh access token" dance
- Generate cURL command output
- Automatically show and hide network activity indicator

`Alamofire` is a great framework with a solid code base and comprehensive documentation. The only real alternative to `Alamofire` is writing your own abstraction on top of `Foundation.URLSession`. It's absolutely possible and relatively simple, but it requires a lot of boilerplate code.

### 2. RxSwift

[RxSwift](https://github.com/ReactiveX/RxSwift) has become a must-have tool for me. It gives you all of the advantages of promises and [much more](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Why.md). One of its underrated features which happen to be one of me my favorite is its built-in [testing support](https://kean.github.io/post/rxswift-testing). Why does it make sense to wrap your API calls into `Observables`? I'm going to provide a couple of examples later in a "Usage" section to show exactly that.

> There is a number of reasons why I prefer RxSwift over ReactiveCocoa, but I'd like not to dive into this discussion as part of this article.

### 3. Codable

The new [Swift Encoders and Decoders](https://github.com/apple/swift-evolution/blob/master/proposals/0167-swift-encoders.md) - `Codable` - is the way to go for the majority of the apps. `Codable` was introduces in Swift 4 with a [motivation](https://github.com/apple/swift-evolution/blob/master/proposals/0167-swift-encoders.md#motivation) to replace old `NSCoding` APIs. Unlike `NSCoding` it has a first class JSON support which makes it a promising option for consuming JSON APIs. `Codable` does have a few drawbacks which you can learn more about in a separate post [**Codable: Tips and Tricks**](https://kean.github.io/post/codable-tips-and-tricks).

There are a lot of third-party alternatives which were in use prior to Swift 4. Some of them still have advantages over `Codable.`. There is a [great article](https://github.com/bwhiteley/JSONShootout) which gives an overview of all of the major third-party JSON libraries.

## Endpoint

Now let's dive into the code. One of the questions to be answered is how to model API endpoints.

> [Moya](https://github.com/Moya/Moya) [recommends that you model](https://gist.github.com/kean/a2461a4520969a995c62d9ee65ff7e1a) put your endpoints into an enum (but doesn't enforce this), some open source projects do the same, and Alamofire itself [suggests](https://github.com/Alamofire/Alamofire#crud--authorization) that you do the same. This is surprising because it doesn't seem like an optimal way to represent endpoints. Enums are great for representing concepts where mutual exclusivity of cases is important - `Optional`, `Result`, `Bool`. This is not the case with the API endpoints. There is no cohesion in [this code](https://gist.github.com/kean/a2461a4520969a995c62d9ee65ff7e1a). Instead of pulling together things that describe a single endpoint the details are scattered across the entire file. It's going to be unnecessary tedious to read, add new endpoint or edit existing ones.

My priorities are that the endpoints should be modeled in a type-safe, declarative and concise way, and it should be easy to add, use, and change them. In its simplest form `Endpoint` is described using `HTTP method`, `path`, `parameters`, and a way to decode the response:

```swift
final class Endpoint<Response> {
    let method: Method
    let path: Path
    let parameters: Parameters?
    let decode: (Data) throws -> Response

    init(method: Method = .get,
         path: Path,
         parameters: Parameters? = nil,
         decode: @escaping (Data) throws -> Response) {
        self.method = method
        self.path = path
        self.parameters = parameters
        self.decode = decode
    }
}

typealias Parameters = [String: Any]
typealias Path = String

enum Method {
    case get, post, put, patch, delete
}
```

> There are no prefixes to make the code more concise. In reality, you would either add them or put those types in a separate module.

To make endpoint initializers more concise `decode` closures can be inferred automatically (depends on the JSON mapper being used):

```swift
extension Endpoint where Response: Swift.Decodable {
    convenience init(method: Method = .get,
                     path: Path,
                     parameters: Parameters? = nil) {
        self.init(method: method, path: path, parameters: parameters) {
            try JSONDecoder().decode(Response.self, from: $0)
        }
    }
}

extension Endpoint where Response == Void {
    convenience init(method: Method = .get,
                     path: Path,
                     parameters: Parameters? = nil) {
        self.init(
            method: method,
            path: path,
            parameters: parameters,
            decode: { _ in () }
        )
    }
}
```

Let's define a couple of endpoints to see how it works in practice:

```swift
extension API {
    static func getCustomer() -> Endpoint<Customer> {
        return Endpoint(path: "customer/profile")
    }

    static func patchCustomer(firstName: String, lastName: String) -> Endpoint<Customer> {
        return Endpoint(
            method: .patch,
            path: "customer/profile",
            parameters: ["firstName" : firstName,
                         "lastName" : lastName]
        )
    }
}
```

This is not the only way to organize endpoints. My personal preference is to put endpoints in a hierarchy of namespaces (the closest thing to "emulate" namespaces in Swift are enums with no cases):

```swift
extension API {
    /* namespace */ enum Customer {
        private static let path = "customer/profile"

        static func get() -> Endpoint<App.Customer> {
            return Endpoint(path: path)
        }

        static func patch(firstName: String, lastName: String) -> Endpoint<App.Customer> {
            return Endpoint(
                method: .patch,
                path: path,
                parameters: ["firstName" : firstName,
                             "lastName" : lastName]
            )
        }
    }
}
```

This approach closely resembles the way REST models resources, but it has it's downsides too - it's less auto-complete friendly, and it introduces a bunch of new types (enums) into the system.


## Client

The next question is how to actually perform the requests for those endpoints - `Client`. It takes an endpoint parameters, fills the rest of the defaults including the access token, base URL, etc, and carry out the request using an underlying `Alamofire.SessionManager`:

```swift
final class Client: ClientProtocol {
    private let manager: Alamofire.SessionManager
    private let baseURL = URL(string: "<your_server_base_url>")!
    private let queue = DispatchQueue(label: "<your_queue_label>")

    init(accessToken: String) {
        var defaultHeaders = Alamofire.SessionManager.defaultHTTPHeaders
        defaultHeaders["Authorization"] = "Bearer \(accessToken)"

        let configuration = URLSessionConfiguration.default

        // Add `Auth` header to the default HTTP headers set by `Alamofire`
        configuration.httpAdditionalHeaders = defaultHeaders

        self.manager = Alamofire.SessionManager(configuration: configuration)
        self.manager.retrier = OAuth2Retrier()
    }

    func request<Response>(_ endpoint: Endpoint<Response>) -> Single<Response> {
        return Single<Response>.create { observer in
            let request = self.manager.request(
                self.url(path: endpoint.path),
                method: httpMethod(from: endpoint.method),
                parameters: endpoint.parameters
            )
            request
                .validate()
                .responseData(queue: self.queue) { response in
                    let result = response.result.flatMap(endpoint.decode)
                    switch result {
                    case let .success(val): observer(.success(val))
                    case let .failure(err): observer(.error(err))
                    }
            }
            return Disposables.create {
                request.cancel()
            }
        }
    }

    private func url(path: Path) -> URL {
        return baseURL.appendingPathComponent(path)
    }
}
```

Each request is wrapped in a `Single` observable provided by `RxSwift`. I'm going to show why this is useful later in a `Usage` section.

> A `Single` is a variation of `Observable` that, instead of emitting a series of elements, is always guaranteed to emit either a single element or an error. The common use case of `Single` is to wrap HTTP requests. See [Traits](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Traits.md#single) for more info.

The `OAuth2Retrier` type is responsible for refreshing access tokens:

```swift
private class OAuth2Retrier: Alamofire.RequestRetrier {
    func should(_ manager: SessionManager, retry request: Request, with error: Error, completion: @escaping RequestRetryCompletion) {
        if (error as? AFError)?.responseCode == 401 {
            // TODO: implement your Auth2 refresh flow
            // See https://github.com/Alamofire/Alamofire#adapting-and-retrying-requests 
        }
        completion(false, 0)
    }
}
```


## Usage

We now have everything in place to start using our API endpoints. Let's create a client and start a request.

```swift
let client = Client(accessToken: "<access_token>")
_ = client.request(API.Customer.get())
_ = client.request(API.Customer.patch(firstName: "First", lastName: "Last"))
```

> In general, the actual net network calls are only made from Model layer (e.g. `API.Customer` would only be used directly inside `CustomerService` class).

Each of the requests return [cold observables](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/HotAndColdObservables.md). So nothing is going to happen until someone subscribes to the observable. I'm not going to dive into details about RxSwift and how to use it. The article assumes that you are already familiar with it. However, what I would like to do is show a couple of examples of why observables are so useful and why it's a good idea to use them in a networking layer.

**Load two resources simultaneously and continue only when both requests were successful**

Suppose you need to load two entities form the backed simultaneously and only then you would be able to continue. Without Rx it would be an exercise of manually managing state of both of those requests. With Rx [`combineLatest`](http://reactivex.io/documentation/operators/combinelatest.html) operator it's a single line:

```swift
let dependencies = Observable.combineLatest(
    client.request(API.Categories.get()),
    client.request(API.Customer.get())
)
```

**Chain two requests**

Suppose you may want to upload an image to the resource service and the patch a customer entry with an `imageId` returned by the first request. Here's how you can do it with Rx:

```swift
_ = resourseService.upload(image).flatMap { imageId in
    customerService.patch(avatarId: imageId)
}
```

**Implement autocomplete field**

This is a classic example of Rx which you would find in almost any reactive library:

```swift
let isBusy = ActivityIndicator()
self.suggestions = input
    .throttle(0.3)
    .distinctUntilChanged()
    .flatMapLatest { input in
        guard input.characters.count > 1 else { return .just([]) }
        return locationService.autocomplete(input)
            .trackActivity(isBusy)
            .asDriver(onErrorJustReturn: [])
}
```

There are a bunch of other benefits of using Rx. For example:

- Tracking activity of multiple network requests executing at the same time can be done automatically with [ActivityIndicator](https://github.com/RxSwiftCommunity/RxSwiftUtilities) class
- Dispose bags allow you to easily tie the lifetime of your network operations to lifetime of other objects (e.g. view controllers)


## Type-Safe Authorization Scopes

Another interesting problem is how to model authorization scopes using Swift. The web service API which I'm currently working with has two levels of authorizations: 

- `Guest` - the lowest level of authorization, has access only to a handful of methods
- `Customer` - superset of `Guest` in terms of permissions

I'd like to represent those concepts using Swift type system. I want to be able to "say" that "this API client is initialized with a guest token", and "that API endpoint can only be performed by customer", and the compiler should prevent me from performing endpoints with customer scope using a client initialized with a guest token.

The simplest way to implement this is to create a specific type to represent each of those concepts (e.g. `GuestEndpoint`, `CustomerClient`). However, we can do something different. Another way to achieve this is by leveraging Swift generics system. What I would do is create two separate [phantom types](https://rustbyexample.com/generics/phantom.html) that represent each of authorization scopes (`Scope.Guest` and `Scope.Customer`).

> A phantom type parameter is one that doesn't show up at runtime, but is checked statically (and only) at compile time. Types can use extra generic type parameters to act as markers or to perform type checking at compile time. These extra parameters hold no storage values, and have no runtime behavior.

Then I would wrap API client and each of the endpoints in generic wrappers (`AuthorizedClient<Authorization>` and `AuthorizedEndpoint<Authorization, Response>` respectively) which I would parameterize with concrete scopes:

```swift
struct Scope {
    enum Guest {}
    enum Customer {}
}

struct AuthorizedEndpoint<Authorization, Response> {
    fileprivate let raw: Endpoint<Response>
    init(raw: Endpoint<Response>) { self.raw = raw }
}

struct AuthorizedClient<Authorization> {
    fileprivate let raw: ClientProtocol
    init(raw: ClientProtocol) { self.raw = raw }
}
```

We now take advantage of [extensions with generic where clauses](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Generics.html#//apple_ref/doc/uid/TP40014097-CH26-ID553) to represent permissions in `AuthorizedClient`:

```swift
// API client with a `Guest` authorization can only perform requests
// with a lowest (`Guest`) authorization scope.
extension AuthorizedClient where Authorization == Scope.Guest {
    func request<Response>(_ endpoint: AuthorizedEndpoint<Scope.Guest, Response>) -> Single<Response> {
        return raw.request(endpoint.raw)
    }
}

// API client with a `Customer` authorization can perform requests
// with both `Guest` and `Customer` authorization scopes.
extension AuthorizedClient where Authorization == Scope.Customer {
    func request<Response>(_ endpoint: AuthorizedEndpoint<Scope.Guest, Response>) -> Single<Response> {
        return raw.request(endpoint.raw)
    }

    func request<Response>(_ endpoint: AuthorizedEndpoint<Scope.Customer, Response>) -> Single<Response> {
        return raw.request(endpoint.raw)
    }
}
```

Let's use new `AuthorizedEndpoint` type to represent a couple of endpoints:

```swift
extension API {
    static func postFeedback(email: String, message: String) -> AuthorizedEndpoint<Scope.Guest, Void> {
        return AuthorizedEndpoint(
            raw: Endpoint(
                method: .post,
                path: "/feedback",
                parameters: ["email": email,
                             "message": message]
            )
        )
    }

    static func getCustomer() -> AuthorizedEndpoint<Scope.Customer, Customer> {
        return AuthorizedEndpoint(
            raw: Endpoint(path: "/customer/profile")
        )
    }
}
```

Now let's put those new types into test:

```swift
let accessToken = "customer_auth_token"
let client = AuthorizedClient<Scope.Guest>(raw: Client(accessToken: accessToken))

// This line gets compiled successfully.
_ = client.request(API.postFeedback(email: "email", message: "message"))

// And this doesn't.
_ = client.request(API.getCustomerProfile())
```

Great, this works just as expected! I'm still experimenting with this approach and haven't yet put this in production code. However, it looks really promising. The Swift type system is obviously so much more powerful than Objective-C, but I can't stop myself from wanting more.

## Conclusion

I hope you've enjoyed this! Please keep in mind that all of those decisions were made in the context of the app and the web service for which it was implemented. I would advise against adopting any of the described decisions without careful consideration.

It's hard to imagine now that there was a time when the only relatively easy-to-use tool for iOS developers to do networking was `ASIHTTPRequest`. It's just amazing that we now have so many great tools at our disposal. It's now up to us to make the best use of them.

> All of the code from the post is [available here](https://gist.github.com/kean/64b9fc0963fd430594fdb3eb848bccf3) (requires Swift 4).

{% include references-start.html %}

- [Alamofire](https://github.com/Alamofire/Alamofire)
- [RxSwift](https://github.com/ReactiveX/RxSwift)
- [JSONShootout: Compare Several Swift JSON Mappers](https://github.com/bwhiteley/JSONShootout#marshal)
- [RxSwift: Traits](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Traits.md)
- [Rust by Example: phantom Type Parameters](https://rustbyexample.com/generics/phantom.html)
- [The Swift Programming Languages: Extensions with Generic Where Clauses](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Generics.html#//apple_ref/doc/uid/TP40014097-CH26-ID553)

{% include references-end.html %}