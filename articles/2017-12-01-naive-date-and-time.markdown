---
layout: post
title: "Naive Date and Time"
description: "Foundation offers great APIs for manipulating dates with <i>time zones</i>, however, it might be missing a few things"
date: 2017-12-01 18:00:00 +0300
category: programming
tags: ios
permalink: /post/naive-date-time
uuid: 60d37e50-dd6b-40e5-bfa9-c22e21a7a5a9
---

Foundation offers [great APIs](https://developer.apple.com/documentation/foundation/dates_and_times) for manipulating dates with time zones (e.g. working with something like `2017-09-29T15:00:00+0300`). However, sometimes it might be a bit frustrating when you just need a **naive** date or time.

{% include ad-hor.html %}

So what exactly is a naive date or a naive time? They are called "naive" because they don't have a time zone associated with them. For example:

```swift
- naive date: "2017-09-29"
- naive time: "15:00:00"
- naive datetime: "2017-09-29T15:00:00"
```

In comparison here's how you could represent specific points in time by specifying a time zone (or offset):

```swift
- datetime in UTC: "2017-09-29T15:00:00Z"
- datetime with timezone offset: "2017-09-29T15:00:00+0300"
- datetime with timezone: datetime="2017-09-29T15:00:00", timezone="Europe/London"
```

Turns out there are a lot of scenarios in which the time zone is not needed. For example, let's say you want to show user's birthday. You don't need to know either a time zone, or a specific point in time - [`Date`](https://developer.apple.com/documentation/foundation/date) - of birth. Or suppose that you want to display opening hours of a restaurant. There are many situations like these.

An obvious way to implement those scenarios using `Foundation` is with a help of [`DateComponents`](https://developer.apple.com/documentation/foundation/datecomponents). However, there are a few problems:

- All of the components (e.g. `.month`, `.hour`, etc) are optional so it's not (type) safe to pass them around.
- If you wanted to format a date to display it to the user you would have to use [`DateFormatter`](https://developer.apple.com/documentation/foundation/dateformatter), which requires conversion from `DateComponents` to `Date` which might seems a bit confusing. In general, it's fairly straightforward, but since both `DateFormatter` and `DateComponents` can carry time zone information associated with them, there is always a room for error.
- Parsing date components might be challenging.

Since I've realized that in my recent app I mostly use naive dates and times, I decided to build a type-safe abstraction to eliminate the potential errors: [NaiveDate](https://github.com/kean/NaiveDate).

## NaiveDate

The library implements three types:
- `NaiveDate` (e.g. `2017-09-29`)
- `NaiveTime` (e.g. `15:30:00`)
- `NaiveDateTime` (e.g. `2017-09-29T15:30:00` - no time zone and no offset).

Each of them implements `Equatable`, `Comparable`, `LosslessStringConvertible`, and `Codable` protocols. Naive types can also be converted to  `Date`, and `DateComponents`.

### Create

Naive dates and times can be created from a string (using a predefined format), using `Decodable`, or with a memberwise initializer:

```swift
NaiveDate("2017-10-01")
NaiveDate(year: 2017, month: 10, day: 1)

NaiveTime("15:30:00")
NaiveTime(hour: 15, minute: 30, second: 0)

NaiveDateTime("2017-10-01T15:30")
NaiveDateTime(
    date: NaiveDate(year: 2017, month: 10, day: 1),
    time: NaiveTime(hour: 15, minute: 30, second: 0)
)
```

### Format

Format dates without having to worry about time zones:

```swift
let date = NaiveDate("2017-11-01")!
NaiveDateFormatter(dateStyle: .short).string(from: date)
// prints "Nov 1, 2017"

let time = NaiveTime("15:00")!
NaiveDateFormatter(timeStyle: .short).string(from: time)
// prints "3:00 PM"

let dateTime = NaiveDateTime("2017-11-01T15:30:00")!
NaiveDateFormatter(dateStyle: .short, timeStyle: .short).string(from: dateTime)
// prints "Nov 1, 2017 at 3:30 PM"
```

### Convert

When you do need time zones, convert `NaiveDate` to `Date`:

```swift
let date = NaiveDate(year: 2017, month: 10, day: 1)

// Creates `Date` in a calendar's time zone
// "2017-10-01T00:00:00+0300" if user is in MSK
Calendar.current.date(from: date)
```

```swift
let dateTime = NaiveDateTime(
    date: NaiveDate(year: 2017, month: 10, day: 1),
    time: NaiveTime(hour: 15, minute: 30, second: 0)
)

// Creates `Date` in a calendar's time zone
// "2017-10-01T15:30:00+0300" if user is in MSK
Calendar.current.date(from: dateTime)
```

## Notes

Naive dates and times are extremely easy to use. There are no time zones to worry about. You can easily parse them without having to use custom date formatters. It works just as you expect. But, it's important to understand the limitations behind them.

They are called "naive" because they don't have a time zone associated with them. This means the date may not actually exist in some areas in the world, even though they are "valid". For example, when daylight saving changes are applied the clock typically moves forward or backward by one hour. This means certain dates never occur or may occur more than once. The moment you need to do any precise manipulations with time, always use native `Date` and `Calendar` types.

## Resources

- [Apple Developer Documentation: Dates and Times](https://developer.apple.com/documentation/foundation/dates_and_times)
- [NaiveDateTime](https://docs.rs/chrono/0.3.1/chrono/naive/datetime/struct.NaiveDateTime.html)
