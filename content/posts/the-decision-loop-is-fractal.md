---
title: "The Decision Loop Is Fractal"
date: 2026-08-27
draft: true
description: "The same Sense/Score/Commit/Reconcile loop that governs a single resource call turns out to repeat at every scale — up to the point where an entire network is making it."
tags: ["inference-infrastructure", "decision-loop"]
---

In [The Layer Below MCP](https://jeffgeiser.dev/posts/the-layer-below-mcp/) I described a decision loop — Sense, Score, Commit, Reconcile — that sits underneath any single inference call: which GPU, which model, which node answers this request, right now.

The more I look at it, the more that loop doesn't look like something that happens once, at one scale. It looks fractal — the same shape recurring whether you're looking at a single agent or an entire network of them.

An individual agent runs it when it picks which local resource should serve a request: sense what's available, score the candidates, commit to one, reconcile against what actually happened.

A node running multiple models runs the same loop when it decides which model on the box should take an incoming request — a bigger model waiting on a queue, a smaller one idle and ready.

A cluster runs it when it decides whether to serve a request locally or hand it off to a neighboring node with more headroom.

And a network — many operators, many nodes, no single owner — runs it when it decides, request by request, where inference should actually happen: whose hardware, at what cost, under what latency budget.

Same four steps, different scope, different stakes. What changes as you go up in scale isn't the shape of the decision — it's who's making it, how much they can see, and how expensive it is to get wrong. An agent that mis-scores a GPU wastes a few seconds. A network that mis-scores where inference should run wastes real money and real capacity, over and over, at a scale nobody manually audits.

I think this is worth naming plainly, because right now each scale gets solved separately, by different teams, with different tools, and none of them are talking to each other about the fact that they're solving the same problem. An agent framework's routing logic and a network's placement logic are answering the same four questions. That's not a coincidence — it's the same loop, fractal.

I'm not going to describe how any particular layer should implement this — that's a harder and more contested question than this post is trying to answer, and part of it overlaps with work that isn't mine to publish yet. What I want to establish here is just the shape: this is one problem showing up at every scale, not many unrelated problems that happen to look similar.

The next piece names what that problem looks like at the top of the stack — the network scale — and the metric that would tell you whether it's being solved well.
