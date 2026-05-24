---
title: "Why I fine-tune small models instead of prompting big ones"
date: 2026-05-22
draft: false
---

About the ELM work: why bother?
You can prompt Claude or GPT-4 to generate an account brief. It works.
There are tools to connect.. it it does a good job of coupling together data.
Why spend weeks building a fine-tuning pipeline?

Three reasons.

**Cost at scale.** A frontier API call costs money.. every call. Not much in 
a single call, but it adds up. While there are costs to running a local model,
I believe that cost is a well justified investment and will quickly amortize..
 A locally-deployed 7B model costs the electricity to run it and does not require
much in the way of hardware/vram.. If we can progress small, expert models it
might not even need a gpu long term - maybe.. we will see.. 

**Latency and availability.** A local model responds in milliseconds with no
network dependency. No rate limits, no outages (hmm - maybe), no waiting. Maybe
this point is weak, but there is something here to tease out in terms of our 
ability to take a small model and deploy it broadly across the edge. Closer 
to end users.

**The narrow task advantage.** A frontier model is optimized to be good at
everything. A fine-tuned 7B is optimized to be excellent at one thing.
On the specific task of synthesizing enterprise account data into structured
JSON briefs across six defined surfaces, I expect the fine-tuned model to
outperform GPT-4 — not because it's smarter, but because it's been trained
on exactly this task, with exactly this schema, evaluated against exactly
these metrics.

**A fourth point** I simple believe that there will be a healthy balance of 
small, expert models and use of large frontier model APIs in the enterprise
and I think it is useful to explore these expert models. There is a non-zero
chance that a monolith model will be the way to go, but I believe (at the momeent)
that there is a greater chance that we will use scattered expert models, use
agents to route to the correct model and build a context strata across those 
requests to maintain understanding.

That's the thesis. If I am wrong I will learn something.  The methodology is open at
[elm-research](https://github.com/jeffgeiser/elm-research).
