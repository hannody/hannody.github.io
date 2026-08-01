---
title: "Spec-Driven Development with AI Agents: Constitutions, Checkpoints, and Handoffs"
date: 2026-08-01
draft: false
slug: "spec-driven-development-with-ai-agents"
tags:
  - ai-agents
  - spec-driven-development
  - context-engineering
  - code-review
  - llm
  - developer-workflow
---

AI coding agents are strangely lopsided. Hand one a well-scoped task and it writes better code, faster, than I do. Hand it a *large* feature, the kind that takes days and touches a dozen files across several layers, and the failure mode changes. The context window fills up and quality quietly degrades (AKA **context rot**). A fresh session, missing the history, confidently invents an API that never existed. And even when the code is good, you're left staring at a 3,000-line diff you can't meaningfully review.

Those failures are not solved by the model alone. They are largely a *process* problem.

The backbone I work inside isn't mine and isn't novel: a constitution, specs, and deliberately stabilized plans. It's spec-driven development, which has been around in various shapes for years. What I added from running it day to day were two operational mechanisms that made it survive long features: explicit **checkpoints** and structured **handoffs**.

The mental model I use: **treat the agent like a brilliant but forgetful senior engineer.** Sharp in the moment, no long-term memory, occasionally overconfident. You wouldn't hand that person a two-week feature and walk away. You'd give them the ground rules, a spec, and a plan, then review their work in slices. That's the whole idea.

## The loop: constitution → spec → plan → implement

Standard stuff, so I'll keep it short.

- **Constitution.** A short, binding document of principles the project may never violate: dependency direction, the testing bar, security rules, "spec before code." It's the highest authority; when a request conflicts with it, the conflict gets *surfaced*, not silently coded around.
- **Spec.** WHAT and WHY only. User value, functional requirements, acceptance criteria. No tech, no code.
- **Plan.** HOW. Architecture, contracts, and an ordered list of tiny, test-first tasks, each justified against the constitution. Crucially, the plan is **frozen** once approved: it's the source of truth, and the agent doesn't get to quietly rewrite it mid-flight.

That gives the agent rails. But rails aren't enough for long work.

## Checkpoints: break the feature into slices you can actually review

A plan might have 25 tasks. Executing all 25 in one shot lands you right back at the unreviewable-diff problem. So I slice the plan into **checkpoints**, review-sized groups of tasks, and the agent works one checkpoint at a time, then **stops**.

At each stop it has to: get the gate green (lint, types, architecture checks, tests), tick off the tasks, write a short note recording *every deviation* from the plan, and hand off for review.

The review itself is two-stage. The coding agent doesn't send its work straight to me. It writes a brief for a separate **reviewer agent**, stating what changed and why. I read that brief first, because what the coder chooses to explain (and what it leaves out) tells me where to look. Then the reviewer agent goes to work on the diff, and I read its findings against my own. The slice is a few hundred lines, one coherent unit, so this is actually tractable. Approval is mine either way; nothing continues on an agent's say-so.

Two things fall out of this:

1. **Complexity stays bounded.** I'm never reviewing "a feature"; I'm reviewing a checkpoint. Drift and bad assumptions get caught at checkpoint 2, not discovered at checkpoint 7.
2. **Verification is real.** Because each checkpoint ends on a green gate *with evidence*, "done" means something. I run the commands and read the output before I believe it.

The idea of pausing between steps isn't new. What I've made repeatable is the artifact: a shared checkpoint tracker that both the human and the agent write to, so stopping-for-review is the default, not an afterthought.

## Handoffs: kill context rot and hallucination

Long features outlive a single agent session. The context window fills, or I come back the next day, or I switch models. A fresh session starts with *zero* memory of the last three days of decisions. Left alone, it will re-litigate settled choices or hallucinate the current state.

I learned this the tedious way. Fresh sessions reintroduced an API we'd deleted two days earlier, reopened architectural decisions that were already settled, quietly dropped an invariant because the reasoning behind it had scrolled out of the window. After the third time reviewing the same decision, I stopped waiting for it to happen.

So I don't wait for a session to degrade. I retire it on purpose and have it hand the work to a fresh agent on my terms.

Every feature carries a **HANDOFF** document, a living page that tells the next session exactly what it needs and nothing it doesn't:

- Orientation: which docs to read, in order (constitution, spec, the frozen plan, the checkpoint tracker).
- The hard rules it must not break.
- Current state: branch, commit, which checkpoints are done.
- **Decisions already made, with the *why***, so they don't get reopened.
- The next task, and the items consciously deferred.

A fresh agent reads the handoff and the frozen plan, and it's grounded again. No rot, no invention, because the ground truth lives in the documents rather than in a context window that's about to overflow.

### Six sessions, one document

On a recent AI-system backend feature, six different agent sessions touched the code across its seven checkpoints. Each started from a fresh context, some were different models, and none of them needed me to re-explain a decision that had already been made. I read every handoff as it was written, and tightened the template whenever a fresh session asked a question the document should have already answered.

## Tiering models by leverage

One more lever: not every step deserves the same horsepower. Planning and code review are the highest-leverage, hardest-to-reverse steps, and a bad plan poisons everything downstream. Implementation against a *frozen, detailed* plan is comparatively mechanical.

So I tier:

- **Top reasoning tier for plans and the reviewer agent** (currently Fable 5), where judgment compounds.
- **A tier down for the actual coding** (Opus 5, or Sonnet 5 on the mechanical stretches), where the plan and the checkpoint gate are already the guardrails.

The human review at each checkpoint is the safety net that makes the cheaper coding pass safe. It's a nice cost/quality trade: spend where thinking matters, save where the rails are already in place.

## The whole thing, on five lines

```text
Request → Constitution → Spec → Frozen Plan
    │
    └─> ( Checkpoint → Gate → Review ) × N
              ↑                    │
              └───── HANDOFF ◀─────┘
```

## What it buys me

- **Reviewable diffs.** Always a checkpoint, never a monolith.
- **Grounded agents.** The handoff plus the frozen plan beat context rot and hallucination.
- **Real verification.** A green gate and evidence at every stop.
- **Cost control.** Heavy models where they earn it, a tier down where the rails already hold.
- **Continuity.** The feature survives any single session ending.

The surprising lesson: as agents get better at *writing code*, the bottleneck moves to **keeping them grounded and keeping yourself in the loop**. A constitution, checkpoints, and handoffs are how I do both, and honestly, they've made me a more disciplined engineer even on the parts I write by hand.

I've templated the whole thing (constitution, spec/plan templates, the checkpoint tracker, and the handoff doc) as a starter kit I drop into new projects: **[hannody/spec-kit](https://github.com/hannody/spec-kit)**. It's language- and domain-agnostic; fork it and tune the constitution to your stack.
