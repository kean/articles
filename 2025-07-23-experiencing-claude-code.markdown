---
layout: post
title: Claude Code Experience
subtitle: A pragmatic take on AI coding with practical insights
description: A pragmatic take on AI coding with practical insights
date: 2025-07-23 09:00:00 -0500
category: programming
tags: programming, ai, ios
permalink: /post/experiencing-claude-code
mastodon: TBD
uuid: a8d013bd-532e-4a9c-a252-422112781c68
---

There's a lot of anxiety around AI coding, but what people often fear is uncertainty. The best way to address it is through action. These tools aren't going away, so I decided to give one of the latest ones a fair shot this week.

I've been using Claude chats for a while, but tools like Cursor never clicked for me. I just didn't want to introduce a separate IDE into my workflow, and I didn't feel they offered enough value for an experienced engineer. That changed with [Claude Code](https://www.anthropic.com/claude-code).

## Getting Started

Claude Code is an agentic coding assistant that runs in a terminal and can research your codebase without manual context selection and make coordinated changes across files automatically. It can use command line tools (like git) and MCP servers (like GitHub) to extend its capabilities.

I tried Claude Code when it launched a few months ago and was immediately drawn to its terminal-based interface. I've never been a fan of similar products that required you to learn a separate IDE. Claude Code works alongside Xcode and feels like a natural extension of my current workflow.

<img class="Screenshot" alt="Claude Code" src="/images/posts/claude/claude-code-screen-01.png">

Unfortunately, the initial version only had a pay-per-token model where costs ramp up quickly. I tried it a few times, but that was it. But now it's included in the Claude subscription. I upgraded to a Max monthly plan and started exploring without worrying about costs.

I began by watching the video about [mastering Claude Code](https://www.youtube.com/watch?v=6eBSHbLKuN0) from Boris Cherny, its creator. It was extremely helpful and showed me that I had a lot to learn. I've read the bare minimum of the official documentation and started exploring what it can do.

## Asking Questions

Following Boris's recommendation, I started by asking questions about the codebase. Claude can explore your project and answer questions, but don't take its responses at face value. I currently work on the [WordPress iOS apps](https://github.com/wordpress-mobile/WordPress-iOS), which are open-source, and when I asked about post syncing conflicts, it mentioned "collaborative editing" which our app doesn't support. See the full interaction [here](https://gist.github.com/kean/e88f6de0bd4f234b55fa76cfb345b137).

That's OK though. I didn't install it for Q&A.

## Writing Code

### Starting Small: Refactoring

I had this unused method sitting in one of my files, so I asked: "Remove unused code from this file @file_path". It worked correctly, though not particularly fast. I opened [PR #24590](https://github.com/wordpress-mobile/WordPress-iOS/pull/24590). Definitely not the most efficient use of AI, but hey, it made the right changes.

My next attempt was more ambitious. I asked Claude to rewrite the entire Objective-C "Support Logs" screen using SwiftUI. Here's the [PR #24591](https://github.com/wordpress-mobile/WordPress-iOS/pull/24591). It's a simple screen with low stakes, and my brief prompt worked. While it didn't match my coding style perfectly, I manually corrected it.

### Feature: Activity Logs and Backups

Now confident, I tackled a large planned project: implementing new "Activity Logs" and "Backups" screens in SwiftUI using the refreshed WordPress design system. This is where things got interesting.

I created a small standalone app target ([PR #24598](https://github.com/wordpress-mobile/WordPress-iOS/pull/24598)) to iterate quickly while still being able to access all the app dependencies. I asked Claude to generate code based on the previous implementation but with a new design based on the screenshot of a different screen I dropped in the terminal.

It generated code, but it ignored our coding standards. That's when it hit me. While Claude *can* explore your project automatically by reading files and running bash commands, it starts from zero every time unless you pre-configure it using `CLAUDE.md`. Think of it as Claude's onboarding document. The better you write it, the better code Claude writes for you.

<img class="NewScreenshot" src="/images/posts/claude/give-claude-context.png">

> You can generate a `CLAUDE.md` file using `/init`. The file should include coding standards, project structure, common patterns, and any domain-specific knowledge. The goal is to update it regularly as you discover what Claude needs to know. This file is part of CLAUDE.md [memory management](https://docs.anthropic.com/en/docs/claude-code/memory) with a few more tools at your disposal.
{:.info}

With an improved `CLAUDE.md`, I regenerated the code with much better results. If Claude isn't generating what you want, your prompt is likely underspecified.

To review changes, I asked Claude to generate previews. Claude generated tons of mocks and code with perfect accuracy, nearly instantly, using realistic domain data. It even knew to use actual WordPress terminology. This is where LLMs truly shine.

<img class="Screenshot" alt="Xcode Preview" src="/images/posts/claude/xcode-preview.png">

I continued iterating, opening multiple PRs resulting in this feature branch: [PR #24597](https://github.com/wordpress-mobile/WordPress-iOS/pull/24597). It has `+3,672 −4,967` changes with around 90% generated by prompts. I even used Claude Code to create commits by simply asking "commit".

### Prototyping

The next day, a colleague had a couple of questions about how `WKWebView` interacts with the keyboard and whether the app has any control over it. I asked Claude to generate a sample project with an HTML form and add keyboard observers in Swift code. It did so instantly. This is another area where it excels. Normally, I wouldn't bother – too time-consuming. Now I can verify anything using prototypes instantly, a massive productivity unlock.

## My Workflow

I started with short one-liner prompts, with predictably poor results. After a week of trial and error, my workflow evolved to:

<div class="blog-new-li" markdown="1">
1. Start a new Claude session (`/clear`)
2. Ask it to read files I'll work on and any related files. Pass related screenshots or documents[^5].
3. (Optional) Switch to the "Plan" mode (⇧Tab)
4. Request the changes (e.g., "rewrite this" or "generate that") and maybe throw an "ultrathink" at it[^2]
5. Iterate on the plan
6. Switch to "Auto-Accept Edits" mode (⇧Tab) and execute
7. Review changes and iterate until you are satisfied with the results
</div>

I typically make one change at a time for easier review. Claude is more accurate with focused tasks. Asking it to do too much usually fails as it makes too many assumptions that can be hard to correct later. Even with detailed prompts, expect multiple iterations. It rarely one-shots changes unless the changes themselves aren't very specific like generating mock data.

<img class="NewScreenshot" src="/images/posts/claude/common-workflows.png">

I often skip the planning, especially for small changes. As you get more experienced with Claude, you are more likely to generate what you want on the first plan. There doesn't seem to be that much science around prompts. For me, as long as you include some relevant details, it works fine. Don't overthink it.

> One limitation: Claude 4 models have a 200K token context window. It's decent for daily use but can't process entire projects, so keep your context focused. If it overflows, Claude automatically compacts it, which works but takes time.
{:.info}

## Where Claude Code Excels

LLMs excel at generating text (including code):

- Generating unit tests
- Creating mock data and previews
- Anonymizing production data
- Refactoring existing code
- Prototyping with sample projects
- Generating documentation
- Summarizing changes for commits, PRs, and release notes

## Where It Struggles

- Making a large number of small changes across the entire project
- Working with newest APIs that the model doesn't know about yet
- Tasks requiring genuine reasoning

As Apple [has shown](https://arxiv.org/abs/2410.05229), AI can't actually reason – it's all statistics under the hood. This shows in practice as it occasionally makes nonsensical changes or continuously walks down paths that are obvious dead-ends.

For an example of small changes, if you ask to rename a class, it spawns an agent to search for usage and read individual files, taking minutes for what should be instant. In one case, it took 25 minutes to rename `EmptyStateView` across 137 occurrences in 41 files, while also making unnecessary changes like renaming unrelated methods. Here's [the full session](https://gist.github.com/kean/89a3031c17f5d9925f6801c786bc46a6). The workaround? Ask Claude to generate and run a script instead, which is surprisingly effective[^3].

The amount of time certain operations take is a limiting factor, but it depends on what you ask it to do. It's the art of writing optimal prompts that you know it can execute well. The other option is parallelization. I currently work interactively with Claude Code, making changes one at a time, as it makes it easier to review the code and it just requires too much steering. But I can see myself parallelizing the work using git worktrees, a [recommended approach](https://www.anthropic.com/engineering/claude-code-best-practices#c-use-git-worktrees), and spawning additional Claude sessions to work on other tickets in parallel, particularly bug fixes or other small tasks that can be tested with unit tests and don't require a long review or a lot of steering.

One thing worth mentioning is the security and privacy considerations. If you are working on sensitive codebases or in regulated industries, you should think twice before deciding whether to use these tools and how.

## Productivity Gains

Let's talk numbers. Last week, I opened these PRs:
- [Feature: Jetpack Activity Logs](https://github.com/wordpress-mobile/WordPress-iOS/pull/24597)
- [Improve TimeZonePickerViewController](https://github.com/wordpress-mobile/WordPress-iOS/pull/24612)
- [Remove TimeZoneObserver from SiteSettingsViewController](https://github.com/wordpress-mobile/WordPress-iOS/pull/24613)
- [Remove unused code](https://github.com/wordpress-mobile/WordPress-iOS/pull/24590)
- [Rewrite Support Activity Logs using SwiftUI](https://github.com/wordpress-mobile/WordPress-iOS/pull/24591)

That's +4,263 lines and −6,018 lines of real code changes in my typical coding style and with a high level of quality. In a normal week, I'd manually write maybe 1-2K lines of meaningful changes. And this was my first week seriously using Claude Code, so I wasn't effective and I spent half the time learning best practices.

Your mileage may vary, but for common tasks, I'm easily **2x more productive**. It unlocks changes I'd never make manually: comprehensive unit tests, documentation, elaborate SwiftUI previews. For these specific tasks, it provides potentially *infinite* productivity gains – generating in seconds what you would never otherwise even attempt to write. Of course, measuring overall productivity is impossible since software engineering involves much more than writing code, but the impact is there.

I'm especially looking forward to sharing the workflows I learned with a team so we could all take advantage of the Claude instances configured to have the same idea of how to write code.

<img class="NewScreenshot" src="/images/posts/claude/share-with-team.png">

## Final Thoughts

> Disclosure: I wrote the entire article myself and then used Claude Code as my editor. It underwent multiple iterations, with Claude reviewing the article according to my stated target audience and goals. The thoughts are my own.
{:.info}

After a week with Claude Code, I'm convinced. It's not about replacing programmers – it's about amplifying them. The better you are at coding and communicating, the bigger the multiplier[^7]. It's still a tool that requires time and skill to master, with plenty of room for learning and min-maxing.

Claude Code works best when you're already an expert who knows what to build and how. It turns the boring parts like writing boilerplate, unit tests, documentation into quick prompts. It also surprises you with novel approaches. I learned more this week than my typical one[^6].

For junior developers, these tools present both opportunity and risk – they can accelerate learning by showing patterns and approaches, but there's danger in accepting generated code without understanding it. The fundamentals still matter.

Some compare AI to the transition from assembly to high-level languages, but I'm not sure that's accurate. You still write and read the same code. The abstractions haven't changed. What's changed is velocity. I can now spend more time on design and architecture instead of mechanical code writing. The other parts are LLMs used as part of the existing products, with English as a sort of programming language for prompts.

Do I have any concerns about code quality or long-term maintenance implications? In my experience – the opposite. I love the craft of engineering software. Our material is code. With AI, you have the power to effortlessly mold it in any shape or form.

{% include references-start.html %}

1. [Anthropic: Mastering Claude Code in 30 minutes](https://www.youtube.com/watch?v=6eBSHbLKuN0)
2. [Anthropic: Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices)
3. [Anthropic: Claude Code Memory Management](https://docs.anthropic.com/en/docs/claude-code/memory)
4. [Anthropic: Prompt Engineering Overview](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)

{% include references-end.html %}

[^2]: Special phrases map to thinking levels: "think" < "think hard" < "think harder" < "ultrathink". These thinking levels actually correspond to different amounts of processing time Claude uses to consider your request. Use higher levels for complex architectural decisions or difficult debugging scenarios.

[^3]: Even Claude Code itself reveals these limitations. It's not unlike [Guided Generation](https://youtu.be/mJMvFyBvZEk?si=3UU1IGSGOMyAmmhz&t=407) in new Apple's Foundation Models where agents combine language models with deterministic tool execution.

[^5]: Claude can search for context automatically, but doing it yourself is faster and more accurate. When Claude searches on its own, it spawns background agents that can take significant time and often include unrelated content. By explicitly using "Read @file" commands, you give Claude exactly the context it needs.

[^6]: For example, I learned a pretty cool async/await pattern where you execute `Task.sleep` inside `while !Task.isCancelled` to implement polling.

[^7]: It could be famous last words, lol. We don't know what the future holds.