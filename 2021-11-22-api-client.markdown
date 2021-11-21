---
layout: post
title: "Web API Client in Swift"
subtitle: Building a modern web API client in Swift using Async/Await and Actors
description: Building a modern web API client in Swift using Async/Await and Actors
date: 2021-11-21 09:00:00 -0500
category: programming
tags: programming
favorite: true
permalink: /post/new-api-client
uuid: 681de9e6-02c7-4e55-ab73-630625edac63
image:
  path: /images/posts/api-client/cover.png
  height: 1200
  width: 600
---

It's been more than four years since my previous [API Client in Swift (Archived)](/post/api-client) post. A lot has changed since then. With the addition of Async/Await and Actors, it's now easier and more fun than ever to design custom web API clients in Swift. The new version went through a radical redesign and I can't wait to share more about it.
 
I'm going to focus on REST APIs and use [GitHub API](https://docs.github.com/en/rest/guides/getting-started-with-the-rest-api) as an example. Before I jump in, here is a quick look at the final result:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="nv">client</span> <span class="o">=</span> <span class="kc">APIClient</span><span class="p">(</span><span class="nv">host</span><span class="p">:</span> <span class="s">"api.github.com"</span><span class="p">)</span>

<span class="c1">// Using the client directly</span>
<span class="k">let</span> <span class="nv">user</span><span class="p">:</span> <span class="kc">User</span> <span class="o">=</span> <span class="k">try</span> <span class="k">await</span> <span class="n">client</span><span class="o">.</span><span class="kt">send</span><span class="p">(</span><span class="o">.</span><span class="kt">get</span><span class="p">(</span><span class="s">"/user"</span><span class="p">))</span>
<span class="k">try</span> <span class="k">await</span> <span class="n">client</span><span class="o">.</span><span class="kt">send</span><span class="p">(</span><span class="o">.</span><span class="kt">post</span><span class="p">(</span><span class="s">"/user/emails"</span><span class="p">,</span> <span class="kt">body</span><span class="p">:</span> <span class="p">[</span><span class="s">"kean@example.com"</span><span class="p">]))</span>

<span class="c1">// Using a predefined API definition</span>
<span class="k">let</span> <span class="nv">repos</span> <span class="o">=</span> <span class="k">try</span> <span class="k">await</span> <span class="n">client</span><span class="o">.</span><span class="kt">send</span><span class="p">(</span><span class="kc">Resources</span><span class="o">.</span><span class="kt">users</span><span class="p">(</span><span class="s">"kean"</span><span class="p">)</span><span class="o">.</span><span class="kt">repos</span><span class="o">.</span><span class="kt">get</span><span class="p">)</span>
</code></pre></div></div>

> The code from this article is available at [kean/APIClient](https://github.com/kean/APIClient).
{:.info}

## Overview

Every backend has its quirks and usually requires a client optimized for it. This article is a collection of ideas that you can use for writing your clients. The goal is to use the minimum number of abstractions and make the code easy to understand and extend.

The previous version of the client built using [Alamofire](https://github.com/Alamofire/Alamofire) and [RxSwift](https://github.com/ReactiveX/RxSwift) was a good design at the time, but with the recent Swift changes, I don’t think you need dependencies anymore.  I’m going with Apple technologies exclusively for this project: [URLSession](https://developer.apple.com/documentation/foundation/urlsession), [Codable](https://developer.apple.com/documentation/swift/codable), [Async/Await](https://developer.apple.com/videos/play/wwdc2021/10132/), and [Actors](https://developer.apple.com/videos/play/wwdc2021/10133/).

## Implementing a Client

Let’s start by defining a type for representing requests – you’ll need it for API definitions later. It can be as complicated as you need, but here is a good starting point:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">public</span> <span class="kd">struct</span> <span class="kc">Request</span><span class="o">&lt;</span><span class="kc">Response</span><span class="o">&gt;</span> <span class="p">{</span>
    <span class="k">var</span> <span class="nv">method</span><span class="p">:</span> <span class="xc">String</span>
    <span class="k">var</span> <span class="nv">path</span><span class="p">:</span> <span class="xc">String</span>
    <span class="k">var</span> <span class="nv">query</span><span class="p">:</span> <span class="p">[</span><span class="xc">String</span><span class="p">:</span> <span class="xc">String</span><span class="p">]?</span>
    <span class="k">var</span> <span class="nv">body</span><span class="p">:</span> <span class="kc">AnyEncodable</span><span class="p">?</span>
<span class="p">}</span>
</code></pre></div></div>

To make it easier to define REST APIs, let's add a few factory methods which are also going to help us [erase](https://www.donnywals.com/understanding-type-erasure-in-swift/) generic types that we don't need to carry around:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">extension</span> <span class="kc">Request</span> <span class="p">{</span>
    <span class="kd">public</span> <span class="kd">static</span> <span class="kd">func</span> <span class="nf">get</span><span class="p">(</span><span class="n">_</span> <span class="nv">path</span><span class="p">:</span> <span class="xc">String</span><span class="p">,</span> <span class="nv">query</span><span class="p">:</span> <span class="p">[</span><span class="xc">String</span><span class="p">:</span> <span class="xc">String</span><span class="p">]?</span> <span class="o">=</span> <span class="k">nil</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="kc">Request</span> <span class="p">{</span>
        <span class="kc">Request</span><span class="p">(</span><span class="kt">method</span><span class="p">:</span> <span class="s">"GET"</span><span class="p">,</span> <span class="kt">path</span><span class="p">:</span> <span class="n">path</span><span class="p">,</span> <span class="kt">query</span><span class="p">:</span> <span class="n">query</span><span class="p">)</span>
    <span class="p">}</span>
    
    <span class="kd">public</span> <span class="kd">static</span> <span class="kd">func</span> <span class="n">post</span><span class="o">&lt;</span><span class="o">U</span><span class="p">:</span> <span class="xc">Encodable</span><span class="o">&gt;</span><span class="p">(</span><span class="n">_</span> <span class="nv">path</span><span class="p">:</span> <span class="xc">String</span><span class="p">,</span> <span class="nv">body</span><span class="p">:</span> <span class="o">U</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="kc">Request</span> <span class="p">{</span>
        <span class="kc">Request</span><span class="p">(</span><span class="kt">method</span><span class="p">:</span> <span class="s">"POST"</span><span class="p">,</span> <span class="kt">path</span><span class="p">:</span> <span class="n">path</span><span class="p">,</span> <span class="kt">body</span><span class="p">:</span> <span class="kc">AnyEncodable</span><span class="p">(</span><span class="n">body</span><span class="p">))</span>
    <span class="p">}</span>
    
    <span class="kd">public</span> <span class="kd">static</span> <span class="kd">func</span> <span class="n">patch</span><span class="o">&lt;</span><span class="o">U</span><span class="p">:</span> <span class="xc">Encodable</span><span class="o">&gt;</span><span class="p">(</span><span class="n">_</span> <span class="nv">path</span><span class="p">:</span> <span class="xc">String</span><span class="p">,</span> <span class="nv">body</span><span class="p">:</span> <span class="o">U</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="kc">Request</span> <span class="p">{</span>
        <span class="kc">Request</span><span class="p">(</span><span class="kt">method</span><span class="p">:</span> <span class="s">"PATCH"</span><span class="p">,</span> <span class="kt">path</span><span class="p">:</span> <span class="n">path</span><span class="p">,</span> <span class="kt">body</span><span class="p">:</span> <span class="kc">AnyEncodable</span><span class="p">(</span><span class="n">body</span><span class="p">))</span>
    <span class="p">}</span>
    
    <span class="c1">// ...</span>
<span class="p">}</span>
</code></pre></div></div>

What do Swift developers love more than anything else? Type-safety. By separating each HTTP method, `Request` stops invalid parameter combinations at compile time. For example, you [shouldn't pass](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET) body to GET requests, and `URLSession` throws an error at runtime if you try. With the `Request`, you can't.

To execute the requests, you use `APIClient`. It's a small wrapper on top of `URLSession` that is easy to modify or extend. You initialize it with a `host` making it easy to change the environments at runtime.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">public</span> <span class="k">actor</span> <span class="kc">APIClient</span> <span class="p">{</span>
    <span class="kd">private</span> <span class="k">let</span> <span class="nv">session</span><span class="p">:</span> <span class="xc">URLSession</span>
    <span class="kd">private</span> <span class="k">let</span> <span class="nv">host</span><span class="p">:</span> <span class="xc">String</span>
    <span class="c1">// ..</span>
    
    <span class="kd">public</span> <span class="k">init</span><span class="p">(</span><span class="nv">host</span><span class="p">:</span> <span class="xc">String</span><span class="p">,</span>
                <span class="nv">configuration</span><span class="p">:</span> <span class="xc">URLSessionConfiguration</span> <span class="o">=</span> <span class="o">.</span><span class="xv">default</span><span class="p">,</span>
                <span class="nv">delegate</span><span class="p">:</span> <span class="kc">APIClientDelegate</span><span class="p">?</span> <span class="o">=</span> <span class="k">nil</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">host</span> <span class="o">=</span> <span class="n">host</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">session</span> <span class="o">=</span> <span class="xc">URLSession</span><span class="p">(</span><span class="nv">configuration</span><span class="p">:</span> <span class="n">configuration</span><span class="p">)</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">delegate</span> <span class="o">=</span> <span class="n">delegate</span> <span class="p">??</span> <span class="kc">DefaultAPIClientDelegate</span><span class="p">()</span>
    <span class="p">}</span>
</code></pre></div></div>

There are two types of `send()` method – one for `Decodable` types and one for `Void` that isn't decodable.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">extension</span> <span class="kc">APIClient</span> <span class="p">{</span>
    <span class="kd">public</span> <span class="kd">func</span> <span class="n">send</span><span class="o">&lt;</span><span class="o">T</span><span class="p">:</span> <span class="xc">Decodable</span><span class="o">&gt;</span><span class="p">(</span><span class="n">_</span> <span class="nv">request</span><span class="p">:</span> <span class="kc">Request</span><span class="o">&lt;</span><span class="o">T</span><span class="o">&gt;</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="o">T</span> <span class="p">{</span>
        <span class="k">try</span> <span class="k">await</span> <span class="kt">send</span><span class="p">(</span><span class="n">request</span><span class="p">,</span> <span class="kt">serializer</span><span class="o">.</span><span class="kt">decode</span><span class="p">)</span>
    <span class="p">}</span>
    
    <span class="kd">public</span> <span class="kd">func</span> <span class="nf">send</span><span class="p">(</span><span class="n">_</span> <span class="nv">request</span><span class="p">:</span> <span class="kc">Request</span><span class="o">&lt;</span><span class="xc">Void</span><span class="o">&gt;</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="xc">Void</span> <span class="p">{</span>
        <span class="k">try</span> <span class="k">await</span> <span class="kt">send</span><span class="p">(</span><span class="n">request</span><span class="p">,</span> <span class="p">{</span> <span class="n">_</span> <span class="k">in</span> <span class="p">()</span> <span class="p">})</span>
    <span class="p">}</span>

    <span class="kd">private</span> <span class="kd">func</span> <span class="n">send</span><span class="o">&lt;</span><span class="o">T</span><span class="o">&gt;</span><span class="p">(</span><span class="n">_</span> <span class="nv">request</span><span class="p">:</span> <span class="kc">Request</span><span class="o">&lt;</span><span class="o">T</span><span class="o">&gt;</span><span class="p">,</span> 
                         <span class="n">_</span> <span class="nv">decode</span><span class="p">:</span> <span class="kd">@escaping</span> <span class="p">(</span><span class="xc">Data</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="o">T</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="o">T</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">request</span> <span class="o">=</span> <span class="k">try</span> <span class="k">await</span> <span class="kt">makeRequest</span><span class="p">(</span><span class="kt">for</span><span class="p">:</span> <span class="n">request</span><span class="p">)</span>
        <span class="k">let</span> <span class="p">(</span><span class="nv">data</span><span class="p">,</span> <span class="nv">response</span><span class="p">)</span> <span class="o">=</span> <span class="k">try</span> <span class="k">await</span> <span class="kt">send</span><span class="p">(</span><span class="n">request</span><span class="p">)</span>
        <span class="k">try</span> <span class="kt">validate</span><span class="p">(</span><span class="kt">response</span><span class="p">:</span> <span class="n">response</span><span class="p">,</span> <span class="kt">data</span><span class="p">:</span> <span class="n">data</span><span class="p">)</span>
        <span class="k">return</span> <span class="k">try</span> <span class="k">await</span> <span class="nf">decode</span><span class="p">(</span><span class="n">data</span><span class="p">)</span>
    <span class="p">}</span>

    <span class="c1">// The final implementation is a bit more complicated because it supports auto-retries</span>
    <span class="kd">private</span> <span class="kd">func</span> <span class="nf">send</span><span class="p">(</span><span class="n">_</span> <span class="nv">request</span><span class="p">:</span> <span class="xc">URLRequest</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="xc">Data</span><span class="p">,</span> <span class="xc">URLResponse</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">try</span> <span class="k">await</span> <span class="kt">session</span><span class="o">.</span><span class="xv">data</span><span class="p">(</span><span class="xv">for</span><span class="p">:</span> <span class="n">request</span><span class="p">,</span> <span class="xv">delegate</span><span class="p">:</span> <span class="k">nil</span><span class="p">)</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

`APIClient` takes full advantage of async/await, including the new `URLSession` [async/await APIs](https://developer.apple.com/videos/play/wwdc2021/10095/). It also performs encoding and decoding on a separate actor reducing the amount of work done on the `APIClient`.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">private</span> <span class="k">actor</span> <span class="kc">Serializer</span> <span class="p">{</span>
    <span class="kd">func</span> <span class="n">encode</span><span class="o">&lt;</span><span class="o">T</span><span class="p">:</span> <span class="xc">Encodable</span><span class="o">&gt;</span><span class="p">(</span><span class="n">_</span> <span class="nv">entity</span><span class="p">:</span> <span class="o">T</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="xc">Data</span> <span class="p">{</span>
        <span class="k">try</span> <span class="xc">JSONEncoder</span><span class="p">()</span><span class="o">.</span><span class="xv">encode</span><span class="p">(</span><span class="n">entity</span><span class="p">)</span>
    <span class="p">}</span>
    
    <span class="kd">func</span> <span class="n">decode</span><span class="o">&lt;</span><span class="o">T</span><span class="p">:</span> <span class="xc">Decodable</span><span class="o">&gt;</span><span class="p">(</span><span class="n">_</span> <span class="nv">data</span><span class="p">:</span> <span class="xc">Data</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="o">T</span> <span class="p">{</span>
        <span class="k">try</span> <span class="xc">JSONDecoder</span><span class="p">()</span><span class="o">.</span><span class="xv">decode</span><span class="p">(</span><span class="o">T</span><span class="o">.</span><span class="k">self</span><span class="p">,</span> <span class="xv">from</span><span class="p">:</span> <span class="n">data</span><span class="p">)</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

> **Actor Runtime**
>
> Initially, I wasn’t sure whether using actors to send work to a different “thread” was a good idea – the primary role of actors is to protect mutable state. But if you watch [Swift concurrency: Update a sample app (WWDC21)](https://developer.apple.com/videos/play/wwdc2021/10194/), you'll see on minute 36 [Ben Cohen](https://twitter.com/AirspeedSwift) suggesting replacing a serial `DispatchQueue` with an actor to perform work in background. It's not exactly the same thing, because actor runtime [doesn't use](https://developer.apple.com/videos/play/wwdc2021/10254/) GCD, but it's close enough for this scenario.
{:.info}

Getting back to `Codable`, I think we, as a developer community, have finally tackled the challenge of parsing JSON in Swift. So I'm not going to focus on it. If you want to learn more, I wrote [a post](https://kean.blog/post/codable-tips-and-tricks) a couple of years ago with some `Codable` tips – most are still relevant today. For example, it has [some ideas](https://kean.blog/post/codable-tips-and-tricks#5-encoding-patch-parameter) on encoding PATCH, which is useful for REST APIs.

The rest of the code is relatively straightforward. I'm using [`URLComponents`](https://developer.apple.com/documentation/foundation/urlcomponents) to create URLs, which is important because it [percent-encodes](https://en.wikipedia.org/wiki/Percent-encoding) the parts of the URLs that need it.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">func</span> <span class="n">makeRequest</span><span class="o">&lt;</span><span class="o">T</span><span class="o">&gt;</span><span class="p">(</span><span class="o">for</span> <span class="nv">request</span><span class="p">:</span> <span class="kc">Request</span><span class="o">&lt;</span><span class="o">T</span><span class="o">&gt;</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="xc">URLRequest</span> <span class="p">{</span>
    <span class="k">let</span> <span class="nv">url</span> <span class="o">=</span> <span class="k">try</span> <span class="kt">makeURL</span><span class="p">(</span><span class="kt">path</span><span class="p">:</span> <span class="n">request</span><span class="o">.</span><span class="kt">path</span><span class="p">,</span> <span class="kt">query</span><span class="p">:</span> <span class="n">request</span><span class="o">.</span><span class="kt">query</span><span class="p">)</span>
    <span class="k">return</span> <span class="k">try</span> <span class="k">await</span> <span class="kt">makeRequest</span><span class="p">(</span><span class="kt">url</span><span class="p">:</span> <span class="n">url</span><span class="p">,</span> <span class="kt">method</span><span class="p">:</span> <span class="n">request</span><span class="o">.</span><span class="kt">method</span><span class="p">,</span> <span class="kt">body</span><span class="p">:</span> <span class="n">request</span><span class="o">.</span><span class="kt">body</span><span class="p">)</span>
<span class="p">}</span>

<span class="kd">func</span> <span class="nf">makeURL</span><span class="p">(</span><span class="nv">path</span><span class="p">:</span> <span class="xc">String</span><span class="p">,</span> <span class="nv">query</span><span class="p">:</span> <span class="p">[</span><span class="xc">String</span><span class="p">:</span> <span class="xc">String</span><span class="p">]?)</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="xc">URL</span> <span class="p">{</span>
    <span class="k">guard</span> <span class="k">let</span> <span class="nv">url</span> <span class="o">=</span> <span class="xc">URL</span><span class="p">(</span><span class="xv">string</span><span class="p">:</span> <span class="n">path</span><span class="p">),</span>
          <span class="k">var</span> <span class="nv">components</span> <span class="o">=</span> <span class="xc">URLComponents</span><span class="p">(</span><span class="xv">url</span><span class="p">:</span> <span class="n">url</span><span class="p">,</span> <span class="xv">resolvingAgainstBaseURL</span><span class="p">:</span> <span class="k">false</span><span class="p">)</span> <span class="k">else</span> <span class="p">{</span>
        <span class="k">throw</span> <span class="xc">URLError</span><span class="p">(</span><span class="o">.</span><span class="xv">badURL</span><span class="p">)</span>
    <span class="p">}</span>
    <span class="k">if</span> <span class="n">path</span><span class="o">.</span><span class="xv">starts</span><span class="p">(</span><span class="xv">with</span><span class="p">:</span> <span class="s">"/"</span><span class="p">)</span> <span class="p">{</span>
        <span class="n">components</span><span class="o">.</span><span class="xv">scheme</span> <span class="o">=</span> <span class="s">"https"</span>
        <span class="n">components</span><span class="o">.</span><span class="xv">host</span> <span class="o">=</span> <span class="n">host</span>
    <span class="p">}</span>
    <span class="k">if</span> <span class="k">let</span> <span class="nv">query</span> <span class="o">=</span> <span class="n">query</span> <span class="p">{</span>
        <span class="n">components</span><span class="o">.</span><span class="xv">queryItems</span> <span class="o">=</span> <span class="n">query</span><span class="o">.</span><span class="xv">map</span><span class="p">(</span><span class="xc">URLQueryItem</span><span class="o">.</span><span class="xv">init</span><span class="p">)</span>
    <span class="p">}</span>
    <span class="k">guard</span> <span class="k">let</span> <span class="nv">url</span> <span class="o">=</span> <span class="n">components</span><span class="o">.</span><span class="xv">url</span> <span class="k">else</span> <span class="p">{</span>
        <span class="k">throw</span> <span class="xc">URLError</span><span class="p">(</span><span class="o">.</span><span class="xv">badURL</span><span class="p">)</span>
    <span class="p">}</span>
    <span class="k">return</span> <span class="n">url</span>
<span class="p">}</span>
</code></pre></div></div>

The client works with JSON, so its sets the respective ["Content-Type"](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type) and ["Accept"](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept) HTTP header values automatically.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">func</span> <span class="nf">makeRequest</span><span class="p">(</span><span class="nv">url</span><span class="p">:</span> <span class="xc">URL</span><span class="p">,</span> <span class="nv">method</span><span class="p">:</span> <span class="xc">String</span><span class="p">,</span> <span class="nv">body</span><span class="p">:</span> <span class="kc">AnyEncodable</span><span class="p">?)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="xc">URLRequest</span> <span class="p">{</span>
    <span class="k">var</span> <span class="nv">request</span> <span class="o">=</span> <span class="xc">URLRequest</span><span class="p">(</span><span class="xv">url</span><span class="p">:</span> <span class="n">url</span><span class="p">)</span>
    <span class="n">request</span><span class="o">.</span><span class="xv">httpMethod</span> <span class="o">=</span> <span class="n">method</span>
    <span class="k">if</span> <span class="k">let</span> <span class="nv">body</span> <span class="o">=</span> <span class="n">body</span> <span class="p">{</span>
        <span class="n">request</span><span class="o">.</span><span class="xv">httpBody</span> <span class="o">=</span> <span class="k">try</span> <span class="k">await</span> <span class="kt">serializer</span><span class="o">.</span><span class="kt">encode</span><span class="p">(</span><span class="n">body</span><span class="p">)</span>
        <span class="n">request</span><span class="o">.</span><span class="xv">setValue</span><span class="p">(</span><span class="s">"application/json"</span><span class="p">,</span> <span class="xv">forHTTPHeaderField</span><span class="p">:</span> <span class="s">"Content-Type"</span><span class="p">)</span>
    <span class="p">}</span>
    <span class="n">request</span><span class="o">.</span><span class="xv">setValue</span><span class="p">(</span><span class="s">"application/json"</span><span class="p">,</span> <span class="xv">forHTTPHeaderField</span><span class="p">:</span> <span class="s">"Accept"</span><span class="p">)</span>
    <span class="k">return</span> <span class="n">request</span>
<span class="p">}</span>
</code></pre></div></div>

Being able to use async functions right inside the `makeRequest()` method without all the boilerplate of using closures is just pure joy.

The only remaining bit is a `validate()` method where I check that the response status code is acceptable. It also gives a delegate a chance to decide what error to throw. Most APIs will have a standard JSON format for errors – this is a good place to parse it.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code> <span class="kd">func</span> <span class="nf">validate</span><span class="p">(</span><span class="nv">response</span><span class="p">:</span> <span class="xc">URLResponse</span><span class="p">,</span> <span class="nv">data</span><span class="p">:</span> <span class="xc">Data</span><span class="p">)</span> <span class="k">throws</span> <span class="p">{</span>
    <span class="k">guard</span> <span class="k">let</span> <span class="nv">httpResponse</span> <span class="o">=</span> <span class="n">response</span> <span class="k">as?</span> <span class="xc">HTTPURLResponse</span> <span class="k">else</span> <span class="p">{</span> <span class="k">return</span> <span class="p">}</span>
    <span class="k">if</span> <span class="o">!</span><span class="p">(</span><span class="mi">200</span><span class="o">..&lt;</span><span class="mi">300</span><span class="p">)</span><span class="o">.</span><span class="xv">contains</span><span class="p">(</span><span class="n">httpResponse</span><span class="o">.</span><span class="xv">statusCode</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">throw</span> <span class="kt">delegate</span><span class="o">.</span><span class="kt">client</span><span class="p">(</span><span class="k">self</span><span class="p">,</span> <span class="kt">didReceiveInvalidResponse</span><span class="p">:</span> <span class="n">httpResponse</span><span class="p">,</span> <span class="kt">data</span><span class="p">:</span> <span class="n">data</span><span class="p">)</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

And that’s it, a baseline `APIClient`. It already covers most of the needs of most users and there many ways to extend it.

## Extending the Client

Getting the basics right is usually easy, but what about more advanced use-cases?

### User Authorization

Every authorization system has its quirks. If you use [OAuth 2.0](https://oauth.net/2/) or a similar protocol, you need to send an access token with every request. One of the common ways is by setting an ["Authorization"](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization) header.

One of the advantages of `Alamofire` is the infrastructure for adapting and retrying requests which is [often used for authorization](https://www.avanderlee.com/swift/authentication-alamofire-request-adapter/). Reimplementing it with callbacks is no fun, but with async/await, it's a piece of cake.

The first piece is the `client(_:willSendRequest:)` delegate method that you can use to "sign" the requests.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">final</span> <span class="kd">class</span> <span class="kc">YourAPIClientDelegate</span><span class="p">:</span> <span class="kc">APIClientDelegate</span> <span class="p">{</span>
    <span class="kd">func</span> <span class="nf">client</span><span class="p">(</span><span class="n">_</span> <span class="nv">client</span><span class="p">:</span> <span class="kc">APIClient</span><span class="p">,</span> <span class="n">willSendRequest</span> <span class="nv">request</span><span class="p">:</span> <span class="k">inout</span> <span class="xc">URLRequest</span><span class="p">)</span> <span class="p">{</span>
        <span class="n">request</span><span class="o">.</span><span class="xv">setValue</span><span class="p">(</span><span class="s">"Bearer: </span><span class="se">\(</span><span class="kt">accessToken</span><span class="se">)</span><span class="s">"</span><span class="p">,</span> <span class="xv">forHTTPHeaderField</span><span class="p">:</span> <span class="s">"Authorization"</span><span class="p">)</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

> If you look at `setValue(_:forHTTPHeaderField:)` [documentation](https://developer.apple.com/documentation/foundation/urlrequest/2011447-setvalue), you'll see a list of [Reserved HTTP Headers](https://developer.apple.com/documentation/foundation/nsurlrequest#1776617) that you shouldn't set manually. "Authorization" is one of them... `URLSession` supports seemingly every authorization mechanism except for the most popular one - OAuth 2.0. So setting an "Authorization" header manually is still [the least worst](https://developer.apple.com/forums/thread/89811) solution.
{:.warning}

> The `client(_:willSendRequest:)` method is also a good way to provide default headers, like ["User-Agent"](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent). But you can also provide fields that don't change using [`httpAdditionalHeaders`](https://developer.apple.com/documentation/foundation/urlsessionconfiguration/1411532-httpadditionalheaders) property of [`URLSessionConfiguration`](https://developer.apple.com/documentation/foundation/urlsessionconfiguration).
{:.info}

If your access tokens are short-lived, it is important to implement a proper refresh flow. Again, easy with async/await.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">extension</span> <span class="kc">APIClient</span> <span class="p">{</span>
    <span class="kd">private</span> <span class="kd">func</span> <span class="nf">send</span><span class="p">(</span><span class="n">_</span> <span class="nv">request</span><span class="p">:</span> <span class="xc">URLRequest</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="xc">Data</span><span class="p">,</span> <span class="xc">URLResponse</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">do</span> <span class="p">{</span>
            <span class="k">return</span> <span class="k">try</span> <span class="k">await</span> <span class="kt">actuallySend</span><span class="p">(</span><span class="n">request</span><span class="p">)</span> 
        <span class="p">}</span> <span class="k">catch</span> <span class="p">{</span>
            <span class="k">guard</span> <span class="k">await</span> <span class="kt">delegate</span><span class="o">.</span><span class="kt">shouldClientRetry</span><span class="p">(</span><span class="k">self</span><span class="p">,</span> <span class="kt">withError</span><span class="p">:</span> <span class="n">error</span><span class="p">)</span> <span class="k">else</span> <span class="p">{</span> <span class="k">throw</span> <span class="n">error</span> <span class="p">}</span>
            <span class="k">return</span> <span class="k">try</span> <span class="k">await</span> <span class="kt">actuallySend</span><span class="p">(</span><span class="n">request</span><span class="p">)</span>
        <span class="p">}</span>
    <span class="p">}</span>

    <span class="kd">private</span> <span class="kd">func</span> <span class="nf">actuallySend</span><span class="p">(</span><span class="n">_</span> <span class="nv">request</span><span class="p">:</span> <span class="xc">URLRequest</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="xc">Data</span><span class="p">,</span> <span class="xc">URLResponse</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">var</span> <span class="nv">request</span> <span class="o">=</span> <span class="n">request</span>
        <span class="n">delegate</span><span class="o">.</span><span class="kt">client</span><span class="p">(</span><span class="k">self</span><span class="p">,</span> <span class="kt">willSendRequest</span><span class="p">:</span> <span class="o">&amp;</span><span class="n">request</span><span class="p">)</span>
        <span class="k">return</span> <span class="k">try</span> <span class="k">await</span> <span class="kt">session</span><span class="o">.</span><span class="xv">data</span><span class="p">(</span><span class="xv">for</span><span class="p">:</span> <span class="n">request</span><span class="p">,</span> <span class="xv">delegate</span><span class="p">:</span> <span class="k">nil</span><span class="p">)</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

I count four [suspension points](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md#suspension-points). Now think how more complicated and error-prone it would've been to implement it with callbacks.

Now all you need is to implement a `shouldClientRetry(_:withError:)` method in your existing delegate and add the token refresh logic.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">final</span> <span class="kd">class</span> <span class="kc">YourAPIClientDelegate</span><span class="p">:</span> <span class="kc">APIClientDelegate</span> <span class="p">{</span>
    <span class="kd">func</span> <span class="nf">shouldClientRetry</span><span class="p">(</span><span class="n">_</span> <span class="nv">client</span><span class="p">:</span> <span class="kc">APIClient</span><span class="p">,</span> <span class="n">withError</span> <span class="nv">error</span><span class="p">:</span> <span class="xc">Error</span><span class="p">)</span> <span class="k">async</span> <span class="o">-&gt;</span> <span class="k">Bool</span> <span class="p">{</span>
        <span class="k">if</span> <span class="k">case</span> <span class="o">.</span><span class="kt">unacceptableStatusCode</span><span class="p">(</span><span class="k">let</span> <span class="nv">status</span><span class="p">)</span> <span class="o">=</span> <span class="p">(</span><span class="n">error</span> <span class="k">as?</span> <span class="kc">YourError</span><span class="p">),</span> <span class="n">status</span> <span class="o">==</span> <span class="mi">401</span> <span class="p">{</span>
            <span class="k">return</span> <span class="k">await</span> <span class="kt">refreshAccessToken</span><span class="p">()</span>
        <span class="p">}</span>
        <span class="k">return</span> <span class="k">false</span>
    <span class="p">}</span>
    
    <span class="kd">private</span> <span class="kd">func</span> <span class="nf">refreshAccessToken</span><span class="p">()</span> <span class="k">async</span> <span class="o">-&gt;</span> <span class="xc">Bool</span> <span class="p">{</span>
        <span class="c1">// TODO: Refresh access token</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

> The client might call `shouldClientRetry(_:withError:)`  multiple times (once for each failed request). Make sure to coalesce the requests to refresh the token. Also, don't forget the scenario with an expired refresh token.
{:.warning}

I didn't show this code in the original `APIClient` implementation to not over-complicate things, but the project you find at GitHub already [supports it](https://github.com/kean/APIClient).

> If you are thinking about using auto-retries for connectivity issues, consider using [`waitsForConnectivity`](https://developer.apple.com/documentation/foundation/urlsessionconfiguration/2908812-waitsforconnectivity) instead. If the request does fail with a network issue, it's usually best to communicate an error to the user. With [`NWPathMonitor`](https://developer.apple.com/documentation/network/nwpathmonitor) you can still monitor the connection to your server and retry automatically.
{:.info}

### Client Authorization

On top of authorizing the user, most services will also have a way of authorizing the client. If it's an API key, you can set it using the same way as an "Authorization" header. You may also want to [obfuscate](https://nshipster.com/secrets/) it, but remember that client secrecy is impossible.

Another less common but more interesting approach is [mTLS](https://www.cloudflare.com/learning/access-management/what-is-mutual-tls/) (mutual TLS) where it's not just the server sending a certificate – the client does too. One of the advantages of using certificates is that the secret (private key) never leaves the device and doesn't travel over the network.

`URLSession` supports mTLS natively and it's easy to implement, even when using the new `async/await` API (thanks, Apple!).

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">final</span> <span class="kd">class</span> <span class="kc">YourTaskDelegate</span><span class="p">:</span> <span class="kc">URLSessionTaskDelegate</span> <span class="p">{</span>
    <span class="kd">func</span> <span class="nf">urlSession</span><span class="p">(</span><span class="n">_</span> <span class="nv">session</span><span class="p">:</span> <span class="xc">URLSession</span><span class="p">,</span> <span class="n">didReceive</span> <span class="nv">challenge</span><span class="p">:</span> <span class="xc">URLAuthenticationChallenge</span><span class="p">)</span>
        <span class="k">async</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="xc">URLSession</span><span class="o">.</span><span class="xc">AuthChallengeDisposition</span><span class="p">,</span> <span class="xc">URLCredential</span><span class="p">?)</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">protectionSpace</span> <span class="o">=</span> <span class="n">challenge</span><span class="o">.</span><span class="xv">protectionSpace</span>
        <span class="k">if</span> <span class="n">protectionSpace</span><span class="o">.</span><span class="xv">authenticationMethod</span> <span class="o">==</span> <span class="xc">NSURLAuthenticationMethodServerTrust</span> <span class="p">{</span>
            <span class="c1">// You'll probably need to create it somewhere else.</span>
            <span class="k">let</span> <span class="nv">credential</span> <span class="o">=</span> <span class="xc">URLCredential</span><span class="p">(</span><span class="xv">identity</span><span class="p">:</span> <span class="o">...</span><span class="p">,</span> <span class="xv">certificates</span><span class="p">:</span> <span class="o">...</span><span class="p">,</span> <span class="xv">persistence</span><span class="p">:</span> <span class="o">...</span><span class="p">)</span>
            <span class="k">return</span> <span class="p">(</span><span class="o">.</span><span class="xv">useCredential</span><span class="p">,</span> <span class="n">credential</span><span class="p">)</span>
        <span class="p">}</span> <span class="k">else</span> <span class="p">{</span>
            <span class="k">return</span> <span class="p">(</span><span class="o">.</span><span class="xv">performDefaultHandling</span><span class="p">,</span> <span class="k">nil</span><span class="p">)</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

The main challenge with mTLS is getting the private key to the client. You can embed an obfuscated `.p12` file in the app and obfuscate it, making it hard to discover, but it's still not impenetrable.

### SSL Pinning

The task delegate method for handling the server challenges is also a good place for [implementing SSL pinning](https://www.raywenderlich.com/1484288-preventing-man-in-the-middle-attacks-in-ios-with-ssl-pinning). 

There are [multiple](https://owasp.org/www-community/controls/Certificate_and_Public_Key_Pinning) ways how to approach this, especially when it comes to _what_ to pin. If you pin leaf certificates, you sign up for constant maintenance because certificates need to be rotated. The simplest option seems to be pinning CA certificates or public keys. Starting with iOS 14, it can be [done very easily](https://developer.apple.com/news/?id=g9ejcf8y) by adding the keys to the app's plist file.

### HTTP Caching

Caching is a great way to improve application performance and end-user experience. Developers often overlook [HTTP cache](https://tools.ietf.org/html/rfc7234) natively [supported](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html) by `URLSession` To enable HTTP caching the server sends special HTTP headers along with the request. Here is an example:

```
HTTP/1.1 200 OK
Cache-Control: public, max-age=3600
Expires: Mon, 26 Jan 2016 17:45:57 GMT
Last-Modified: Mon, 12 Jan 2016 17:45:57 GMT
ETag: "686897696a7c876b7e"
```

This response is cacheable and will be *fresh* for 1 hour. When it becomes *stale*, the client validates it by making a *conditional* request using the `If-Modified-Since` and/or `If-None-Match` headers. If the response is still fresh the server returns status code [`304 Not Modified`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/304) to instruct the client to use cached data, or it would return `200 OK` with a new data otherwise.

> By default, `URLSession` uses `URLCache.shared` with a small disk and memory capacity. You might not know it, but already be taking advantage of HTTP caching.
{:.info}

HTTP caching is a flexible system where both the server and the client get a say over what gets cached and how. With HTTP, a server can set restrictions on which responses are cacheable, set an expiration age for responses, provide validators (`ETag`, `Last-Modified`) to check stale responses, force revalidation on each request, and more.

`URLSession` (and `URLCache`) support HTTP caching out of the box. There is a [set of requirements](https://developer.apple.com/documentation/foundation/nsurlsessiondatadelegate/1411612-urlsession) for a response to be cached. It's not just the server that has control. For example, you can use [`URLRequest.CachePolicy`](https://developer.apple.com/documentation/foundation/nsurlrequest/cachepolicy) to modify caching behavior from the client. You can easily extend `APIClient` to support it if needed.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">extension</span> <span class="kc">APIClient</span> <span class="p">{</span>
    <span class="kd">func</span> <span class="n">send</span><span class="o">&lt;</span><span class="o">Response</span><span class="o">&gt;</span><span class="p">(</span>
        <span class="n">_</span> <span class="nv">request</span><span class="p">:</span> <span class="kc">Request</span><span class="o">&lt;</span><span class="o">Response</span><span class="o">&gt;</span><span class="p">,</span>
        <span class="nv">cachePolicy</span><span class="p">:</span> <span class="xc">URLRequest</span><span class="o">.</span><span class="xc">CachePolicy</span> <span class="o">=</span> <span class="o">.</span><span class="xv">useProtocolCachePolicy</span>
    <span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="o">Response</span> <span class="k">where</span> <span class="kc">Response</span><span class="p">:</span> <span class="xc">Decodable</span> <span class="p">{</span>
        <span class="c1">// Pass the policy to the URLRequest</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

### Downloads

By default, the `send()` method of `APIClient` uses [`data(for:delegate:)`](https://developer.apple.com/documentation/foundation/urlsession/3767352-data) method from `URLSession`, but with the existing design, you can easily extend it to support other types of tasks, such as downloads.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">extension</span> <span class="kc">APIClient</span> <span class="p">{</span>
    <span class="kd">public</span> <span class="kd">func</span> <span class="nf">download</span><span class="p">(</span><span class="n">_</span> <span class="nv">request</span><span class="p">:</span> <span class="kc">Request</span><span class="o">&lt;</span><span class="xc">Void</span><span class="o">&gt;</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="xc">URL</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">request</span> <span class="o">=</span> <span class="k">try</span> <span class="k">await</span> <span class="kt">makeRequest</span><span class="p">(</span><span class="nv">for</span><span class="p">:</span> <span class="n">request</span><span class="p">)</span>
        <span class="k">let</span> <span class="p">(</span><span class="nv">url</span><span class="p">,</span> <span class="nv">response</span><span class="p">)</span> <span class="o">=</span> <span class="k">try</span> <span class="k">await</span> <span class="kt">session</span><span class="o">.</span><span class="xv">download</span><span class="p">(</span><span class="xv">for</span><span class="p">:</span> <span class="n">request</span><span class="p">,</span> <span class="xv">delegate</span><span class="p">:</span> <span class="k">nil</span><span class="p">)</span>
        <span class="k">try</span> <span class="kt">validate</span><span class="p">(</span><span class="kt">response</span><span class="p">:</span> <span class="n">response</span><span class="p">)</span>
        <span class="k">return</span> <span class="n">url</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

And if the `Request` type is not working for you, it's easy to extend too. You can also just as easily add new request types, or even `URLRequest` directly if you need to. Although for a typical REST API, `Request` is all you are ever going to need.

### Environments

`APIClient` is initialized with a `host`, making it easy to switch between the environments in runtime. I typically have a debug menu in the apps with all sorts of debug settings, including an environment picker – it's fast and easy to build with SwiftUI. I added the code generating this screen in [a gist](https://gist.github.com/kean/846eba91cf3471071760ec0db3ddc23e).

<img width="452px" class="NewScreenshot" src="{{ site.url }}/images/posts/api-client/02.png">

## Defining an API

For smaller apps, using `APIClient` directly without creating an API definition can be acceptable. But it's generally a good idea to define the available APIs somewhere to reduce the clutter in the rest of the code and remove possible duplication.

I've tried a few different approaches for defining APIs using `APIClient`, but couldn't decide which one was the best. They all have their pros and cons and are often just a matter of personal preference.

REST APIs are designed around resources. One of the ideas I had was to create a separate type to represent each of the resources and expose HTTP methods that are available on them. It works best for APIs that closely follow ([REST API design](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design0). GitHub API is a great example of a REST API, so that's why I used it in the examples.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">public</span> <span class="kd">enum</span> <span class="kc">Resources</span> <span class="p">{}</span>

<span class="c1">// MARK: - /users/{username}</span>

<span class="kd">extension</span> <span class="kc">Resources</span> <span class="p">{</span>
    <span class="kd">public</span> <span class="kd">static</span> <span class="kd">func</span> <span class="nf">users</span><span class="p">(</span><span class="n">_</span> <span class="nv">name</span><span class="p">:</span> <span class="xc">String</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="kc">UsersResource</span> <span class="p">{</span>
        <span class="kc">User</span><span class="p">(</span><span class="nv">path</span><span class="p">:</span> <span class="s">"/users/</span><span class="se">\(</span><span class="n">name</span><span class="se">)</span><span class="s">"</span><span class="p">)</span>
    <span class="p">}</span>
    
    <span class="kd">public</span> <span class="kd">struct</span> <span class="kc">UsersResource</span> <span class="p">{</span>
        <span class="kd">public</span> <span class="k">let</span> <span class="nv">path</span><span class="p">:</span> <span class="xc">String</span>

        <span class="kd">public</span> <span class="k">var</span> <span class="nv">get</span><span class="p">:</span> <span class="kc">Request</span><span class="o">&lt;</span><span class="kc">User</span><span class="o">&gt;</span> <span class="p">{</span> <span class="o">.</span><span class="kt">get</span><span class="p">(</span><span class="kt">path</span><span class="p">)</span> <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>

<span class="c1">// MARK: - /users/{username}/repos</span>

<span class="kd">extension</span> <span class="kc">Resources</span><span class="o">.</span><span class="kc">UsersResource</span> <span class="p">{</span>
    <span class="kd">public</span> <span class="k">var</span> <span class="nv">repos</span><span class="p">:</span> <span class="kc">ReposResource</span> <span class="p">{</span> <span class="kc">ReposResource</span><span class="p">(</span><span class="nv">path</span><span class="p">:</span> <span class="n">path</span> <span class="o">+</span> <span class="s">"/repos"</span><span class="p">)</span> <span class="p">}</span>
    
    <span class="kd">public</span> <span class="kd">struct</span> <span class="kc">ReposResource</span> <span class="p">{</span>
        <span class="kd">public</span> <span class="k">let</span> <span class="nv">path</span><span class="p">:</span> <span class="xc">String</span>

        <span class="kd">public</span> <span class="k">var</span> <span class="nv">get</span><span class="p">:</span> <span class="kc">Request</span><span class="o">&lt;</span><span class="p">[</span><span class="kc">Repo</span><span class="p">]</span><span class="o">&gt;</span> <span class="p">{</span> <span class="o">.</span><span class="kt">get</span><span class="p">(</span><span class="kt">path</span><span class="p">)</span> <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

Usage:

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="nv">repos</span> <span class="o">=</span> <span class="k">try</span> <span class="k">await</span> <span class="n">client</span><span class="o">.</span><span class="kt">send</span><span class="p">(</span><span class="kc">Resources</span><span class="o">.</span><span class="kt">users</span><span class="p">(</span><span class="s">"kean"</span><span class="p">)</span><span class="o">.</span><span class="kt">repos</span><span class="o">.</span><span class="kt">get</span><span class="p">)</span>
</code></pre></div></div>

This API is visually appealing, but it can be a bit tedious to write and less discoverable than simply listing all available calls. I’m also still a bit cautious about over-using nesting. I used to avoid it in the past, but the recent improvements to the Xcode code completion made working with nested APIs much easier. But again, this is just an example.

> I've seen [suggestions](https://github.com/Moya/Moya/blob/master/docs/Examples/Basic.md) to model APIs as an enum where each propery has a separate switch. This isn't ideal because you are setting yourself for merge conflicts, and it harder to read and modify than other approaches. When you add a new call, you should ideally only need to make a change in one place.
{:.warning}

## Tools

It's not just Swift that's getting better. The modern tooling is also fantastic.

### OpenAPI

Remember the infamous [Working with JSON in Swift](https://developer.apple.com/swift/blog/?id=37) post from 2016? Yeah, we are way ahead now. Ask your backend developers to provide [OpenAPI](https://www.openapis.org/) spec for their APIs and use code generation to create `Codable` entities.

[Postman](https://www.postman.com) and [Paw](https://paw.cloud) are great ways to explore APIs. You can import an OpenAPI spec and it will generate a Collection for you so that you have all the APIs always at your fingertips. GitHub API that's I'm using in the examples also [recently git](https://github.blog/2020-07-27-introducing-githubs-openapi-description/) its own OpenAPI spec. You can [download](https://github.com/github/rest-api-description) it and import it into Postman to try it.

<img class="NewScreenshot" src="{{ site.url }}/images/posts/api-client/03.png">

Generating `Codable` entities is also extremely easy. There are [a ton of tools](https://openapi.tools) available for working with OpenAPI specs. For example, you can use [swagger-codegen](https://github.com/swagger-api/swagger-codegen). It supports all popular languages, including Swift. Here is an example of one of the generated types.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">public</span> <span class="kd">struct</span> <span class="kc">PublicUser</span><span class="p">:</span> <span class="xc">Codable</span> <span class="p">{</span>
    <span class="kd">public</span> <span class="k">var</span> <span class="nv">login</span><span class="p">:</span> <span class="xc">String</span>
    <span class="kd">public</span> <span class="k">var</span> <span class="nv">id</span><span class="p">:</span> <span class="xc">Int</span>
    <span class="kd">public</span> <span class="k">var</span> <span class="nv">nodeId</span><span class="p">:</span> <span class="xc">String</span>
    <span class="c1">// ...</span>

    <span class="kd">public</span> <span class="k">init</span><span class="p">(</span><span class="nv">login</span><span class="p">:</span> <span class="xc">String</span><span class="p">,</span> <span class="nv">id</span><span class="p">:</span> <span class="xc">Int</span><span class="p">,</span> <span class="nv">nodeId</span><span class="p">:</span> <span class="xc">String</span><span class="p">,</span> <span class="cm">/* ... */</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">login</span> <span class="o">=</span> <span class="n">login</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">id</span> <span class="o">=</span> <span class="n">id</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">nodeId</span> <span class="o">=</span> <span class="n">nodeId</span>
        <span class="c1">// ...</span>
    <span class="p">}</span>

    <span class="kd">public</span> <span class="kd">enum</span> <span class="kc">CodingKeys</span><span class="p">:</span> <span class="xc">String</span><span class="p">,</span> <span class="xc">CodingKey</span> <span class="p">{</span> 
        <span class="k">case</span> <span class="n">login</span>
        <span class="k">case</span> <span class="n">id</span> <span class="o">=</span> <span class="s">"id"</span>
        <span class="k">case</span> <span class="n">nodeId</span> <span class="o">=</span> <span class="s">"node_id"</span>
        <span class="c1">// ...</span>
<span class="p">}</span>
</code></pre></div></div>

> By default, it generates structs, which is not always ideal – passing large structs around can be expensive. Fortunately, there is a way to generate classes instead (see `useModelClasses` option):
> <br/><br/>
>
>     swagger-codegen generate \
>         -i api.github.com.yaml \
>         -o ~/github-api
>         -l swift5
>         --additional-properties useModelClasses=true
{:.warning}

By default, [swagger-codegen](https://github.com/swagger-api/swagger-codegen) generates not just the `Codable` entities but also attempts to generate an entire API client. Unfortunately, it leaves much to be desired. I suggest only using it for generating entities.

> If you don’t have an OpenAPI spec, you can turn a sample JSON into a Codable struct using [quicktype.io](https://quicktype.io/) and tweak it manually to make sure it complete it.
{:.info}

### Logging

[Pulse](https://kean.blog/pulse/home) is a powerful logging system for Apple Platforms. In addition to regular messages, it records `URLSession` tasks and allows you to inspect logs right from your iOS app using [Pulse Console](https://kean.blog/pulse/guides/pulseui). You can also share and view them in a dedicated [macOS app](https://kean.blog/pulse/guides/pulse-pro). 

<a href="/images/posts/api-client/08-full.png">
<img class="NewScreenshot" src="{{ site.url }}/images/posts/api-client/08.png">
</a>

It's easy to integrate Pulse in an `APIClient` by modifying the `send()` method and providing a custom task delegate. The advantage of doing it at that level is that you don't need to worry about TLS and SSL pinning, and you collect more information than a typical network proxy does thanks to the direct access to `URLSession`.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">import</span> <span class="o">PulseCore</span>

<span class="kd">public</span> <span class="k">actor</span> <span class="kc">APIClient</span> <span class="p">{</span>
    <span class="kd">private</span> <span class="k">let</span> <span class="nv">logger</span> <span class="o">=</span> <span class="kt">LoggerStore</span><span class="o">.</span><span class="k">default</span>

    <span class="kd">public</span> <span class="kd">func</span> <span class="nf">send</span><span class="p">(</span><span class="n">_</span> <span class="nv">request</span><span class="p">:</span> <span class="kc">URLRequest</span><span class="p">)</span> <span class="k">async</span> <span class="k">throws</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="xc">Data</span><span class="p">,</span> <span class="xc">URLResponse</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">let</span> <span class="nv">delegate</span> <span class="o">=</span> <span class="kc">TaskDelegate</span><span class="p">()</span>
        <span class="kd">func</span> <span class="nf">log</span><span class="p">(</span><span class="nv">response</span><span class="p">:</span> <span class="xc">URLResponse</span><span class="p">?</span> <span class="o">=</span> <span class="k">nil</span><span class="p">,</span> <span class="nv">data</span><span class="p">:</span> <span class="xc">Data</span><span class="p">?</span> <span class="o">=</span> <span class="k">nil</span><span class="p">,</span> <span class="nv">error</span><span class="p">:</span> <span class="xc">Error</span><span class="p">?</span> <span class="o">=</span> <span class="k">nil</span><span class="p">)</span> <span class="p">{</span>
            <span class="n">logger</span><span class="o">.</span><span class="kt">storeRequest</span><span class="p">(</span><span class="n">delegate</span><span class="o">.</span><span class="kt">currentRequest</span> <span class="p">??</span> <span class="n">request</span><span class="p">,</span> <span class="kt">response</span><span class="p">:</span> <span class="n">response</span><span class="p">,</span>
                <span class="kt">error</span><span class="p">:</span> <span class="n">error</span><span class="p">,</span> <span class="kt">data</span><span class="p">:</span> <span class="n">data</span><span class="p">,</span> <span class="kt">metrics</span><span class="p">:</span> <span class="n">delegate</span><span class="o">.</span><span class="n">metrics</span><span class="p">)</span>
        <span class="p">}</span>
        <span class="k">do</span> <span class="p">{</span>
            <span class="k">let</span> <span class="p">(</span><span class="nv">data</span><span class="p">,</span> <span class="nv">response</span><span class="p">)</span> <span class="o">=</span> <span class="k">try</span> <span class="k">await</span> <span class="kt">session</span><span class="o">.</span><span class="xv">data</span><span class="p">(</span><span class="xv">for</span><span class="p">:</span> <span class="n">request</span><span class="p">,</span> <span class="xv">delegate</span><span class="p">:</span> <span class="n">delegate</span><span class="p">)</span>
            <span class="kt">log</span><span class="p">(</span><span class="kt">response</span><span class="p">:</span> <span class="n">response</span><span class="p">,</span> <span class="kt">data</span><span class="p">:</span> <span class="n">data</span><span class="p">)</span>
            <span class="k">return</span> <span class="p">(</span><span class="n">data</span><span class="p">,</span> <span class="n">response</span><span class="p">)</span>
        <span class="p">}</span> <span class="k">catch</span> <span class="p">{</span>
            <span class="kt">log</span><span class="p">(</span><span class="kt">error</span><span class="p">:</span> <span class="n">error</span><span class="p">)</span>
            <span class="k">throw</span> <span class="n">error</span>
        <span class="p">}</span>
    <span class="p">}</span>
<span class="p">}</span>

<span class="kd">private</span> <span class="kd">final</span> <span class="kd">class</span> <span class="kc">TaskDelegate</span><span class="p">:</span> <span class="xc">NSObject</span><span class="p">,</span> <span class="xc">URLSessionTaskDelegate</span> <span class="p">{</span>
    <span class="k">var</span> <span class="nv">currentRequest</span><span class="p">:</span> <span class="xc">URLRequest</span><span class="p">?</span>
    <span class="k">var</span> <span class="nv">metrics</span><span class="p">:</span> <span class="xc">URLSessionTaskMetrics</span><span class="p">?</span>
        
    <span class="kd">func</span> <span class="nf">urlSession</span><span class="p">(</span><span class="n">_</span> <span class="nv">session</span><span class="p">:</span> <span class="xc">URLSession</span><span class="p">,</span> <span class="nv">task</span><span class="p">:</span> <span class="xc">URLSessionTask</span><span class="p">,</span>
                    <span class="n">didFinishCollecting</span> <span class="nv">metrics</span><span class="p">:</span> <span class="xc">URLSessionTaskMetrics</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">currentRequest</span> <span class="o">=</span> <span class="n">task</span><span class="o">.</span><span class="xv">currentRequest</span>
        <span class="k">self</span><span class="o">.</span><span class="kt">metrics</span> <span class="o">=</span> <span class="n">metrics</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

> For a complete guide on using Pulse, see the [official documentation](https://kean.blog/pulse/guides/overview).
{:.info}

### Network Debugging Proxies

Tools like [Proxyman](https://proxyman.io) or [Charles](https://www.charlesproxy.com) are indispensable for debugging your apps because they allow you not only to inspect the traffic but also manipulate the requests and responses without changing your app's code.

### Mocking

My preferred way of testing ViewModels is by writing integration tests where I mock only the network responses. These tests are easy to write and maintain and they give you a lot of confidence in your app.

There are a many mocking tools to chose from. I used [`Mocker`](https://github.com/WeTransfer/Mocker) for unit-testing `APIClient` itself.

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kd">private</span> <span class="k">let</span> <span class="nv">host</span> <span class="o">=</span> <span class="s">"api.github.com"</span>

<span class="k">override</span> <span class="kd">func</span> <span class="nf">setUp</span><span class="p">()</span> <span class="p">{</span>
    <span class="k">super</span><span class="o">.</span><span class="nf">setUp</span><span class="p">()</span>
    
    <span class="k">let</span> <span class="nv">configuration</span> <span class="o">=</span> <span class="xc">URLSessionConfiguration</span><span class="o">.</span><span class="xv">default</span>
    <span class="n">configuration</span><span class="o">.</span><span class="xv">protocolClasses</span> <span class="o">=</span> <span class="p">[</span><span class="kc">MockingURLProtocol</span><span class="o">.</span><span class="k">self</span><span class="p">]</span>

    <span class="kt">client</span> <span class="o">=</span> <span class="kc">APIClient</span><span class="p">(</span><span class="kt">host</span><span class="p">:</span> <span class="n">host</span><span class="p">,</span> <span class="kt">configuration</span><span class="p">:</span> <span class="n">configuration</span><span class="p">)</span>
<span class="p">}</span>

<span class="kd">func</span> <span class="nf">testGet</span><span class="p">()</span> <span class="k">async</span> <span class="k">throws</span> <span class="p">{</span>
    <span class="c1">// Given</span>
    <span class="k">let</span> <span class="nv">url</span> <span class="o">=</span> <span class="xc">URL</span><span class="p">(</span><span class="xv">string</span><span class="p">:</span> <span class="s">"https://</span><span class="se">\(</span><span class="n">host</span><span class="se">)\(</span><span class="kc">Resources</span><span class="o">.</span><span class="kt">user</span><span class="o">.</span><span class="kt">get</span><span class="o">.</span><span class="kt">path</span><span class="se">)</span><span class="s">"</span><span class="p">)</span><span class="o">!</span>
    <span class="kc">Mock</span><span class="p">(</span><span class="kt">url</span><span class="p">:</span> <span class="n">url</span><span class="p">,</span> <span class="kt">dataType</span><span class="p">:</span> <span class="o">.</span><span class="kt">json</span><span class="p">,</span> <span class="kt">statusCode</span><span class="p">:</span> <span class="mi">200</span><span class="p">,</span> <span class="kt">data</span><span class="p">:</span> <span class="p">[</span>
        <span class="o">.</span><span class="kt">get</span><span class="p">:</span> <span class="kt">json</span><span class="p">(</span><span class="kt">named</span><span class="p">:</span> <span class="s">"user"</span><span class="p">)</span>
    <span class="p">])</span><span class="o">.</span><span class="kt">register</span><span class="p">()</span>
    
    <span class="c1">// When</span>
    <span class="k">let</span> <span class="nv">user</span> <span class="o">=</span> <span class="k">try</span> <span class="k">await</span> <span class="kt">client</span><span class="o">.</span><span class="kt">send</span><span class="p">(</span><span class="kc">Resources</span><span class="o">.</span><span class="kt">user</span><span class="o">.</span><span class="kt">get</span><span class="p">)</span>
                                        
    <span class="c1">// Then</span>
    <span class="xv">XCTAssertEqual</span><span class="p">(</span><span class="n">user</span><span class="o">.</span><span class="kt">login</span><span class="p">,</span> <span class="s">"kean"</span><span class="p">)</span>
<span class="p">}</span>
</code></pre></div></div>

### cURL

[cURL](https://curl.se) doesn't need an introduction. There is an [extension](https://gist.github.com/kean/cacb6d2e6bafa912bf130d3db1c2f116) to `URLRequest` that I borrowed from `Alamfire` that I love. It creates a cURL command for `URLRequest`.

Ideally, you should call `cURLDescription` on task's [`currentRequest`](https://developer.apple.com/documentation/foundation/urlsessiontask/1411649-currentrequest) - it's has all of the cookies and [additional HTTP headers](https://developer.apple.com/documentation/foundation/urlsessionconfiguration/1411532-httpadditionalheaders) automatically added by the system.

## Final Thoughts

Before I started using Async/Wwait in Swift, I thought it was mostly syntax sugar, but oh how wrong I was. The way Async/Await is integrated into the language is just brilliant. It solves a bunch of common problems in an elegant way. I think in the future async APIs should be defined exclusively using async functions – bye callbacks and `[weak self]`.

If you want to learn more about Async/Await and Structured Concurrency, look no further than this [WWDC21 session videos](https://developer.apple.com/videos/wwdc2021/). They are all fantastically well made, and, if you want to dive a bit deeper, read the Swift Evolution proposals.

If you just look at the surface level, let's see how much code I wrote to implement most of the features from this article:

```swift
$ cloc Sources/APIClient/.
       2 text files.
       2 unique files.
       0 files ignored.

github.com/AlDanial/cloc v 1.90  T=0.01 s (305.1 files/s, 17795.7 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Swift                            3             26             11            148
-------------------------------------------------------------------------------
SUM:                             3             26             11            148
-------------------------------------------------------------------------------
```

It is not a direct comparison, but Alamofire version 5.4.4 has 7428 lines. This is bonkers. And to think how far we've come in just about 10 years – remember [ASIHTTPRequest](https://github.com/pokeb/asi-http-request)?

{% include references-start.html %}

- [Meet async/await in Swift](https://developer.apple.com/videos/play/wwdc2021/10132/) (WWDC21)
- [Explore structured concurrency in Swift](https://developer.apple.com/videos/play/wwdc2021/10134/) (WWDC21)
- [Swift concurrency: Update a sample app](https://developer.apple.com/videos/play/wwdc2021/10194/) (WWDC21)
- [Protect mutable state with Swift actors](https://developer.apple.com/videos/play/wwdc2021/10133/) (WWDC21)
- [Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/) (WWDC21)
- [RESTful web API design](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design) (Azure / Architecture)
- [URL Loading System](https://developer.apple.com/documentation/foundation/url_loading_system) (Apple)
- [Increasing Application Performance with HTTP Cache Headers](https://devcenter.heroku.com/articles/increasing-application-performance-with-http-cache-headers) (Heroku Dev Center)
- [RFC 7234. HTTP/1.1 Caching](https://tools.ietf.org/html/rfc7234) (IETF)
- [OpenAPI Initiative](https://www.openapis.org/)
- [Introducing GitHub’s OpenAPI Description](https://github.blog/2020-07-27-introducing-githubs-openapi-description/) (GitHub Blog)
- [Certificate and Public Key Pinning](https://owasp.org/www-community/controls/Certificate_and_Public_Key_Pinning) (OWASP)
- [Authentication with signed requests in Alamofire 5](https://www.avanderlee.com/swift/authentication-alamofire-request-adapter/) (SwiftLee)
- [Secret Management on iOS](https://nshipster.com/secrets/) (NSHipster)
- [What is mutual TLS (mTLS)?](https://www.cloudflare.com/learning/access-management/what-is-mutual-tls/) (Cloudflare)
- [Preventing Man-in-the-Middle Attacks in iOS with SSL Pinning](https://www.raywenderlich.com/1484288-preventing-man-in-the-middle-attacks-in-ios-with-ssl-pinning) (Ray Wanderlich)
- [Identity Pinning: How to configure server certificates for your app](https://developer.apple.com/news/?id=g9ejcf8y) (Apple)

{% include references-end.html %}