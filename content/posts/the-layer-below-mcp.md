---
title: "The Layer Below MCP"
date: 2026-08-26
draft: false
description: "MCP standardizes how an agent calls a tool. It says nothing about which resource should serve that call right now — and that gap is where cost, latency, and reliability actually get decided."
tags: ["inference-infrastructure", "decision-loop"]
---

MCP solved a real problem: it gave agents a standard way to discover and call tools, instead of every integration being a bespoke adapter. That's genuinely useful, and I don't want to undersell it.

But standardizing the call doesn't standardize the decision behind it. When an agent needs an inference request served, MCP tells it how to ask. It doesn't tell it — or anything upstream of it — which of the available resources should actually answer: which GPU, which model, which node, at what cost and what latency, right now, given that all of those numbers were different five minutes ago and will be different again in five more.

That's the layer below MCP. It isn't a protocol problem. It's a decision problem, and today most systems solve it by not really solving it — a hardcoded default endpoint, round-robin load balancing, whichever provider answers fastest. Those approaches work fine until the resource picture changes: a GPU throttles, a node drops offline, a smaller model turns out to be good enough for the task, spot pricing moves. Then the decision that was never actually made starts costing money or availability that nobody notices until the bill or the outage arrives.

I think the decision that's missing has a shape, even without prescribing how any particular system should implement it. I've been calling it a loop, because it isn't a one-time choice — it has to keep running as conditions change.

**Sense.** What resources are actually available right now, and what do we actually know about their current state — cost, latency, load, thermal headroom, whatever matters for the workload? Not what the spec sheet says. What's true this second.

**Score.** Given the request in front of us, how does each candidate resource compare against the objective that matters — cheapest, fastest, most available, some blend of the three? This is where "it depends what you're optimizing for" stops being a shrug and becomes an actual input.

**Commit.** Pick one, route the request, and — this part matters — record what you expected to happen. Expected cost, expected latency, expected outcome.

**Reconcile.** Compare what actually happened to what you expected. Feed the gap back into Sense. This is the step almost everyone skips, and it's the one that turns a one-shot routing decision into a system that gets better at making the next one.

None of this is exotic. It's closer to how a good ops team already thinks about capacity planning than to anything novel. What's missing is that nobody names it as a layer of its own — it gets buried inside individual agent frameworks, individual internal scripts, each reinventing a worse version of the same four steps. MCP standardized the call. Nothing yet standardizes the decision behind the call.

I don't have a system to sell you here, and I'm intentionally not describing an implementation — partly because I don't think one architecture is obviously right yet, and partly because some of my current thinking on this overlaps with patent-pending work at my employer that isn't public. What I do think is worth saying out loud: this gap is real, it's going to get more expensive to ignore as more inference moves off a single provider's API and onto a mix of local, edge, and cloud resources, and naming it clearly is a useful first step even before anyone agrees on how to close it.

This is the first piece in a short series on how distributed systems make resource decisions — the same loop turns out to show up at more than one scale. The rest of it is still being drafted.
