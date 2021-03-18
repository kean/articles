---
layout: post
title: "What's a Document?"
subtitle: A quick write-up on selecting a document format for Pulse
description: A quick write-up on selecting a document format for Pulse
date: 2021-03-15 10:00:00 -0500
category: programming
tags: programming
permalink: /post/pulse-store
uuid: 424b2065-a294-4311-9bc7-f85ed82d1290
---

<div class="BlogVideo NewScreenshot">
<video autoplay loop muted playsinline preload="auto">
  <source src="{{ site.url }}/videos/pulse-store/01.mp4" type="video/mp4">
</video>
</div>

Registering your custom document type and *extension* is straightforward: you can follow the [official sample](https://developer.apple.com/documentation/uikit/view_controllers/building_a_document_browser_app_for_custom_file_formats). The problem is, only files can have extensions, directories can not. And [Pulse](https://github.com/kean/Pulse) store is a directory: it has a manifest (`.json`), a database (`.sqlite`), and a directory with blobs.

<img width="500px" class="NewScreenshot" src="{{ site.url }}/images/posts/pulse-store/01.png">

Now, how do you turn a directory into a file? Science!

The real answer is: **zip archives**. There are, however, other options and the choice largely depends on your requirements. In this post, I want to share the thought process behind going with zip for *my use case* and share some of the implementation details.

## Apple Document Types

### Archives (.pages)

[Pages](https://www.apple.com/pages/) files are zip archives. You can change the extension to `.pages` to unarchive it (or use `unzip` from Terminal).

<img width="500px" class="NewScreenshot" src="{{ site.url }}/images/posts/pulse-store/02.png">

When you open a document, Pages unarchives it and puts it in memory. Archiving a Pages document makes a lot of sense as it saves a bit of space.

You can think of a zip archive as a key/value storage, optimized for the case of write-once/read-many and a relatively small number of distinct keys. Zip is easy to use, it's ubiquitous and fast.

One of the main _disadvantages_ of zip, or other binary formats, is its content is inaccessible making it impossible to use with source control.

### Packages (.pbxproj, .app)

Xcode project files also consist of multiple files, but they are not archives.

<img width="500px" class="NewScreenshot" src="{{ site.url }}/images/posts/pulse-store/03.png">

Xcode project files are [packages](https://en.wikipedia.org/wiki/Package_(macOS)) (a macOS concept). It is a directory displayed to the user as a single file.

Unlike archives, which are just binary blobs, packages can be added to source control, which is the main reason `.pbxproj` files are packages.

There are many other documents that Apple represents as packages: apps are packages (`.app`), photo libraries are too (`.photoslibrary`). A great advantage of packages is that they don't require any special code, `FileManager` can simply directly access the files and the directory inside the package.

## Other Approaches

Apple typically uses either packages or packages, but there are also a couple of other approaches to consider.

### SQLite

It might sound counterintuitive at first, but [SQLite claims](https://www.sqlite.org/appfileformat.html) to be an excellent document file format. From the listed advantages, I would like to point out two things:

- **Atomic Transactions**. Writes to SQLite are atomic meaning there is practically no danger of corrupting the document. This is pretty much impossible to achieve with "pile-of-files" approaches.
- **Performance**. For relatively small files, SQLite claims to be [35% faster](https://www.sqlite.org/fasterthanfs.html) than the filesystem. Is SQLite faster than ZIP archives? The document doesn't say, but I'm assuming it isn't compared with uncompressed archives.

For Pulse, SQLite seemed like an overkill, and I already commited to using Core Data[^2].

### Binary Formats

`.sqlite` and `.zip` are both binary formats. Nothing stops you from defining your own, just like some people come up with custom networking protocols. And just like with networking, it's almost always best to stick with one of the existing formats.


## Pulse Store (.pulse)

As I mentioned in the beginning, I decided to go with zip archives. The primary reasons: space savings[^3] and ease of use. Logs typically have a lot of repetitive data, the savings can be massive.

### ZIPFoundation

To implement archiving, I use [ZIPFoundation](https://github.com/weichsel/ZIPFoundation). Unlike other libraries, it doesn't have any binary dependencies and uses Apple's [Compression](https://developer.apple.com/documentation/compression) framework.

Creating an archive:

```swift
func copyStore(to storeURL: URL) throws {
    let tempDirectoryURL = // ... create temporary directory
    // ... create copy of the store ...
    // ... create copy of the blobs ...
    // ... generate manifest ....

    // Create archive
    try FileManager.default.zipItem(
        at: tempDirectoryURL,
        to: storeURL,
        shouldKeepParent: false,
        compressionMethod: .deflate, // You have an option not to
        progress: nil
    )
    try FileManager.default.removeItem(at: tempDirectoryURL)
}
```

Opening an archive:

```swift
final class LoggerStore {
    private var archive: Archive?

    init(storeURL: URL) throws {
        guard let archive = Archive(url: storeURL, accessMode: .read) else {
            throw // ...
        }
        let manifestData = try archive.dataForEntity["manifest.json"]
        let manifest = try JSONDecoder().decode(Manifest.self, manifestData)
        let unarchivedURL = FileManager.default.temporaryDirectory
            .appendingPathComponent("com.github.kean.pulse", isDirectory: true)
            .appendingPathComponent(manifest.id)
        if !FileManager.default.fileExists(atPath: unarchivedURL.path),
            let database = archive["logs.sqlite"] {
            try archive.extract(database, to: unarchivedURL + "logs.sqlite")
        }

        // ...
    }
}

private extension Archive {
    func dataForEntity(_ name: String) throws -> Data {
        guard let entity = entities[name] else {
            throw // ...
        }
        var data = Data()
        _ = try self.extract(entity) { data.append($0) }
        return data
    }
}
```

The great thing about zip archives is they are random access. Some archives, e.g. `.tar.gz`, have inter-file compression, but zip doesn't. It allows you to access individual files without unarchiving the whole thing. This is perfect!

> `Archive` doesn't implement `RandomAccessCollection` protocol, only `Sequence`. If you look at [`subscript(path: String)`](https://github.com/weichsel/ZIPFoundation/blob/cf10bbff6ac3b873e97b36b9784c79866a051a8e/Sources/ZIPFoundation/Archive.swift#L229), every time you call it, it reads and iterates over entries in the [central directory](https://en.wikipedia.org/wiki/ZIP_(file_format)#Central_directory_file_header) in a sequential fashion which is O(N). For random access, you are going to need to construct your own index in memory.
{:.warning}

As mentioned earlier, each Pulse store has a manifest (`.json`), a database (`.sqlite`), and a directory for blobs. While a database is usually relatively small, blobs can grow quite a bit because that's where all the network responses are. I have to unarchive a manifest and a database when opening a store, but with zip I can keep the blobs in an archive and access each individual blob on-demand (see `dataForEntity`)[^1].

[^1]: I’m glad I initially decided not to put blobs into Core Data, it would have prevented me from implementing some of these optimizations.

> Blob store has another optimization in place where responses automatically get deduplicated. It calculates a cryptographic hash (sha256) for each response. If two responses have the same hash, only one blob gets stored. This is great especially if the app keeps fetching the resource that doesn't change over and over again.
{:.info}

### Opening Documents

Setting the supported document type was pretty easy, I just followed the [sample code](https://developer.apple.com/documentation/uikit/view_controllers/building_a_document_browser_app_for_custom_file_formats). After adding the document type to the plist, I got the system to recognize my documents and even auto-generate an icon.

<img class="NewScreenshot" src="{{ site.url }}/images/posts/pulse-store/m1.png">

When the user double-clicks the document, the app receives an [`onOpenURL()`](https://developer.apple.com/documentation/swiftui/menu/onopenurl(perform:)) callback.

```swift
struct AppView: View {
    var body: some View {
        ConsoleView()
            .onOpenURL(perform: model.openDatabase)
    }
}
```

Another approach is to use [DocumentGroup](https://developer.apple.com/documentation/swiftui/documentgroup), but it comes with its own caveats. For a simple scenario like this `onOpenURL()` can do.

Adding a custom document type is just part of the job. I'm now also working on [`QLPreviewingController`](https://developer.apple.com/documentation/quartz/qlpreviewingcontroller) to offer previews with some store metadata, and more.

## Conclusion

A clear, concise, and easy to understand file format is a crucial part of any application design. Make sure to chose the format that works best for your requirements.

<blockquote class="quotation">
   <p>Data dominates. If you've chosen the right data structures and organized things well, the algorithms will almost always be self-evident. Data structures, not algorithms, are central to programming.</p>
   <footer>Fred Brooks</footer>
</blockquote>

<div class="References" markdown="1">

<h2 class="PostLink SectionTitle">References</h2>

- [**Sample Code**: Building a Document Browser App for Custom File Formats](https://developer.apple.com/documentation/uikit/view_controllers/building_a_document_browser_app_for_custom_file_formats)
- [**ZIPFoundation**](https://github.com/weichsel/ZIPFoundation)
- [**SQLite**: SQLite As An Application File Format](https://www.sqlite.org/appfileformat.html)

</div>

<div class="FootnotesSection" markdown="1">
[^2]: The primary reason was to make it easier to programmatically access the storage. I also have a branch with an SQLite re-write, but decided against it because I didn't find a lot of advantages compared to Core Data.
[^3]: Zip archives can also work without compression if you just want to put multiple files in a single binary.
</div>
