---
layout: post
title: Formatted Localizable Strings
subtitle: Using XMLParser and NSAttributedString to add support for basic formatting in localizable strings
description: Using XMLParser and NSAttributedString to add support for basic formatting in localizable strings
date: 2020-11-29 9:00:00 -0400
category: programming
tags: programming
permalink: /post/formatted-strings
uuid: 3fc81326-49ae-4daf-894f-1a3d49f457c0
---

How do you localizale a text label that has rich formatting?

<img alt="Formatting Example" width="400px" src="/images/posts/formatting/formatting.png">

Unforunately, there are not a lot of built-in options in the Apple SDKs, so people often end up using sub-optimal appoaches.

{% include ad-hor.html %}

## Poor Practices

### Concatenated Strings

One of the common approaches is to split the text into several keys and concatenate them later in code.

> BAD EXAMPLE
{:.error}

```swift
// Localizable.strings
"macbook.title-p1" = "M1 delivers up to ";
"macbook.title-p2" = "2.8x faster ";
"macbook.title-p3" = "processing performance than the ";
"macbook.title-p4" = "previous generation.";

// Usage.swift
let components = [
    NSAttributedString(string: NSLocalizedString("macbook.title-p1"), attributes: [
        .font: UIFont.systemFont(ofSize: 15)
    ]),
    NSAttributedString(string: NSLocalizedString("macbook.title-p2"), attributes: [
        .font: UIFont.boldSystemFont(ofSize: 15)
    ]),
    NSAttributedString(string: NSLocalizedString("macbook.title-p3"), attributes: [
        .font: UIFont.systemFont(ofSize: 15)
    ]),
    NSAttributedString(string: NSLocalizedString("macbook.title-p4"), attributes: [
        .link: URL(string: "https://support.apple.com/kb/SP799")!,
        .font: UIFont.systemFont(ofSize: 15),
        .underlineColor: UIColor.clear
    ])
]
let string = NSMutableAttributedString()
components.forEach(string.append)
label.attributedText = string
```

The problem is that the order of words and phrases is hard-coded. You should never make assumptions about grammar rules and a certain sentence structure. The structure of the sentence will often be completely different in another language. This will often make it impossible to translate the text.

The granularity of keys will also cause confusion during the translation. Nobody likes guessing games.


### Substring Lookup

Another (bad) approach is to use substring lookup to apply attributes.

> BAD EXAMPLE
{:.error}

```swift
// Localizable.strings
"macbook.title" = "M1 delivers up to 2.8x faster processing
    performance than the previous generation";
"macbook.title-substring-bold" = "2.8x faster";
"macbook.title-substring-link" = "previous generation";

// Usage.swift
let text = NSLocalizedString("macbook.title")
let string = NSMutableAttributedString(
    string: text,
    attributes: [.font: UIFont.systemFont(ofSize: 15)]
)
if let range = text.range(of: NSLocalizedString("macbook.title-substring-bold")) {
    string.addAttributes([
        .font: UIFont.boldSystemFont(ofSize: 15)
    ], range: NSRange(range, in: text))
}
if let range = text.range(of: NSLocalizedString("macbook.title-substring-link")) {
    string.addAttributes([
        .link: URL(string: "https://apple.com")!,
        .underlineColor: UIColor.clear
    ], range: NSRange(range, in: text))
}
label.attributedText = string
```

This is a lit bit better than the previous solution as it gives more control to translators. There still can be a problem if a substring repeats two or more times into a full string – only the first match will get the custom attributes, and it might be the wrong one.

But the major issue is that this is going to be a massive pain to translate and can often lead to an issue where someone will update the original string but will forget to update one of the substrings.

### HTML

If you search online, one of the common suggestions is to use HTML. You can either use a full-blown web view if you need to render a big portion of the screen. And for labels, there is a native way to convert [HTML to NSAttributedString](https://www.hackingwithswift.com/example-code/system/how-to-convert-html-to-an-nsattributedstring).

```swift
// Localizable.strings
"macbook.title" = "M1 delivers up to <b>2.8x faster</b> processing
    performance than the <a href='%@'>previous generation.</a>";

// Usage.swift
let format = NSLocalizedString("macbook.title")
let string = String(format: format, "https://support.apple.com/kb/SP799")
label.attributedText = try? NSAttributedString(
    data: string.data(using: .utf8) ?? Data(),
    options: [.documentType: NSAttributedString.DocumentType.html],
    documentAttributes: nil
)
```

Now, this is better, but there are some issues that you should be aware of. First, it doesn't quite produce the result we want. By default, it uses WebKit text styles.

<img alt="Formatting Example" width="400px" src="/images/posts/formatting/html.png">

One of the ways you can customize styles is by using CSS.

```swift
public extension NSAttributedString {
    static func make(html: String, size: CGFloat) -> NSAttributedString? {
        let style = """
        body {
          font-family: -apple-system;
          font-size: \(size)px;
        }
        b {
          font-weight: 600;
        }
        a {
          text-decoration: none;
        }
        """

        let template = """
        <!doctype html>
        <html>
          <head>
            <style>
              \(style)
            </style>
          </head>
          <body>
            \(html)
          </body>
        </html>
        """

        return try? NSAttributedString(
            data: template.data(using: .utf8) ?? Data(),
            options: [.documentType: NSAttributedString.DocumentType.html],
            documentAttributes: nil
        )
    }
}
```


Now it matches the expected design. There are still at least four major issues you should be aware of.

> WARNING
{:.warning}

1. It produces attributes you might not necessarily want, such as `.kern`, `.paragraphStyle`, etc. You might want to remove those. 
```
M1 delivers up to {
    NSColor = "kCGColorSpaceModelRGB 0 0 0 1 ";
    NSFont = "<UICTFont: 0x7ff6c6c11d80> font-family: \".SFUI-Regular\"; 
        font-weight: normal; font-style: normal; font-size: 15.00pt";
    NSKern = 0;
    NSParagraphStyle = "Alignment 4, LineSpacing 0, ParagraphSpacing 0,
        ParagraphSpacingBefore 0, HeadIndent 0, TailIndent 0, FirstLineHeadIndent 0,
        LineHeight 0/0, LineHeightMultiple 0, LineBreakMode 0, Tabs (\n),
        DefaultTabInterval 36, Blocks (\n), Lists (\n), BaseWritingDirection 0,
        HyphenationFactor 0, TighteningForTruncation NO, LineBreakStrategy 0";
    NSStrokeColor = "kCGColorSpaceModelRGB 0 0 0 1 ";
    NSStrokeWidth = 0;
} ...
```

2. It only supports a subset of HTML and it is not documented which
3. It can hang or [crash](http://www.openradar.me/20978452) with certain inputs. I've experienced hanging. Without documentation, it's next to impossible to properly sanitize the input. So make sure you have full control over the HTML you are feeding it.
4. If you think using HTML for this is overkill, you are right. It is very very slow. How slow? On iPhone 11 Pro parsing the HTML from this article takes 11 ms. You *will* lose frames if you use it.

## Proposed Solution

With HTML, you no longer need to concatenate strings or lookup substrings, which is great. The only real problems with HTML are lack of control and performance.

Most HTML tags (and all tags we are interested in) are valid XML. Do we need the entire power of WebKit to parse a few basic XML tags? No.

I created a repo, called [Formatting](https://github.com/kean/Formatting), where I use basic [XMLParser](https://developer.apple.com/documentation/foundation/xmlparser) to parse tags and apply respective text attributes. The entire solution takes less than 100 slocs. Here is a usage example:

```swift
let input = "M1 delivers up to <b>2.8x faster</b> processing performance
    than the <a href='%@'>previous generation.</a>"
let text = String(format: input, "https://support.apple.com/kb/SP799")

// You can define styles in one place and use them across the app
let style = FormattedStringStyle(attributes: [
    "body": [.font: UIFont.systemFont(ofSize: 15)],
    "b": [.font: UIFont.boldSystemFont(ofSize: 15)],
    "a": [.underlineColor: UIColor.clear]
])

label.attributedText = NSAttributedString(formatting: text, style: style)
```

Result using standard `UILabel`[^1]:

<img alt="Formatting Example" width="400px" src="/images/posts/formatting/formatting.png">

How fast is it? ~200 times faster than the HTML-based solution[^2].

<img alt="Formatting Example" width="600px" class="Any-responsiveCard" src="/images/posts/formatting/performance.png">

Typically I would be skeptical when I see a performance difference like this, but not in this case. If you profile and see what it's doing, there is WebView, DOM, CSS engine – all sorts of things just to apply two string attributes.

<img alt="Formatting Example" width="600px" class="Any-responsiveCard" src="/images/posts/formatting/performance-2.png">

[Formatting](https://github.com/kean/Formatting), on the other hand, only does two things: parses XML and directly applies the attributes.

> If you are already sold, you should also be aware of the following pitfals of usig XML parser:
>
> - You can't put arbitraty text in it. For example, `<`, `>`, and other reserved symbols need to be escaped.
> - If you were previously using HTML, make sure to replace the following notation `&copy;` with characters supported in XML. You can simply put a character inline, e.g. `©`.
> - Unlike HTML, you can't use ampersands (`&`) in XML attributes. This can be a problem for href links which have multiple query parameters. They have to be replaced with `&amp;`. Strickly speaking, HTML also [doesn't allow](https://t.co/sVqlfH1TCz?amp=1) the use of `&`. But the browsers have workaround. [Formatting](https://github.com/kean/Formatting) also does.
{:.warning}

You can use [Formatting](https://github.com/kean/Formatting) as is or modify it to fit your needs. You have full control to determine which tags you want to support and in which way. You also get the performance that makes it viable to use it even in scroll views. No compromises.

<div class="References" markdown="1">

<h2 class="PostLink SectionTitle">References</h2>

- [**10 Common Mistakes in Software Localization and How to Avoid Them**](https://phrase.com/blog/posts/10-common-mistakes-in-software-localization/)
- [Hacking with Swift: **How to convert HTML to an NSAttributedString**](https://www.hackingwithswift.com/example-code/system/how-to-convert-html-to-an-nsattributedstring)

<div class="FootnotesSection" markdown="1">

[^1]: To make links clickable, use `UITextView`.
[^2]: Measured on iPhone 11 Pro using Xcode 12.2, -Os. I also had to disable system logging by setting `OS_ACTIVITY_MODE` to `disable` because WebKit was spending half of its time doing logging. You can find the tests in the [repo](https://github.com/kean/Formatting).

</div>