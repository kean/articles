---
layout: post
title: "Core Data Progressive Migrations"
description: "Using NSPersistentStoreCoordinato and NSMigrationManager for progressive schema migrations in Core Data"
date: 2016-06-18 10:00:00 +0300
category: programming
tags: ios
permalink: /post/core-data-progressive-migrations
redirect_from: /blog/core-data-progressive-migrations
uuid: cdd8c605-9632-4de9-a370-2049aa69f8b4
---

[Schema migration](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreDataVersioning/Articles/Introduction.html#//apple_ref/doc/uid/TP40004399-CH1-SW1) is a daunting process but you must get it right. Otherwise you might end up breaking your app and corrupting user data in the process. This articles aims to make custom Core Data migrations more approachable.

When dealing with something as critical you want to keep things simple. With Core Data it means either avoiding migrations altogether, or using [lightweight migrations](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreDataVersioning/Articles/vmLightweightMigration.html). However, sometimes custom migrations are unavoidable.

**Prerequisite:** [Core Data Model Versioning and Data Migration Programming Guide](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreDataVersioning/Articles/Introduction.html#//apple_ref/doc/uid/TP40004399-CH1-SW1)

{% include ad-hor.html %}

> ## TL;DR
- Core Data doesn't support *progressive migrations* out of the box (via some boolean option)
- Existing libraries that do are often hard and unsafe to use
- It's simple to trick Core Data into performing one using only [`NSPersistentStoreCoordinator`](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSPersistentStoreCoordinator_Class/)
- "Proper" way requires [`NSMigrationManager`](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/NSMigrationManager_class/#//apple_ref/occ/instm/NSMigrationManager/migrateStoreFromURL:type:options:withMappingModel:toDestinationURL:destinationType:destinationOptions:error:)
- Testing migrations is extremely important

## Problem

*This part describes how you might encounter custom migrations for the first time, and what are progressive migrations.*

Imagine you are working on an app. You started with *mom_v1 ([`NSManagedObjectModel`](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/CoreDataFramework/Classes/NSManagedObjectModel_Class/) *version 1*). Later you added *mom_v2* with minimal schema changes. At this point your Core Data setup might look like [this](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreDataVersioning/Articles/vmInitiating.html#//apple_ref/doc/uid/TP40004399-CH7-SW3):

```swift
let psc = NSPersistentStoreCoordinator(managedObjectModel: <#mom#>)
try psc.addPersistentStore(
    ofType: NSSQLiteStoreType,
    configurationName: nil,
    at: <#store_url#>,
    options: [NSMigratePersistentStoresAutomaticallyOption : true,
              NSInferMappingModelAutomaticallyOption : true]
)
```

When you add a store that uses *mom_v1* to the `NSPersistentStoreCoordinator`, it automatically creates an [inferred mapping model](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreDataVersioning/Articles/vmLightweightMigration.html#//apple_ref/doc/uid/TP40004399-CH4-SW2) between *mom_v1* and *mom_v2* and performs a lightweight migration.

Imagine you now need to add *mom_v3* with changes that can’t be carried out by lightweight migration - you need a [custom mapping model](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreDataVersioning/Articles/vmMappingOverview.html) from *mom_v2*. You jump through all the hoops required to create one. But then comes the problem. How does `NSPersistentStoreCoordinator` works with custom and inferred mappings?

Here are some pieces of documentation that we get:

> `NSMigratePersistentStoresAutomaticallyOption`
>
> If the version hash information for the added store is determined to be incompatible with the model for the coordinator, Core Data will attempt to locate the source and mapping models in the application bundles, and perform a migration.

> `NSInferMappingModelAutomaticallyOption`
>
> When combined with `NSMigratePersistentStoresAutomaticallyOption`, coordinator will attempt to infer a mapping model if none can be found.

After some experimentation you find out that:

- When you add a store with *mom_v2* to coordinator it finds your custom mapping *mom_v2 ~> mom_v3* and uses it for a migration (as expected)
- When you add a store with *mom_v1* to coordinator it looks for a custom mapping *mom_v1 ~> mom_v3* which it can't find. It then infers a mapping model and performs a lightweight migration.

The second case worked not how you might have expected, or at least not how you wanted it to. What we want is to migrate from *mom_v1* to *mom_v2* using an inferred mapping, and then migrate from *mom_v2* to *mom_v3* using a custom mapping - perform a *progressive migration*.

Unfortunately, Core Data doesn’t support progressive migrations like this out of the box - via some boolean option. Instead it optimizes for performance by skipping intermediate steps. In fact, [there is no notion](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreDataVersioning/Articles/vmUnderstandingVersions.html#//apple_ref/doc/uid/TP40004399-CH2-SW1) of linear version history built into Core Data at all. While it makes sense for lightweight migrations, it doesn't really work for custom ones.

It might be tempting to create yet another custom mapping *mom_v1 ~> mom_v3* to workaround this problem, but that would be an embarrassment. It obviously doesn’t scale well as we add new model versions. Fortunately, there are at least two different ways to implement progressive migrations using Core Data APIs.


## Progressive Migrations (Automatic)

There is a very simple way to trick Core Data into performing progressive migrations using only `NSPersistentStoreCoordinator`. Each coordinator is initialized with a destination mom. When you add a store to the coordinator it performs a migration to the mom that it was initialized with. A destination mom doesn't have to be your final mom version. We can use these facts to implement progressive migrations:

```
func migrate(store)
    V <- current mom version
    if V is not final
        psc <- create coordinator with mom V+1
        add store to psc which migrates it to V+1
        migrate(store)
```

Here's a slightly modified non-recursive implementation of `migrate` function:

```swift
// moms: [mom_v1, mom_v2, ... , mom_vN]
func migrateStore(at storeURL: URL, moms: [NSManagedObjectModel]) throws {
    let idx = try indexOfCompatibleMom(at: storeURL, moms: moms)
    for mom in moms.suffix(from: (idx + 1)) {
        try autoreleasepool {
            let psc = NSPersistentStoreCoordinator(managedObjectModel: mom)
            try psc.addPersistentStore(
                ofType: NSSQLiteStoreType,
                configurationName: nil,
                at: storeURL,
                options: [NSMigratePersistentStoresAutomaticallyOption : true,
                          NSInferMappingModelAutomaticallyOption : true]
            )
        }
    }
}
```

This function takes a linear version history as a parameter. To create it you should read all moms from the bundle and order them. Compiled moms have `.mom` extension and are in general stored in a subdirectory with `.momd` extension. To create a list of moms you can either provide an ordered list of hard-coded file names, or implement some simple heuristic that reads and sorts them automatically. I prefer the former, because mom names come in handy when writing tests.

And here's an implementation of `indexOfCompatibleMom(at:moms:)` function:

```swift
func indexOfCompatibleMom(at storeURL: URL, moms: [NSManagedObjectModel]) throws -> Int {
    let meta = try NSPersistentStoreCoordinator.metadataForPersistentStore(ofType: NSSQLiteStoreType, at: storeURL)
    guard let idx = moms.index(where: { $0.isConfiguration(withName: nil, compatibleWithStoreMetadata: meta) }) else {
        throw MigrationError.IncompatibleModels
    }
    return idx
}

// Core Data forces us to use Swift exceptions so we just go with it
enum MigrationError: ErrorProtocol {
    case IncompatibleModels
}
```

This simple solution does just what we wanted, but it has some downsides:

- You won’t be able to perform [multi-pass migrations](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreDataVersioning/Articles/vmCustomizing.html#//apple_ref/doc/uid/TP40004399-CH8-SW9) this way
- Core Data looks for mapping models in bundles returned by `NSBundle`'s `allBundles` and `allFrameworks` methods. It can't search for mappings in other locations, which can hurt testing.

Eventually, you might want to customize a migration process.


## Progressive Migrations (Manual)

The algorithm is similar to the one introduced in the previous part, but this time we use `NSMigrationManager` class to [perform  migrations](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreDataVersioning/Articles/vmCustomizing.html#//apple_ref/doc/uid/TP40004399-CH8-SW1). Here's an outline of what we need to do:

```swift
func migrate(store)
    V <- current mom version
    if V is not final
        migrate(store, V, V+1)
        migrate(store)

func migrate(store, v1, v2)
    destURL <- destination URL in temp folder
    mapping <- findMapping(v1, v2)
    migrate store to destURL using mapping
    replace store from store at destURL

func findMapping(v1, v2)
    mapping <- custom mapping from v1 to v2
    if mapping = nil
        mapping <- inferred mapping from v1 to v2
    return mapping
```

You can find a full non-recursive implementation [here](https://gist.github.com/kean/28439b29532993b620497621a4545789). The provided functions match the ones defined in the pseudocode. Let's dive into the code to see some of the details.

Here's the first function that loops through the object models. It uses an `indexOfCompatibleMom(at:moms:)` function that we added earlier.

```swift
// moms: [mom_v1, mom_v2, ... , mom_vN]
func migrateStore(at storeURL: URL, moms: [NSManagedObjectModel]) throws {
    let idx = try indexOfCompatibleMom(at: storeURL, moms: moms)
    let remaining = moms.suffix(from: (idx + 1))
    guard remaining.count > 0 else {
        return // migration not necessary
    }
    _ = try remaining.reduce(moms[idx]) { smom, dmom in
        try migrateStore(at: storeURL, from: smom, to: dmom)
        return dmom
    }
}
```

Next is a function that performs an actual migration. As you can see it relies heavily on Swift [error handling model](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/ErrorHandling.html).

```swift
func migrateStore(at storeURL: URL, from smom: NSManagedObjectModel, to dmom: NSManagedObjectModel) throws {
    // Prepare temp directory
    let dir = try URL(fileURLWithPath: NSTemporaryDirectory()).appendingPathComponent(UUID().uuidString)
    try FileManager.default().createDirectory(at: dir, withIntermediateDirectories: true, attributes: nil)
    defer {
        _ = try? FileManager.default().removeItem(at: dir)
    }

    // Perform migration
    let mapping = try findMapping(from: smom, to: dmom)
    let destURL = try dir.appendingPathComponent(storeURL.lastPathComponent!)
    let manager = NSMigrationManager(sourceModel: smom, destinationModel: dmom)
    try autoreleasepool {
        try manager.migrateStore(
            from: storeURL,
            sourceType: NSSQLiteStoreType,
            options: nil,
            with: mapping,
            toDestinationURL: destURL,
            destinationType: NSSQLiteStoreType,
            destinationOptions: nil
        )
    }

    // Replace source store
    let psc = NSPersistentStoreCoordinator(managedObjectModel: dmom)
    try psc.replacePersistentStore(
        at: storeURL,
        destinationOptions: nil,
        withPersistentStoreFrom: destURL,
        sourceOptions: nil,
        ofType: NSSQLiteStoreType
    )
}
```

The last step where we replace the source store is very important. The `replacePersistentStore(at:...)` method (which was added in iOS 9.0) honors file locks, journal files, journaling modes, and other intricacies related to SQLite. It guarantees that you won't [corrupt user data](https://www.sqlite.org/howtocorrupt.html) by failing to replace stores in a single transaction.

The last piece is a simple function that finds a mapping model:

```swift
func findMapping(from smom: NSManagedObjectModel, to dmom: NSManagedObjectModel) throws -> NSMappingModel {
    if let mapping = NSMappingModel(from: Bundle.allBundles(), forSourceModel: smom, destinationModel: dmom) {
        return mapping // found custom mapping
    }
    return try NSMappingModel.inferredMappingModel(forSourceModel: smom, destinationModel: dmom)
}
```


## Testing

Testing migrations is extremely important. It makes sense to test that migrations are performed progressively first. And then test each migration step (**mom_v2 ~> mom_v3**, etc) in isolation.

It is very important to test migrations on different data sets and different versions of your store. There are at least two ways to populate stores with data:

- Run a specific version of an app. This way you generate data just like your app actually does, including some potential errors that you might be not aware of.
- Populate stores programatically. This would allow you to simulate some very specific cases that are either hard or impossible to reproduce on the device.

Here's how your test methods might actually look like (this is a very rough draft):

```swift
func testMigrationFrom_v2_to_v3() {
    let mom_v2 = <#create_mom_v2#>
    let mom_v3 = <#create_mom_v3#>

    // Create a copy of v2 store and open it using mom_v2
    let moc_v2 = try! openStore(at: prepareStore(name: "v2"), mom: mom_v2)

    // Create another copy of v2 store, migrate it to mom_v3 and open it
    let storeURL = prepareStore(name: "v2")
    do {
        try migrateStore(at: storeURL, moms: [mom_v2, mom_v3])
    } catch {
        XCTFail()
    }

    let moc_v3 = try! openStore(at: storeURL, mom: mom_v3)

    // Now we have two versions of our store at the same time

    // Assertions for object 1
    var request = NSFetchRequest<NSManagedObject>(entityName: "SomeEntity")
    request.predicate = Predicate(format: "UID == %@", argumentArray: ["572b5da8e25ce500063a3594"])
    let obj_v2 = try! moc_v2.execute(request)
    let obj_v3 = try! moc_v3.execute(request)

    XCTAssertEqual(obj_v2.value(forKey: "title"), "SomeTitle", "Error")
    XCTAssertEqual(obj_v2.value(forKey: "title"), obj_v3.value(forKey: "title"), "Error")
}

func openStore(at storeURL: URL, mom: NSManagedObjectModel) throws -> NSManagedObjectContext {
    // Open store with a given URL and return moc
}

func prepareStore(name: String) -> URL {
    // Prepare store by copying all it's files from a test bundle to a temp directory
    // either using NSPersistentStoreCoordinator or NSFileManager
}
```


## Notes

- You can skip several moms during migration to improve performance. While it makes sense for lightweight migrations, it doesn't really work for custom migrations. In general it is ill-advised, because it complicates the migration process, and it doesn't really come into play unless users skip multiple versions of your app.
- If the migration fails you might want to delete (or copy to a safe location) the existing store and create a new one using the latest mom.
- Multi-pass migrations can be used to migrate large data sets without using too much memory.
- Mapping model becomes invalid after changes to either its source or destination moms. You have to recreate a mapping model after you make such changes.
- If you set the `com.apple.CoreData.MigrationDebug` preference to `1`, Core Data will log information about exceptional cases as it migrates data.
- The `com.apple.CoreData.SQLDebug` preference lets you see the actual SQL sent to SQLite.

{% include references-start.html %}

1. [Core Data Model Versioning and Data Migration Programming Guide](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreDataVersioning/Articles/Introduction.html#//apple_ref/doc/uid/TP40004399-CH1-SW1)
2. [How to Corrupt an SQLite Database File](https://www.sqlite.org/howtocorrupt.html)
3. [Mac OS X Debugging Magic](https://developer.apple.com/library/mac/technotes/tn2124/_index.html#//apple_ref/doc/uid/DTS10003391-CH1-SECCOREDATA)
4. [Launch Arguments & Environment Variables](http://nshipster.com/launch-arguments-and-environment-variables/)

{% include references-end.html %}