---
title: "Test the Stack, Not the Diagram"
date: 2026-10-06
draft: true
description: "Data in an AI stack doesn't live in one place. It sprawls across caches, logs, vector stores, and traces. Residency is a surface, and you can't prove a surface from a diagram."
tags: ["sovereign-ai", "data-residency", "compliance", "bordercheck", "ai-governance"]
---

**Data in an AI stack doesn't live in one place. It sprawls across caches, logs, vector stores, and traces. Residency is a surface, and you can't prove a surface from a diagram.**

*By Jeff Geiser — October 2026*

---

If you've built an AI stack, you already know the uncomfortable version of this: your data does not live in one place.

You designed a request path. Prompt comes in, hits the gateway, goes to a model, answer goes back. That path is what's on the architecture diagram, and it's real. It's also a small fraction of where the data actually goes.

A real stack isn't the request path. It's the request path plus everything you bolted on to make it fast, reliable, and debuggable. The gateway logs the request. The retrieval step embeds the prompt and writes the vector to a store. The model holds the whole thing in a KV cache for the length of the generation. The observability pipeline ships the full trace somewhere so you can debug it later. A semantic cache keeps the prompt and the answer around so the next similar request is cheap. And when the local model is saturated or down, the fallback path sends the request to a frontier API to keep the lights on.

Every one of those is a place your data came to rest, or a path it left by. None of them is a bug.. they're all good engineering.. and almost none of them are on the diagram, because the diagram shows the flow you designed, not the data-at-rest sprawl your infra added underneath..

So **residency is a surface, not a line.** You can't draw it as a boundary on a slide, because the data isn't sitting on a boundary. It's smeared across a dozen components.. most of which exist specifically to copy it somewhere for latency, reliability, or debuggability. Or whatever..

## Why the paper version catches nothing

This is why residency reviews that stay on paper find nothing real. The policy says data stays local. The diagram backs it up. But the policy and the diagram both describe the request path, and the leak isn't on the request path.

It's in the trace pipeline that forwards to a hosted observability vendor in another jurisdiction. It's in the vector store someone stood up in a different region without thinking about where it lived. It's in the semantic cache that is now a durable copy of every prompt you've ever served. It's in the gateway access log that captured the full payload at info level. You will not find any of that by reading the diagram harder. The only way to know where the data went is to put known data in and go look in every place it could have come to rest.

## What bordercheck actually does

So I built bordercheck... Not to check one path. To inspect the whole residency surface.

It mints a canary: uniquely shaped data in valid-but-nonexistent formats, so it exists nowhere else and a hit is unambiguous about how it got there. It runs that canary through the real stack, the actual gateway, under normal operation and under stress. Then it goes and looks, in every layer where data comes to rest: the system of record, the processing tier, model state, the logs and control plane, caches, vector stores, and the egress records for calls to public APIs. For each layer it tells you whether the canary is present, and hands back an evidence bundle with SHA-256 manifests you can give an auditor instead of a promise. It's read-only against everything it scans. Standard-library Python, no dependencies, because a tool whose job is to prove your data stayed home shouldn't drag a supply chain in to do it.

It's a sweep, not an assertion.. it checks the places data lands and inspects all of them..

The first thing it caught for me was the obvious one: a fallback to a frontier API under load, with customer data in the prompt. I'll be honest, I could have found that one other ways. It's a single egress path and it's easy to test on its own. If that were the whole tool, it wouldn't be worth writing about.

The leaks I didn't expect were the other ones. The prompt sitting in a trace that got forwarded to a hosted dashboard. The embedding in a vector store I'd provisioned without once asking where it physically lived. The semantic cache that had quietly become a system of record for every input. Those are the ones that matter, because those are the ones nobody tests, and they're exactly the ones a point check for "did we call a public API" will never surface.

The value isn't catching the fallback. It's catching the five places you forgot your data goes.

## The point

Residency is a surface. You can't prove a surface from a diagram, and you can't prove it by checking the one path you happened to be worried about. You prove it by running marked data through the real system and inspecting everywhere it could have landed, then keeping the evidence.

bordercheck does that. It's open source. Run it against your own stack and find out what your diagram left out, before someone less friendly does.

---

*Jeff Geiser writes about operator-model, sovereignty-first agentic infrastructure. He builds [Wicklee](https://wicklee.dev) and [bordercheck](https://github.com/jeffgeiser/bordercheck).*
