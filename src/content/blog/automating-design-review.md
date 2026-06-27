---
title: 'We built a linter for design files'
description: "Code has linters. Designs don't. On a large mobile team, the gap between a Figma file and a shipped screen is full of the same mistakes, made over and over. Here's what happened when we automated the review of designs the way we automate the review of code."
pubDate: 'Jun 27 2026'
---

Every mature codebase has a linter. You don't argue about trailing commas in code review because a machine already did. The cost of that argument got paid down to zero a long time ago.

Design review has no such thing. On a large mobile team, every screen passes through a manual review where someone — usually an engineer, sometimes a designer, often both, never the same way twice — checks whether the design handles right-to-left layouts, whether every error state is specified, whether dark mode is covered, whether the text doesn't overflow at the largest font scale. The same checklist, every time, performed by humans, inconsistently.

I spent a while building the thing that fixes this: a linter for design files. This post is about what that taught me, because the lesson generalizes well beyond design.

## The observation that started it

If you watch what an experienced engineer actually does when a design lands in their queue, most of it is mechanical. They're not exercising taste. They're running a checklist:

- Is there a right-to-left version, or at least confirmation that mirroring is handled?
- Is every interactive element's error state designed, or just the happy path?
- Is there a loading state? An empty state? A no-network state?
- Does the design exist in both light and dark mode?
- At the largest accessibility font size, does anything clip or overlap?
- Are the screens in a flow actually connected — can you trace the full click journey, or are there dangling screens with no entry point?
- Are the same components used consistently, or has someone hand-drawn a button that should have come from the design system?

None of this requires judgment. All of it requires attention. And attention is exactly the thing that degrades when a human does the same checklist for the fortieth time this month.

That's the signature of an automatable task: mechanical, repetitive, attention-dependent, and currently done by expensive people who'd rather be doing something else.

## What the tool does

The linter runs against a design file and reports violations in-place, the way a code linter annotates a diff. The categories it checks:

**Right-to-left coverage.** Flags screens that have left-to-right layout assumptions baked in without a mirrored counterpart or an explicit "RTL-safe" annotation.

**State completeness.** For any screen with an interactive element, checks that the obvious states exist — error, loading, empty, success. A login form with no error state is a finding.

**Click-journey completeness.** Builds a graph of screens and the links between them, then flags orphan screens (no incoming link) and dead ends (no outgoing link where one is expected). This catches the "we forgot to design the back-navigation" class of bug before it reaches an engineer.

**Light/dark parity.** Flags any screen that exists in one mode but not the other.

**Font scaling.** Simulates the largest supported accessibility font size and flags text that would clip or overlap its container.

**Component consistency.** Detects elements that look like design-system components but aren't actually instances of them — the hand-drawn button, the off-by-2px spacing, the not-quite-right shade of the brand color.

For each finding, it posts a comment back onto the design itself, where the designer and engineer are already looking. No separate report, no new dashboard to check. The feedback shows up in the tool people already use.

## The part that mattered: feedback goes where the work is

The first version of the linter produced a report. A nicely formatted document listing every violation. Nobody read it.

This is the single most important thing I learned, and it's not about design. **Automation that produces a separate artifact gets ignored. Automation that injects feedback into the existing workflow gets used.**

A code linter that emailed you a PDF of style violations would be useless. The reason linters work is that they annotate the diff, in the editor, at the moment you're looking at the code. The feedback is co-located with the work.

When I moved the design linter from "generates a report" to "posts comments directly onto the design file, in-place," usage went from near-zero to default. Same checks. Same findings. The only thing that changed was *where the feedback appeared.* It appeared where people were already looking.

If you take one thing from this post: when you automate a review process, the output has to land inside the surface where the work already happens. Anything that asks people to go somewhere new to receive feedback has already lost.

## What it's worth

The naive way to value a tool like this is "minutes saved per design review." That undersells it.

The real value is in the violations that *used to slip through.* A manual checklist run forty times a month will miss things — not because the reviewer is careless, but because human attention is not uniform. The missed RTL layout ships, a user in an RTL locale hits a broken screen, it becomes a bug report, then a fix, then a re-review. The cost of the miss is an order of magnitude higher than the cost of the original check.

A linter doesn't get tired on the fortieth review. Its attention on review forty is identical to review one. The value isn't the time saved on the reviews that would have passed anyway — it's the bugs prevented on the reviews where a human would have blinked.

This is the same reason code linters are worth it. Not because formatting arguments are expensive, but because the bug a tired reviewer misses is expensive.

## The generalizable pattern

The design linter is one instance of a pattern I now look for everywhere on a large team:

1. **Find the mechanical checklist.** Somewhere, an expensive person is running the same checklist repeatedly, by hand, with degrading attention. Release verification, design review, dependency audits, analytics-event coverage, accessibility passes — every team has several of these.

2. **Encode the checklist as rules.** Most of these "judgment" tasks turn out to be 80% mechanical rules and 20% actual judgment. Automate the 80%. Leave the 20% to humans, who now have the attention to spend on it because they're not drowning in the mechanical part.

3. **Put the output where the work is.** Not a report. Not a dashboard. An annotation on the artifact, in the tool, at the moment of review.

4. **Let it run on everything, every time.** The value compounds on the high-volume, low-glamour reviews — the ones humans rush.

The design linter is being extended now to also check analytics-spec coverage — whether every interactive element that should fire a tracking event has that event specified in the design. Same pattern, new checklist. Once you have the machinery to inject automated review into the design surface, every new mechanical checklist is cheap to add.

## The honest caveat

This doesn't replace design review. It replaces the *mechanical* part of design review. A linter can tell you a screen has no error state; it cannot tell you the error state is confusing. It can tell you a button isn't a design-system instance; it cannot tell you the layout is ugly. Taste is still human.

What it does is clear the mechanical noise so that human review can spend its attention on the things that actually need taste. That's the whole game with developer-experience work: not replacing people, but moving their attention up the value chain by automating the part below it.

---

*I write about mobile payments engineering and developer experience on large Android teams. I'm at [rishabhships.com](https://rishabhships.com) or [rishabh@rishabhships.com](mailto:rishabh@rishabhships.com).*
