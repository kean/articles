---
layout: post
title: PR Reviews Are for Humans
subtitle: The non-negotiables of PR review when agents write the code
description: "The non-negotiables of PR review when agents write the code: proof that the change works before the PR is open, and a human who owns what gets merged."
date: 2026-10-29 09:00:00 -0400
category: programming
tags: [tools, testing]
permalink: /post/pr-reviews-are-for-humans
uuid: 7c58a70f-d838-4b67-bed0-1a5018bf67df
---

Agents write most of my code now, and they open most of my PRs too. And almost every repo I work in has an AI reviewer waiting for them, like [CodeRabbit](https://www.coderabbit.ai), [Copilot](https://github.blog/changelog/2025-04-04-copilot-code-review-now-generally-available/), or Anthropic's [Code Review](https://claude.com/blog/code-review). One agent opens a PR, and only then does another one review it. I think it's backwards.

We took a process designed for humans and taught machines to mimic it. To be fair, these reviewers do find real bugs[^1], and of course every change should go through one – but why wait until the PR is open? The review should be part of making the change, along with the proof that it works. Once you have both, there's no point in an agent reviewing it again, and the review that's left is for humans.

Others have come to similar conclusions – see the [references](#references). Here's how it looks on an iOS app.

## Review Before the PR

Here's how I do it now, though I'm still iterating on it. Most of my bug fixes go through a `/fix-issues` skill: it takes the next N issues from Linear and fixes each one in its own worktree and simulator. The review happens before the PR is even open: the agent runs `/code-review` on the diff, fixes whatever holds up, and retests. I keep it quick, and so far it's enough: since I added it, the repo's AI reviewer has flagged 1 of 11 PRs, and only for a nit, down from 10 of 23 on the same kind of fixes before.

But a review of the diff only sees what changed, not what's around it. So a second agent looks at how the change fits into the codebase: duplicated code, a helper it could've reused, a pattern it doesn't follow, or a caller it could break. That's what I'd otherwise read the diff for. Neither review is what gives me confidence in the change, though – the proof is.

## Prove It!

I don't want agents to tell me a change works. I want them to show me, with something I can verify in a minute without building the branch:

- **Reproduce the issue first** on a build without the fix. If it doesn't reproduce, the agent stops.
- **Before and after screenshots**, plus **walkthroughs** when a still can't show the change, with the steps and logs next to the video.
- **Unit tests** that fail without the fix, when the logic can be tested.
- **Checking the areas the change could affect**, more of them the riskier the change.

Not that long ago, this would've been way too much work for every PR. Now agents do it easily with the Xcode 27 [MCP server](https://developer.apple.com/documentation/xcode/giving-external-agents-access-to-xcode) and simulators, the same setup I used for 1,700 screenshots in [Pocket Casts on iPhone Duo]({{ site.url }}/post/pocket-casts-iphone-duo). Here's [#5539](https://github.com/Automattic/pocket-casts-ios/pull/5539), which fixes the mini player's progress row in right-to-left languages:

<figure class="PostFigure">
<img class="NewScreenshot" alt="Before and after screenshots of the iPad mini player in a right-to-left language. Before, the remaining time is at the left of the progress bar and the elapsed time at the right. After, the elapsed time is at the left and the remaining time at the right." src="/images/posts/pr-reviews/pr-5539-ipad.webp" width="1200" height="418" loading="lazy" decoding="async">
<figcaption>The before and after on iPad in RTL, straight from the PR</figcaption>
</figure>

It's a layout fix, so screenshots are all the proof it needs. The [PR](https://github.com/Automattic/pocket-casts-ios/pull/5539) has the rest: the root cause, the risk, the steps to test, and the results. The repo's AI reviewer still went over it twice and found nothing, which is exactly how it should be: the review before the PR had already done its job.

## Non-negotiables

So what is the review for, then? It's for humans. You're responsible for what gets merged, and AI can't be responsible for anything.

<figure class="PostFigure PostFigure--narrow">
<img class="NewScreenshot" alt="A slide that reads: A computer can never be held accountable. Therefore a computer must never make a management decision." src="/images/posts/pr-reviews/ibm-1979.webp" width="768" height="711" loading="lazy" decoding="async">
<figcaption>From an internal IBM training in 1979 (<a href="https://simonwillison.net/2025/Feb/3/a-computer-can-never-be-held-accountable/">via Simon Willison</a>)</figcaption>
</figure>

That leaves two non-negotiables: proof that the change works, and a human who owns the merge. Everything else is negotiable, including reading the code. I always review the description and the artifacts, and for small changes, they often tell me more than the diff would. I ask the agent to rate the risk of every PR, like `**Risk:** Low. Only the mini player's progress/time row layout changes.`, and it maps to:

- **Low**: no code review, like a constraint fix on one screen.
- **Medium**: one reviewer, which is me. The agent wrote the code, so I'm already the second pair of eyes.
- **High**: two reviewers, for critical changes – playback, sync, purchases, sign-in – where it's worth having a teammate look too.

I'm not the only one doing this. The Claude Code team [uses](https://claude.dev/blog/solving-code-review-at-anthropic/) the same three tiers – zero, one, and two human approvals – and a rule I've borrowed: "the change sets the tier, not the author." The agent's rating is only a starting point. A script checks the diff and can only raise it: anything in the critical areas is High, and a change to shared code, existing tests, or CI, or to more than a handful of files, is at least Medium. Screenshots show what changed on screen, not what else the diff touched.

Where we differ is what can skip code review. For them, it's docs, copy, internal-only features, and additive changes behind a flag – changes that are safe by nature. For me, it also includes small fixes users can see, like the RTL one above, where the screenshots are the proof. Either way, you have to be honest with yourself and bump the rating whenever a change deserves a closer look.

## Beyond the Diff

PRs used to be a short description and a diff. Now the description matters more than the diff.

But the tools are still built around the diff. My PRs have it all, but they still feel so informal and basic – it's just Markdown and a bunch of emoji[^2]. I think the tools need to focus less on code and more on everything else:

- **A report** with Dynamic Type, accessibility checks, and walkthroughs on different devices.
- **Proof that performance didn't regress**, like launch time, memory, and hitches from the same steps on both builds. "No regression" should be a number, not a feeling.
- **Issues found along the way**, so the testing time doesn't go to waste. If the agent runs into a bug in the related code, it should check whether it's already in the tracker – Linear, in our case – and report it in the PR too.
- **A check against the design**, so a designer can compare the screenshots with the Figma frames without ever seeing the code.
- **A guided tour of the diff** with the few lines that matter called out. Shopify's agents [already narrate](https://shopify.engineering/remote-dev) videos explaining what they did.
- **A simulator in the browser** with the app in the exact state the agent tested, mocks included, so I can pick up where it left off. [Expo](https://docs.expo.dev/preview/simulators/how-to/review-a-pull-request/) and [SimDeck](https://simdeck.sh/guide/github-actions) already run a PR's build in the browser; the state is the missing part.

Some of this already exists, and I use a few pieces, like the Figma and Linear MCP servers. Others need integrating one by one, and I want a more complete solution. Ideally, it would guide me through the change like an onboarding flow in an app: "tap here, look at this."

## Keeping Up

Code review is also how a team learns what changed. That was part of why Google [introduced](https://research.google/pubs/modern-code-review-a-case-study-at-google/) it in the first place; finding bugs was never the only point. So fewer code reviews means fewer chances to share knowledge and keep the codebase consistent. Agents can help here too: have one send you a weekly brief with only the important bits, like a new pattern or an API that changed, plus a video walkthrough of what changed in the app.

You still need to understand the codebase, though – the architecture, its limits, the critical pieces – or you won't know what you're looking at.

## Final Thoughts

I think our goal should be finding more use cases for agents, not blindly automating what worked before. If an agent writes the code, reviewing and proving it is just part of writing it, and it all happens before the PR is open. With good enough proof, you don't even need to read every diff. What's left is for humans: deciding what gets merged and taking responsibility for it. PR reviews are for humans.

Happy shipping!

{% include references-start.html %}

- [Your job is to deliver code you have proven to work](https://simonwillison.net/2025/Dec/18/code-proven-to-work/) (Simon Willison)
- [What is happening with code reviews?](https://newsletter.pragmaticengineer.com/p/what-is-happening-with-code-reviews) (The Pragmatic Engineer)
- [How the Claude Code team does code review](https://claude.dev/blog/solving-code-review-at-anthropic/) (Anthropic)

{% include references-end.html %}

[^1]: Greptile [argues](https://www.greptile.com/blog/ai-code-reviews-conflict) that it's not silly to have AI review AI-written code, because each review starts with a fresh context. I agree, and that's why I run one – just before the PR.

[^2]: My agents used to upload the screenshots and videos to secret gists, which anyone with the link can open. That's fine for an open source app, but agents uploading them outside private repos have [leaked](https://thehackernews.com/2026/09/ai-coding-agents-exposed-13000-internal.html) thousands of internal screenshots. Now `gh` can [attach](https://docs.github.com/en/github-cli/github-cli/attaching-files-with-github-cli) them to the PR directly, and they stay as private as the repo.
