---
name: intent-interview
description: >-
  Find out what outcome, audience and measure of success a person wants before building, when the requested thing itself might be the wrong one. Use interactively when those are unclear. Not for choosing an approach ([[design-tree]]) or for unattended runs.
argument-hint: "[the vague request to interview toward]"
allowed-tools: Skill, Read
---

# Intent interview — find out what they want before building it

The project's own instructions win: where its AGENTS.md, CLAUDE.md, design system or established conventions say otherwise, follow them and skip that part of this.

The costly misunderstanding is building the wrong thing well. A person who asks for "a dashboard for our metrics" may turn out to need a list, and no amount of good speccing recovers from that. [[design-tree]] confirms the problem behind a design; this runs earlier, when the requested thing is itself the guess to test.

## When to use it

Use it in a conversation, when a request names a thing to build but not the outcome, the audience or how success will be judged, and you could not predict how the person would answer basic questions about it. When those are already stated, restate them briefly and proceed.

Do not use it where nobody is there to answer (CI, a subagent, a scheduled run). There, write the underspecified ask into the hand-off as a blocker. If the project's instructions say to decide and record the decision yourself, do that.

## The goal

Learn three things: what is true or exists when this is done, who it is for, and how they will know it worked. Also learn the binding constraint and what is deliberately not being built, since much misalignment is silent disagreement about what is out of scope.

Attach your best guess to each question ("Who reads this every day? My guess: the on-call engineer, during incidents"). People correct a wrong guess faster than they compose an answer from nothing, and the guess shows them what you have assumed. How many questions to ask, and whether to ask them together or one at a time, is your judgment: ask what you cannot predict, and stop when you can predict the answers.

Listen for the answer someone thinks they are supposed to give: best-practice vocabulary ("scalable", "clean architecture"), deference to convention, "I should probably". Asking what they would want if they did not have to justify it to anyone often gets the real requirement.

When you have it, restate the intent briefly (outcome, who it is for, how success is judged, the constraint, what is out of scope) and go on once the person agrees. Plain consent is consent: "yes", "sounds good" and "go ahead" all count. If the conversation is not converging, say so and suggest talking about the problem instead of the thing to build.
