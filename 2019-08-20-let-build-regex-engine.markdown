---
layout: post
title: Let's Build a Regex Engine
description: How to understand the language of <code>&lt;\/?[\w\s]*&gt;|&lt;.+[\W]&gt;</code>
date: 2019-08-20 9:00:00 +0300
category: programming
tags: programming
permalink: /post/lets-build-regex
uuid: ea30401f-f344-4dcc-830c-d513e2f52193
favorite: true
---

> How to understand the language of <code>&lt;\/?[\w\s]*&gt;|&lt;.+[\W]&gt;</code>

Ever wondered how regex works under the hood? How does it understand an incantation like `"<\/?[\w\s]*>|<.+[\W]>"` and magically produces a desired result? This series is going to describe exactly how it works and how to implement a feature-rich regex engine.

Now you might be wondering, why would you want to do that? Well, turns out, this is a fantastic learning opportunity. Parsers, compilers, finite automation, graphs, trees, extended grapheme clusters - it has everything! Last but not least, you get a chance to learn regex.

> - [Let's Build a Regex Engine]({{ site.url }}/post/lets-build-regex) (You are here)
> - [Regex, Prologue: Grammar]({{ site.url }}/post/regex-grammar)
> - [Regex, Part 1: Parser]({{ site.url }}/post/regex-parser)
> - [Regex, Part 2: Compiler]({{ site.url }}/post/regex-compiler)
> - [Regex, Part 3: Matcher]({{ site.url }}/post/regex-matcher)

{% include ad-hor.html %}

## Regular Expressions

Regular expression patterns can be as simple as "[`https?`](https://regex101.com/r/z6Lypq/1)", where "`?`" is a [Zero or One Quantifier](https://docs.microsoft.com/en-us/dotnet/standard/base-types/quantifiers-in-regular-expressions) and which matches either `http` or `https`. Or as complex as [this](https://regex101.com/r/95Clhd/1), which validates an email address, I think:

```
((?:[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+/=?^_`{|}~-]+)*|"(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21\x23-\x5b\x5d-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])*")@(?:(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9](?:[a-z0-9-]*[a-z0-9])?|\[(?:a(?:(2(5[0-5]|[0-4][0-9])|1[0-9][0-9]|[1-9]?[0-9]))\.){3}(?:(2(5[0-5]|[0-4][0-9])|1[0-9][0-9]|[1-9]?[0-9])|[a-z0-9-]*[a-z0-9]:(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21-\x5a\x53-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])+)\])
```

Implementing an engine capable of matching patterns like "`https?`" is easy because you can cut corners. Building one that supports the majority of the features of a modern regex engine is not.

> [**Quick Reference**](https://docs.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-language-quick-reference)
>
> If you are not familiar with regex or need a refresher, I would highly recommend going through this [Quick Reference](https://docs.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-language-quick-reference) or any other tutorial on the subject. There is going to be no introduction into regex in this series. You only need to know the basics to continue.

To make it more challenging, I also wanted it to have performance comparable to `NSRegularExpression` which uses highly optimized [ICU regex engine](http://icu-project.org/apiref/icu4c/uregex_8h_source.html) written in C.

Do you want to see how it turned out? Then take the red pill.

## Overview

There are four main pieces of the puzzle that needs to be solved to make it all work.

### Grammar

Before writing a parser for a regular expression *language*, one needs to define the rules of the language, or *grammar*. This is what [**Regex, Prologue: Grammar**]({{ site.url }}/post/regex-grammar) is about. This article outlines some of the theory behind languages and grammars. If that’s not what you are interested in, you can skip it and jump straight into [parsing](#parser). However, I would still recommend revisiting this article later.

### Parser

Regular expressions have complicated syntax with many constructs, including recursive ones, like [Capture Groups](https://docs.microsoft.com/en-us/dotnet/standard/base-types/grouping-constructs-in-regular-expressions). The pattern itself is just a raw string. To reason about it, you first need to parse it and create an abstract representation which you can easily manipulate – an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) (AST). In [**Regex, Part 1: Parser**]({{ site.url }}/post/regex-parser) we will implement such parser using Parser Combinators (or *monadic* parsers) which are a fantastic example of functional programming used to bring practical benefits.

### Compiler

In [**Regex, Part 2: Compiler**]({{ site.url }}/post/regex-compiler) we will learn about [Finite State Automation](https://en.wikipedia.org/wiki/Nondeterministic_finite_automaton) and how you can use it to represent regular expressions. In this part, some of the theory from [the part about grammars]({{ site.url }}/post/regex-grammar) will come in handy.

### Matcher

And finally, in [**Regex, Part 3: Matcher**]({{ site.url }}/post/regex-matcher) I will introduce two very different matcher algorithms. The first one uses backtracking and is similar to the algorithm found in [.NET](https://docs.microsoft.com/en-us/dotnet/standard/base-types/backtracking-in-regular-expressions) and other platforms. The second one guarantees linear time complexity and is close to what is used by [RE2](https://github.com/google/re2) created by Google.

## What's Next

I hope I got you excited! This is going to be a fun series packed with ideas and concepts. It will be useful for anyone, regardless of whether you have a CS background or not. You can find a complete implementation on [GitHub](https://github.com/kean/Regex) and if you have any comments or suggestions, please, feel free to hit me up on [Twitter](https://twitter.com/a_grebenyuk).

<div class="kb-vert-insets">
<a href="{{ site.url }}/post/regex-grammar">
  <div class="kb-primary-button">
    Continue Reading »
  </div>
</a>
</div>
