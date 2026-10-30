---
layout: post
title: Pocket Casts on iPhone Duo
subtitle: New features, 1,700 screenshots, and a lot of help from agents
description: "How we updated Pocket Casts for iPhone Duo with a bigger mini player and a new Now Playing, and tested hundreds of screens to catch layout issues."
date: 2026-10-25 09:00:00 -0400
category: programming
tags: [whats-new, testing, ios]
permalink: /post/pocket-casts-iphone-duo
uuid: 3f6b0c1e-8a52-4d7e-9b14-c2e5a7d90f38
image:
  path: /images/posts/iphone-duo/cover.png
  thumbnail: /images/posts/iphone-duo/cover-thumb.webp
  card: /images/posts/iphone-duo/cover.webp
  width: 2560
  height: 1280
---

When Apple announced iPhone Duo, we had a couple of weeks to get Pocket Casts ready for the launch, no device to test on, and no native apps to look at for examples – no pressure. I'd switched to the team a few months earlier, and after adopting [Liquid Glass](https://blog.pocketcasts.com/2026/06/11/liquid-glass/) and helping with the new [Apple TV app](https://blog.pocketcasts.com/2026/07/27/pocket-casts-on-the-big-screen/), I was excited to take it on.

The biggest surprise was the horizontal safe area. On Duo, the status bar is a pill on the right edge of the screen, and when you unfold, the tab bar and toolbars stack vertically under it, so apps have to keep clear of the right edge. There were issues with it everywhere, and Pocket Casts has hundreds of screens, sheets, and alerts. How do you find all of them, fix them, and check everything again after every round of fixes, in time for the launch? Here's what we shipped, and then how I got there with the help of agents.

## What's New

We updated the app for iPhone Duo, and, of course, the first step was making the tab bar, toolbars, and native navigation work correctly, folded and unfolded.

<figure class="PostFigure PostFigure--wide">
<img class="NewScreenshot" alt="Podcasts on iPhone Duo" src="/images/posts/iphone-duo/podcasts.webp" width="1540" height="1182">
<figcaption>The Podcasts tab, with the status pill and the tab bar on the right edge</figcaption>
</figure>

The main changes are in the mini player and Now Playing. Once you unfold, the mini player has room for skip buttons and a progress bar, so you can control playback without opening the full player.

<figure class="PostFigure">
<img class="NewScreenshot" alt="Discover with the mini player on the unfolded iPhone Duo" src="/images/posts/iphone-duo/discover.webp" width="1540" height="1113" loading="lazy" decoding="async">
<figcaption>Discover with the expanded mini player</figcaption>
</figure>

Now Playing makes use of the extra space too. Instead of stacking everything in a single column, it shows its tabs, like Transcript and Chapters, side by side with the controls, so you can follow along with the transcript while you listen.

<figure class="PostFigure">
<img class="NewScreenshot" alt="Now Playing with the transcript on the unfolded iPhone Duo" src="/images/posts/iphone-duo/now-playing.webp" width="1540" height="1113" loading="lazy" decoding="async">
<figcaption>Now Playing with the transcript side by side</figcaption>
</figure>

We also made Now Playing a bit nicer visually. The background now takes its colors from the podcast artwork, so the player adapts to each of your favorite shows, from the purple of The Vergecast to the warm orange of The Journal.

<figure class="PostFigure">
<img class="NewScreenshot" alt="Now Playing themed with the colors of six different podcasts" src="/images/posts/iphone-duo/now-playing-themes.webp" width="1540" height="1570" loading="lazy" decoding="async">
<figcaption>Six shows, six moods</figcaption>
</figure>

> Pocket Casts is free, and you can [download it on the App Store](https://apps.apple.com/app/id414834813).
{:.info}

## Harder Than It Looks

The new features are the fun part. The rest of the app looks like it should just work. It never just works, and getting there took an enormous amount of testing and fixing, especially for the horizontal safe area.

Duo also exposed a bigger gap: parts of the app needed more love on a big screen, like split views and popovers. That will take more than one release, so this one was about fixing what was broken and shipping the new mini player and Now Playing.

Last year, in [Claude Code Experience]({{ site.url }}/post/experiencing-claude-code), I wrote that I was still working with Claude interactively, one change at a time, but could see myself "parallelizing the work using git worktrees." Duo is where that happened: agents driving several simulators at once, every fix in its own worktree, and other agents checking their work. Here's how it went, with numbers.

## Apple's Skill Goes First

Apple's [Prepare for iPhone Duo](https://developer.apple.com/iphone-duo/prepare/) guide starts with the Xcode `app resizability` skill. You ask the coding assistant to "get my app ready for iPhone Duo", and it looks for what breaks when a window changes size: `UIScreen.main`, orientation checks, and idiom checks[^1]. I had it split the work into small PRs, one per kind of change, so each could be reviewed and reverted on its own:

- **Screen to view.** Hairlines use the view's display scale, layouts size from their own width, small-screen layouts compare the available height, and Display Zoom comes from the window's screen.
- **Idiom to size class.** Form sheets are picked by size class instead of the iPad idiom (25 files), and the remaining idiom checks read the trait collection.
- **Keyboard and safe area.** Six screens use `keyboardLayoutGuide` instead of hand-rolled keyboard notifications (−202 lines); footer, sheet, and video controls stay inside the safe area; and insets are read at layout time.

> Pocket Casts is open source, so you can follow along with all the changes on [GitHub](https://github.com/Automattic/pocket-casts-ios), or propose your own.
{:.info}

The skill can't see layout, though. It has no way to know that a header ends up under the status pill, or that a row runs under the side bar. That's what the screenshots are for.

## Smoke Test, Then Full Blast

How do you let agents "see" your app? Fortunately, Xcode 27 arrived at exactly the right time. Its [MCP server](https://developer.apple.com/documentation/xcode/giving-external-agents-access-to-xcode) gained new [DeviceInteraction tools](https://redalemeden.com/derived-data/2026/what-new-in-xcode-27-mcp-bridge/) that let agents take screenshots, read the view hierarchy, and tap through the app on a simulator.

The first run was deliberately quick. An agent walked the high-traffic flows on the Duo simulator (Podcasts, podcast page, Playlists, Discover, player), screenshotting every step, and filed the first batch of issues. Agents fixed those too, one worktree and PR per issue across parallel simulators, each PR with before/after screenshots so I could verify fixes without building the branch.

Then came the full sweep of the app across iPhone, Duo, iPad, Slide Over, and other window sizes: about 1,700 screenshots in total. Watching Claude spawn dozens of agents and simulators, eating up so much RAM that 64 GB starts to feel small, never gets old.

<figure class="PostFigure">
<img class="NewScreenshot" alt="A gallery of 1,201 iPhone Duo screenshots grouped by area" src="/images/posts/iphone-duo/screenshot-sweep.webp" width="1540" height="1320" loading="lazy" decoding="async">
<figcaption>The screenshot sweep for iPhone Duo, grouped by area</figcaption>
</figure>

> One gap: agents can't drive the unfolded Duo yet, which looks like an Xcode issue. The sweep covers the folded pose, and I used iPad mini in landscape to check the regular × regular layouts.
{:.warning}

## Agents All the Way Down

The sweep logged nearly a thousand observations. Most were noise: already tracked, working as intended, or fixed by the time I looked. To filter them, a second set of agents independently reviewed all the screenshots without seeing the sweep's notes. Candidates were then de-duplicated against existing issues, and I checked every screenshot that became an issue myself.

What was left went straight into triage. Each issue has screenshots, repro steps, the affected screens, and a likely cause in code where the agent could find one. Here's the only High-priority one:

> **Fix "Edit" in the multi-select actions sheet hidden under the status pill on iPhone Duo.** The sheet is presented full width, so when it's tall (6 or more actions), the header row lands exactly where the status pill is drawn. Edit is covered, and a tap at its position does nothing, so the user can't open Rearrange Actions. *Likely cause:* `MultiSelectActionController.xib` pins the header buttons 16 pt from the trailing edge instead of the safe area. The same fix was made for `ShelfActionsViewController.xib` in an earlier issue; this is the sibling that was missed.

The same fix loop ran on the new batch. The fix for this one is [#5394](https://github.com/Automattic/pocket-casts-ios/pull/5394), an XIB constraint change[^2]. The PR has the root cause, the test steps, and before/after screenshots, including ones from a regular iPhone showing that nothing changed there, a check I'd otherwise skip.

<figure class="PostFigure">
<img class="NewScreenshot" alt="Before and after screenshots in a pull request" src="/images/posts/iphone-duo/pr-screenshots.webp" width="1540" height="1191" loading="lazy" decoding="async">
<figcaption>The Edit button, finally tappable</figcaption>
</figure>

Overall, counting the ones I found myself, the agents and I logged **56 issues** during this project, and the vast majority of them were on point.

## Regression Testing

Then came regression testing, which I'd been slowly handing over to agents. The suite had 77 scenarios, from account and deep links to sync and Up Next[^3]. Each one lists what must be true in plain language, not the steps to check it, so it reads more like acceptance criteria than a test:

```markdown
## Reorder, remove and clear
setup: `--fixture up-next:5`

- Dragging a row by its handle moves it, and the order survives relaunch
- Swiping to remove takes the episode out without playing or archiving it
- Clear asks for confirmation, then shows the empty state
- Settings > Up Next History restores the cleared queue
```

The agent works out the steps itself, and the `setup` line fills Up Next with five episodes first, so it doesn't spend time getting there. Agents also flag anything else that looks off along the way, so the suite covers far more ground than XCUITests. A second agent then reads the code behind each finding, verifies or rejects it, and notes the likely cause before anything gets filed.

<figure class="PostFigure">
<img class="NewScreenshot" alt="A regression run with agents driving the app on several simulators" src="/images/posts/iphone-duo/qa-run.webp" width="1712" height="710" loading="lazy" decoding="async">
<figcaption>A regression run: agents drive the app on several simulators in parallel, then verify the findings</figcaption>
</figure>

It's still experimental, but I find agents drive tests end to end with a higher success rate. They make decisions on the spot and reason across test cases. It's not free, though: runs are slower, cost tokens, and aren't deterministic, which is both a pro and a con.

## Before Every Release

I now run the suite before every release. It packs about 24 hours of testing into a few hours[^4] and gives me tremendous confidence that every flow, even the obscure ones, works.

Across all runs, it has found **more than a hundred issues**. Most are minor, but papercuts hurt, and I've already fixed a bunch of them.

Since Duo, the suite has gotten faster and gained many more scenarios and a dedicated reporter.

<figure class="PostFigure">
<img class="NewScreenshot" alt="A QA report for a full regression run" src="/images/posts/iphone-duo/qa-report.webp" width="1876" height="998" loading="lazy" decoding="async">
<figcaption>The new QA reporter</figcaption>
</figure>

I'm also adding video walkthroughs to bug reports and regression runs. Each one shows the app side by side with the logs and an explanation in a single video.

<figure class="PostFigure">
<div class="BlogVideo NewScreenshot">
<video controls muted playsinline preload="auto" width="1162" height="960">
	<source src="{{ site.url }}/videos/iphone-duo/qa-walkthrough.mp4" type="video/mp4">
</video>
</div>
<figcaption>A walkthrough of a bug from a regression run</figcaption>
</figure>

## If You're Adopting Duo

Start with Apple's skill, but don't expect it to do the whole job. Most of what broke for us was layout, and most of that came down to the horizontal safe area. Anything pinned to the screen edges instead of the safe area ends up under the status pill or the toolbar on the side, and the only reliable way I found to catch it was to look at every screen in every size.

And if your app isn't great on large screens yet, Duo is where it catches up with you. Making the app feel at home on a big screen is the real work, and I feel like for many apps, Duo will be a great excuse to finally put more resources into it. iPad gets all of it for free.

## What's Next

With the broken screens out of the way, there's a lot more coming to Pocket Casts soon, so stay tuned. If you want to be on the latest beta, you can [ask to join it](https://support.pocketcasts.com/knowledge-base/beta-testing/). Pocket Casts is free, and you can [download it on the App Store](https://apps.apple.com/app/id414834813).

My first impression was that it's primarily a great iPad that folds into your pocket, with some interesting ideas about navigation. I was less convinced by the iPhone experience on the outer screen, given its size and weight, but I'm curious to see where Apple takes it.

As a developer, though, it was exciting to work on Duo, come up with new solutions together with the design team, and try new techniques to build for it. The lesson I'll take to the next project is simpler: look at every screen. That used to be too expensive to even try, so you'd test the main flows and hope for the rest. Now agents do it, and they can do it again after every round of fixes.

[^1]: If you use a different agent, `xcrun agent skills export` gives you the skill as files.

[^2]: Yes, the app has a bunch of those, and I don't mind them. I wish choosing between UIKit and SwiftUI was simpler, but agents make it even more complicated.

[^3]: Agents use a set of test accounts, from Free to one with 150+ podcasts, so scenarios that change data don't step on each other.

[^4]: That's the total time the individual agents spend testing the app. With sharding, I can scale it up, and the token cost grows linearly. A full run now takes around 4 hours.
