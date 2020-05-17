---
layout: post
title: "Regex, Prologue: Grammar"
description: Using a formal grammar to describe a regular expression language
date: 2019-09-09 9:00:00 +0300
category: programming
tags: programming
permalink: /post/regex-grammar
uuid: 733fbc2a-7f42-4bad-b8a9-026733e110fd
---

Before writing a parser for a regular expression *language*, one needs to define the rules of the language, or *grammar*. When you learn a new (programming) language, like [Swift](https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html) or [Regex](https://docs.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-language-quick-reference), you typically go through a guide written in a natural language like English. This is a great way to learn a language, but it's not the optimal way to describe it.

{% include ad-hor.html %}

> **Note.** This artcle outlines some of the theory behind languages and grammars. If that's not what you are interested in, you can skip it and jump straight into [parsing]({{ site.url }}/post/regex-parser). I would still recommend revisiting this article later.

The syntaxes of most programming languages in use today are defined through [*formal grammars*](https://en.wikipedia.org/wiki/Formal_grammar). If you've opened the [Swift Language Reference](https://docs.swift.org/swift-book/ReferenceManual/AboutTheLanguageReference.html) before, you've probably seen the blocks like this:

> **Grammar of Structure Declaration**
>
> <i>struct-declaration</i> → attributes<i><sub>opt</sub></i> access-level-modifier<i><sub>opt</sub></i> <b>struct</b> struct-name generic-parameter-clause<i><sub>opt</sub></i> type-inheritance-clause<i><sub>opt</sub></i> generic-where-clause<i><sub>opt</sub></i> struct-body<br>
> <i>struct-name</i> → identifier<br>
> <i>struct-body</i> → <b>{</b> struct-members<i><sub>opt</sub></i> <b>}</b><br><br>
> <i>struct-members</i> → struct-member struct-members<i><sub>opt</sub></i><br>
> <i>struct-member</i> → declaration | compiler-control-statement<br>

This is an example of a formal grammar described using [a special notation](https://docs.swift.org/swift-book/ReferenceManual/AboutTheLanguageReference.html). It might seem a bit scary at first, like any unfamiliar mathematical notation does. But bear with me. By the end of this article you are going to learn what formal grammars are, why you need them, and how to read and write different formal grammar notations. Finally, we will define a subset of a regular expression language grammar using an [Extended Backus-Naur form](https://en.wikipedia.org/wiki/Backus–Naur_form) (EBNF).

## Grammar

What exactly is *grammar*? The term comes from the [*linguistics*](https://en.wikipedia.org/wiki/Linguistics), which is the scientific study of language. If you ever studied a foreign language, you know that grammar consists of rules that describe the language. For example, English grammar contains rules like this:

> <i>Adjective can modify nouns. An adjective can be put before the noun.</i>

A formal linguist or a computer scientist would describe this rule using a *formal, generative grammar*:

```swift
ModifiedNoun ::= Adjective Noun | Adjective ", " ModifiedNoun
Adjective ::= "fast" | "blue"
Noun ::= "car" | "sky"
```

A diagram generated using [Railroad Diagram Generator](https://www.bottlecaps.de/rr/ui) for this grammar:

<img class="AdaptiveImage2" src="{{ site.url }}/images/posts/regex/modified_noun.png" style="max-height: 137.5px;">

But what does it mean for grammar to be *generative*? You can think of a generative grammar as a recipe for constructing sentences in the language. Let's see an example. You start with `ModifiedNoun` – it's the first symbol on the list. Then you begin substituting:

```swift
ModifiedNoun // Start
Adjective Noun // Select the first alternative
fast Noun // Substitute Adjective
fast car // Substitute Noun
```

That was easy. What happens if you substitute another noun?

```swift
fast sky
```

Hmm, this doesn't make any sense! But this is fine. A grammar does not describe the meaning of the strings or what can be done with them in whatever context—only their form.

The first option of `ModifiedNoun` was simple, but the other one is defined using `ModifiedNoun` itself! This is a recursive symbol. Namely, it is *right-recursive* because a *non-terminal* symbol `ModifiedNoun` appears on the right side.

```swift
ModifiedNoun // Start
Adjective, ModifiedNoun // Select the second alternative
fast, ModifiedNoun // Substitute Adjective
fast, Adjective Noun // Select the first alternative
fast, blue Noun // Substitute Adjective
fast, blue car // Substitute Noun
```

Awesome! We just saw a recursive *production rule* in action. I've used a few new terms in this section: *non-terminal*, *production rule*, etc. I think it's time to formally define "formal grammar".

## Formal Grammar

In the formalization of generative grammars first proposed by Noam Chomsky in the 1950s, a grammar `G` is defined the following way:

> A grammar is formally defined as the tuple `(N, Σ, P, S)`:
>
> - A finite set `N` of *non-terminal symbols*
> - A finite set `Σ` of *terminal symbols* that is disjoint from `N`
> - A finite set `P` of *production rules*
> - A *start symbol* `S`

*Terminal symbols* are literal symbols which cannot be changed using the rules. After you finish applying all the rules, it terminates with an output string which consists *only* of terminal symbols, e.g. `Adjective` in the *production rule*:

```swift
Adjective ::= "fast" | "blue"
```

*Non-terminal symbols* are those symbols which should be replaced, e.g. `ModifiedNoun` in the *production rule*:

```swift
ModifiedNoun ::= Adjective Noun | Adjective, ModifiedNoun`
```

> [**Chomsky Hierarchy**](https://en.wikipedia.org/wiki/Chomsky_hierarchy) *(Additional Reading)*
>
> In the 1950s, Noan Chomsky created a hierarchy of grammars. There are four categories of formal grammars in the *Chomsky Hierarchy*, spanning from Type 0, the most general, to Type 3, the most restrictive. Each layer is different in terms of the restriction applied to the production rules, the class of language it generates, the type of automation that recognizes it.
>
> Most programming languages are defined using Type 2 grammars, or [Context-Free Grammars](https://en.wikipedia.org/wiki/Context-free_grammar) (CFG). In context-free grammars, all production rules must have only one (non-terminal) symbol on the left-hand side. It essentially means that regardless in which context the non-terminal symbol appears, it should be also interpreted the same way. Context-free grammars can be recognized using [pushdown automata](https://en.wikipedia.org/wiki/Pushdown_automaton).
>
> Another important type is Type 3 which describes [Regular Languages](https://en.wikipedia.org/wiki/Regular_grammar) which can be recognized using [Finite State Automation](https://en.wikipedia.org/wiki/Finite-state_machine). Sounds familiar?

Now that we know what formal grammars are, let's take a closer look at the notations that are used to describe them.

## Notation Styles

There are several different styles of notations for context-free grammars. One of the most widely used is [Extended Backus-Naur form (EBNF)](https://en.wikipedia.org/wiki/Extended_Backus–Naur_form). It is a combination of standard Backus-Naur Form with an addition of the constructs often found in regular expressions like quantifiers, and some other extensions.

Unfortunately, there is no single EBNF standard. There is a [ISO/IEC 14977 standard](https://www.cl.cam.ac.uk/~mgk25/iso-14977.pdf) which defines EBNF, but some people [don't recommend using it](https://dwheeler.com/essays/dont-use-iso-14977-ebnf.html). Ultimately, it's up to which tools are available on your platform/language and which notation they support. I'm going to stick with a [W3C notation](https://www.w3.org/TR/2010/REC-xquery-20101214/#EBNFNotation) because it is supported by [the tool](https://www.bottlecaps.de/rr/ui) which I'm going to use to generate railroad diagrams.

> **History of Backus-Naur Form** *(Additional Reading)*
>
> In the middle 1950s, computer scientists began to design high–level programming languages and build their compilers. The first two major successes were FORTRAN (FORmula TRANslator), developed by the IBM corporation in the United States, and ALGOL (ALGOrithmic Language), sponsored by a consortium of North American and European countries. John Backus led the effort to develop FORTRAN. He then became a member of the ALGOL design committee, where he studied the problem of describing the syntax of these programming languages simply and precisely.
>
> Backus invented a notation (based on the work of logician Emil Post) that was simple, precise, and powerful enough to describe the syntax of any programming language. Using this notation, a programmer or compiler can determine whether a program is syntactically correct: whether it adheres to the grammar and punctuation rules of the programming language. Peter Naur, as editor of the ALGOL report, popularized this notation by using it to describe the complete syntax of ALGOL. In their honor, this notation is called Backus–Naur Form (BNF). 
>
> [https://www.ics.uci.edu/~pattis/misc/ebnf2.pdf](https://www.ics.uci.edu/~pattis/misc/ebnf2.pdf)

## Regex Grammar

I think we are now ready to define an actual regex grammar! You can start either constructing it bottom-up or top-down, depending on what you prefer. I will start doing it top-down.
	
The regex begins with an optional [Start of String of Line Anchor](https://docs.microsoft.com/en-us/dotnet/standard/base-types/anchors-in-regular-expressions#Start) followed the expression.

<img class="AdaptiveImage2" src="{{ site.url }}/images/posts/regex/grammar_regex.png" style="max-height:115px">

```swift
Regex ::= StartOfStringAnchor? Expression
StartOfStringAnchor ::= "^"
```

Now goes probably the most challenging part. How to define an expression? An expression can be as simple as a single character match. And it can be as complex as multiple nested groups with alternations and quantifiers. Each of these subexpressions can be followed by one another. For example, an expression might start with a single character and can be followed by a group: "`a(bc)*`".

Let's formalize it:

<img class="AdaptiveImage2" src="{{ site.url }}/images/posts/regex/grammar_expression.png" style="max-height:98px;">

<img class="AdaptiveImage2" src="{{ site.url }}/images/posts/regex/grammar_expression_item.png" style="max-height:220px;">

```swift
/* Anything that can be on one side of the alternation. */
Subexpression ::= SubexpressionItem+
SubexpressionItem
	::= Match
	  | Group
	  | Anchor
	  | Backreference
```

I'm not going to go through all of the possible expression items but here are a few examples. Let's start with `Group`. Group is a recursive construct – a group may contain other groups, or actually any subexpression.

<img class="AdaptiveImage2" src="{{ site.url }}/images/posts/regex/grammar_group.png" style="max-height:115px;">

```swift
Group ::= "(" GroupNonCapturingModifier? Expression ")" Quantifier?
GroupNonCapturingModifier ::= "?:"
```

This is the first example where we used recursion. When a parser encounters an opening parentheses, it checks whether the group is [Non-Capturing](https://docs.microsoft.com/en-us/dotnet/standard/base-types/grouping-constructs-in-regular-expressions#noncapturing_group) and then parses a subexpression until in encounters a matching closing parentheses.

> Note: not all parse generator support *indirect* recursion like we used here.

I think you get the idea. You can find a complete regex [grammar](https://github.com/kean/Regex/blob/master/grammar.ebnf) along with a [railroad diagram](https://kean.github.io/Regex/grammar-diagram.xhtml) on GitHub.

## What's Next

In the next article I will write a parser based on [the grammar](https://kean.github.io/Regex/grammar-diagram.xhtml). There are multiple ways to do that. You can write a parser by hand or generate it. A parser can use a top-down or bottom-up approach. You can use [parser combinators](https://en.wikipedia.org/wiki/Parser_combinator), etc. I will look at all of these options.

<div class="Any-vertInsets">
<a href="{{ site.url }}/post/regex-parser">
  <div class="PrimaryButton">
    Continue Reading »
  </div>
</a>
</div>

<div class="References" markdown="1">

## References

1. Dick Grune, Ceriel J.H. Jacobs (2008), [**Parsing Techniques**](https://dickgrune.com/Books/PTAPG_2nd_Edition/), VU University Amsterdam, Amsterdam, The Netherlands
2. Gunther Rademacher, [**Railroad Diagram Generator**](https://www.bottlecaps.de/rr/ui)
3. W3C (2008), [**EBNF Notation**](https://www.w3.org/TR/REC-xml/#sec-notation)

</div>