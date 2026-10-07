---
title: "The Hard Part of Sovereign AI Isn't the Models"
date: 2026-10-06
draft: false
description: "Everyone's composing small specialized models now, and everyone's moving them onto hardware they own. Almost nobody is measuring what happens when those two collide."
tags: ["sovereign-ai", "compound-ai", "local-inference", "operator-model", "wicklee"]
---

**Everyone's composing small specialized models now, and everyone's moving them onto their own hardware. What happens when these two collide.**

*By Jeff Geiser — October 2026*

---

Two shifts are happening.

First, we stopped shipping a single model. If you're building anything serious now, you're composing. A small classifier to triage the request, a retrieval step, a generator, a verifier to catch the generator, etc.. Three to five specialized models wired into a pipeline, each doing their job. The research crew named it compound AI. A composed pipeline of small models beats one big model on accuracy and cost.. as long as you keep the coordination overhead in check.

Second, the models came home. The majority (or at least an increasing amount) of enterprise inference now runs on-prem and most organizations have pulled at least some workloads back from the cloud or are planning to. The reasons are because local inference is cheaper per token once you amortize the hardware, the box can pay for itself in weeks, and plenty of buyers legally can't put their data on someone else's machine anyway.

Both shifts are well covered on their own. Compound AI has its surveys. Sovereign AI has its think-pieces and NVIDIA's sovereign revenue line. But I wanted to consider what happens when you do both. 

## What the cloud was hiding

Run a composed pipeline against a hosted API and the hardest part is someone else's problem. You send a request, you get tokens back, you get a bill. the fact that a classifier and your generator landed on the same GPU and spent the request contending for VRAM is not something you see or care about. You never see that the node running your verifier was thermally throttled and quietly added a few hundred milliseconds to every call in the chain. You never see that the KV cache couldn't be reused between stages because they ran on different machines, so you paid to reprocess the same context three times. You never see per-stage cost at all, because the bill is one number and the provider swallows the rest.

Run that same pipeline on your own stack and every bit of that is yours now. Five composed models is five placement decisions (which node, given what's resident and how hot it is), five cost decisions (stay local or escalate to a frontier API, and what does each cost per token right now), and five health decisions (is this node degrading under load, and should I route around it). On heterogeneous metal, where a 4090, an A100, and a thermally-limited Mac Mini all behave differently under the same workload, you are making all fifteen of those decisions, per request, mostly blind.

The component models are not the hard part. They're commodity. Llama 3.2 3B is the default router. Judges and guardrails are also off the shelf. You can wire the pieces together in an afternoon. The hard part is making the assembled thing run well on infrastructure you own, under real load, and that is exactly the muscle a decade of cloud let everyone's org atrophy.

You can watch the field rediscover this in real time - most papers are about serving: SLO-aware query planners, deployment across heterogeneous clusters, keeping coordination overhead under the threshold, etc.. The community figured out how to build compound systems and is now finding out that running them efficiently on real hardware is a separate, unsolved problem.

There's an interesting few layers here.. :

- **See it.** Per node, per stage: thermal state, VRAM pressure, which model is resident, how efficient it is right now. Not raw metrics. A decision-ready read of whether this node is a good place to run this stage at this moment.
- **Cost it.** Real cost per token, per model, per stage, including the local-versus-frontier tradeoff, in watts and dollars, on your hardware. The thing the cloud bill flattened into one opaque line.
- **Route on it.** Given the above, where does each stage run, local or frontier, and what did the system learn from how that call actually turned out.

None of that is the model. 
The unit that matters here isn't tokens per second, it's tokens per watt. On someone else's hardware, efficiency is their problem. On yours it's the entire economic case. It's what you get if you run the hardware well, and roughly the inverse if you don't. Most of the orgs repatriating right now are going to learn which, the expensive way.. unless they are directly measuring it..

Wicklee, observability for sovereign ai, does the first two jobs, see it and cost it, on production fleets. The routing piece, the contract for how a stage asks a node "are you a good fit for this right now," is what I've been writing about as the decision loop. And the small expert models I build are components in exactly this kind of pipeline, built to run efficiently on owned hardware and emit a signal.

The valuable layer is the unglamorous operator layer underneath. The measurement, the cost, the routing..

## The short version

Compound AI won. The models are commodity, the frameworks are crowded, and the majority of inference is moving onto hardware the operator owns, where the cloud is no longer hiding the hard part.

The open problem is running compound AI efficiently on metal you own. That's not a model you can download or a framework you can fork. It's a layer...

---

*Jeff Geiser writes about operator-model, sovereignty-first agentic infrastructure. He builds [Wicklee](https://wicklee.dev).*
