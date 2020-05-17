---
layout: post
title: "Trunk-Based Development"
description: Why feature branches might prevent continuous integration and discourage refactoring and what other branching strategies are worth considering.
date: 2018-03-23 18:00:00 +0300
category: programming
tags: programming
permalink: /post/trunk-based-development
uuid: f7b08862-3571-4c61-a7a2-9a5f742fc10a
---

Selecting a good branching strategy is an important step for any team, especially the one that values continuous integration. In this post, I would like to share some of the problems that we had experienced when using [feature branches](http://nvie.com/posts/a-successful-git-branching-model/) and explore another branching strategy - [trunk-based development (TBD)](https://www.thoughtworks.com/insights/blog/enabling-trunk-based-development-deployment-pipelines) - which aims to solve them.

Before we begin, let's quickly review what *continuous integration* is. It often gets associated with a set of *tools* which are used to build and test projects on continuous basis. But we should not forget that it is actually a *practice* which requires developers to apply the ideas behind it:

{% include ad-hor.html %}

> Continuous Integration (CI) is a development practice that requires developers to integrate code into a shared repository several times a day. Each check-in is then verified by an automated build, allowing teams to detect problems early.
>
> *- ToughtWorks. Continuous Integration.*

I would argue that feature branches prevent continuous integration and have some other negative effects like discouraging refactoring. The issue is of course a bit more nuanced than that, so keep in mind that I am mostly going to focus on the downsides of feature branches. I will demonstrate those downsides on five scenarios that we used to constantly encounter in practice in our projects.

## Feature Branches

When I worked on my very first team project, there were four engineers in our team. The only branching strategy that we were familiar with was [GitFlow](http://nvie.com/posts/a-successful-git-branching-model/). One of the main concepts of GitFlow is **feature branches**. The idea is that each feature should be developed in its own branch. When the feature is *done*, it gets merged into `develop` branch. We've ended up adopting this branching strategy in our project and in a relatively naive way - some feature branches could last for days or even weeks which we now know is a common anti-pattern. In other projects that I had a pleasure to work on, we also used GitFlow with a few variations.

The idea of feature branches seems very natural and straightforward at first. Developers are isolated in their own "safe" branches and are able to work in parallel, no unfinished work ever finds its way into the `develop` branch. However, at some point, this work has to be **integrated** and that's where the disadvantages of this model become apparent.

### Uncertainty and Risk of Failing to Deliver

> **Scenario:** You are close to the end of the two week sprint. Alice and Bob both finish their features and create pull requests. There were no previous integrations of their code.

In this scenario, the team now has to spend an unknown amount of time to review the changes, make necessary adjustments and resolve conflicts. It's uncertain how much time this would take and there is a risk of failing to deliver the features that the team had committed to during sprint planning. The team is under pressure to complete the sprint quickly and has a risk of introducing regressions or merging the code that does not meat team's quality standards.

This problem arises even if there is just one feature branch waiting to be reviewed. Nevertheless, more often than not, developers tend to complete the work closer to the end of the sprint.

> Work expands so as to fill the time available for its completion.
>
> *- Parkinson's law.*

### Slow and Ineffective Reviews

> **Scenario:** Bob has finished a feature, which took two days to complete. He's created a pull request with 600 lines of code and 40 files touched. A significant portion of this code is refactoring. After 8 hours of waiting the PR is eventually reviewed. The pull request is merged with a message "looks good to me" after a 15 minutes review.

Large pull requests like that are hard to review. Eventually, the reviews become less effective, less thorough and they also tend to take a lot of time (most of which is spent waiting for a reviewer).

### Big Merge Conflicts, Refactoring is Discouraged

> **Scenario:** Alice has finished working on a feature. She's created a pull request with 1000 lines of code, 500 of which were made as part of refactoring, touching 80 files (renaiming some components, moving a few things around). By the time the review was completed, more changes had been merged into `develop` (integration branch), resulting in PR having merge conflicts.

In my view, the most detrimental problem with feature branches is that this strategy discourages refactoring. GitFlow encourages a 1-1 relation between features and branches. This means that refactoring tends to be part of the feature branch. Developers become wary of refactoring because even a small change may lead to a need to resolve merge conflicts later during integration. Such pull requests are also extremely painful for reviewers. Inevitably, the team ends up avoiding refactoring.

> This fear of big merges also acts as a deterrent to refactoring. Keeping code clean is constant effort, to do it well it requires everyone to keep an eye out for cruft and fix it wherever they see it. However this kind of refactoring on a feature branch is awkward because it makes the Big Scary Merge much worse. The result we see is that teams using feature branches shy away from refactoring which leads to uglier code bases.
>
> Indeed I see this as the decisive reason why Feature Branching is a bad idea. Once a team is afraid to refactor to keep their code healthy they are on downward spiral with no pretty end.
>
> *- Martin Fowler. "FeatureBranch"*

### Prevents Continuous Delivery

> **Scenario:** Alice is working on feature A in a feature branch. The team (e.g. product owner, UX designer) is interested in receiving each new iteration of this feature to see how good well the ideas have turned out to be and start producing feedback as soon as possible.

*Continuous delivery* is an extension of continuous integration. If you practice *continuous delivery* you would normally have a single integration branch with an automated workflow, which produces nightly build (your situation might be different). In order to deliver the features to the customer, the feature branch needs to be merged into the integration branch, but you can't do this unless the feature is *done*. As a result, you end up using some ad-hoc way to deliver a build.

> **Scenario:** Alice is working on feature A, Bob is working on feature B. There is a need to deliver an increment of both features in a single build to see how well they work together.

This scenario often results in an ad-hoc solution like a separate integration branch created just for these two features.

### Duplicated Work, Hard to Share Dependencies

> **Scenario:** Alice is working on feature A, Bob is working on feature B. Bob has already implemented some component C and it has already been incorporated into his own branch as part of a bigger commit. Now, Alice realizes that she also needs this component.

It takes a large amount of effort to organize the work on feature branches in a way that there are no shared dependencies between branches. And there is often no agreed-upon mechanism for sharing this work. In the worst case scenario, if your team is dogmatic in terms of not merging unfinished work in a `develop` branch, you might end up creating yet another integration branch, apart from `develop`, just for those two features.

### Postponed Integrations

> **Scenario:** Bob has finished a feature, but the team doesn't merge it because they don't consider the feature to be **done** due to the factors out of Bob's control (e.g. they're waiting for updated graphics from the designer).

As a result, integration is postponed, Bob might need to rebase the work later, the team can't start using Bob's work.

## Trunk-Based Development

If you've ever used feature branches, chances are you had encountered at least some of the scenarios above. Fortunately, many of those problems are largely mitigated by decomposing stories properly and not having feature branches that live for a long time. However, in practice the work on even relatively small feature branch might take a lot of time in some cases due to the factors out of your control. And there is always this mismatch between how you split the stories and how you perform the actual work (e.g. refactoring step, adding some shared components, etc - the work that you want to be integrated as soon as possible).

The shorter lived the branches are, the closer you get to a [Trunk-Based Development](https://trunkbaseddevelopment.com) (TBD). The core principles behind TBD are actually rather simple. In TBD developers always check into one branch (either via pull requests or directly), typically `master` which is also called "trunk". Trunk is a single integration point which is used for continuous integration and delivery. This trunk is always stable, but it can have unfinished features covered by [feature toggles](https://martinfowler.com/bliki/FeatureToggle.html).

### Continuous Integration

TBD enables continuous integration - developers can integrate their work more quicker and easier. You no longer end up in situations where developers are isolated in their own branches with no feedback from the team.

### Cheap Decomposition

Decomposition becomes cheap. You're no longer limited by a 1-1 relation between features and branches mentality. When working on a story, you can complete refactoring on Monday and get it merged immediately, then create some shared component on Tuesday so that your colleague working on a related feature can start using it right away, and finally finish the feature on Friday, when you receive all the remaining assets from the designer and merge the feature with no conflicts.

### Fewer Merge Conflicts

Small incremental changes lead to smaller and less frequent merge conflicts. There are no reasons to hold back on refactoring anymore.

### Continuous Code Review

It also enables [*continuous code review*](https://trunkbaseddevelopment.com/continuous-review/) - the idea that instead of reviewing large pieces of code infrequently you do it often, quickly, and also painlessly.

> There is a cost to multi-tasking, so maybe someone in the dev team who is between work items at that moment should focus on the review before they start new work. With a continuous review ethos, it is critical that code reviews are not allowed to back up.
>
> *- Paul Hammant, Steve Smith and friends. "Trunk Based Development"*

As a result, TBD reduces uncertainty and risk and allows to ship incremental changes to the customer quickly and reliably.

## Conclusion

There are some clear disadvantages of feature branches which often end up preventing very valuable engineering practices, like continuous integration and refactoring. There are also some clear benefits of trunk-based development.

Please keep in mind that TBD is not against branches. It's perfectly fine to have short-lived feature branches, but limiting your team to a 1-1 relation between a feature and a branch might not be the best choice.

Many developers who I talked with have adopted feature branches a few years ago and haven't re-evaluated their decision since. The point of this post is to promote TBD in the mobile app developers community. There are a lot of popular projects that use TBD principles, [Swift](https://swift.org/contributing/#contributing-code) itself is one of them.

To me, personally, TBD seems like "a 25-dollar term for a 5-cent concept", but it is an important one. And it just might be the case that you're doing TBD without calling it this way.

In this post, I've completely avoided how releases are managed in GitFlow and in TBD (I think GitFlow has a bit of an unwanted overhead and complexity). There are a lot of resources on both techniques, which you can jump into to see the differences.

There are definitely better articles about trunk-based development (see references), I encourage you to check them out.

## References

- [Paul Hammant, Steve Smith and friends (2017). Trunk Based Development.](https://trunkbaseddevelopment.com/context/)
- [Apple Inc (2018). Swift: Contributing Code, Incremental Development.](https://swift.org/contributing/#contributing-code)
- [Vishal Naik (2015). Enabling Trunk Based Development with Deployment Pipelines.](https://www.thoughtworks.com/insights/blog/enabling-trunk-based-development-deployment-pipelines)
- [Neal Ford (2015). How CI removes the pain.](http://radar.oreilly.com/2015/03/how-ci-removes-the-pain.html)
- [Robert Ecker (2016). Code Reviews in Trunk Based Development.](https://team-coder.com/code-reviews-in-trunk-based-development/)
- [Martin Fowler (2009). FeatureBranch.](https://martinfowler.com/bliki/FeatureBranch.html)
- [Martin Fowler (2010). FeatureToogle.](https://martinfowler.com/bliki/FeatureToggle.html)
- [Vincent Driessen (2010). A Successful Git Branching Model.](http://nvie.com/posts/a-successful-git-branching-model/)
