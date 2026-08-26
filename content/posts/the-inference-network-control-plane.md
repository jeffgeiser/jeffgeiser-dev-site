---
title: "The Inference-Network Control Plane"
date: 2026-08-28
draft: true
description: "A name for the missing layer between inference observability and programmable interconnection — and the metric that would tell you whether it's working: delivered cost per token."
tags: ["inference-infrastructure", "decision-loop"]
---

Two pieces led here. [The Layer Below MCP](https://jeffgeiser.dev/posts/the-layer-below-mcp/) named a decision loop — Sense, Score, Commit, Reconcile — that sits underneath any single inference call. [The Decision Loop Is Fractal](https://jeffgeiser.dev/posts/the-decision-loop-is-fractal/) argued that loop repeats at every scale, from a single agent up to a whole network deciding where inference should run.

At the top of that scale, the loop stops being a routing detail and becomes an infrastructure question: who — or what — decides, across a network of many operators and many nodes, where a given inference request actually gets served? Today the honest answer is usually "nobody, deliberately." Requests land wherever a static config or a simple heuristic sends them, and the network as a whole never asks whether that placement was any good.

I think that's a missing layer, and it sits in a specific gap. On one side, inference observability tools (Wicklee among them, at a single-operator scale) can tell you what a resource is actually costing and how it's actually performing. On the other side, programmable interconnection — the networking layer that already exists to move traffic between providers and regions on demand — can act on a decision once it's made. Neither one answers the question in between: given what we can observe, and given what the network can act on, where should this request go? That's the inference-network control plane. Not a product I'm building, and not a mechanism I'm going to describe here — a name for a layer that doesn't have one yet, sitting between two layers that already do.

Naming a layer is only useful if you can tell whether it's working. I don't think uptime or raw throughput are the right measures for this one — a network can be up and fast while still routing every request through the most expensive path available. The metric I keep coming back to is delivered cost per token: not list price, not a theoretical floor, but what a token actually cost to deliver, end to end, given where it was actually routed and what actually happened on the way. It's the network-scale version of the same honesty Wicklee tries to bring to a single fleet — you should be able to see the real number, not the vendor's number.

I don't have a system to announce here, and I'm being deliberately vague about implementation, for the same reason as the last two pieces: some of the mechanism-level thinking behind this overlaps with patent-pending work tied to my employer, and isn't mine to publish yet. What I do think is worth putting on the record now is the shape of the problem and the name for it — because I expect more of the infrastructure world to start converging on some version of this question as inference keeps moving off single-provider APIs and onto a mix of local, edge, and networked resources. Better to name the layer clearly before everyone builds a slightly different, unnamed version of it.
