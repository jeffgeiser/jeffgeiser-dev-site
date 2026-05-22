---
title: "Why I fine-tune small models instead of prompting big ones"
date: 2026-05-22
draft: false
---

About the ELM work: why bother?
You can prompt Claude or GPT-4 to generate an account brief. It works.
Why spend weeks building a fine-tuning pipeline?

Three reasons.

**Cost at scale.** A frontier API call costs money. Every call. When you're
generating account briefs for hundreds of accounts before a QBR cycle, that
adds up fast. A locally-deployed 7B model costs the electricity to run it.
At our inference setup that's roughly $0.00 per call.

**Latency and availability.** A local model responds in milliseconds with no
network dependency. No rate limits, no outages, no waiting.

**The narrow task advantage.** A frontier model is optimized to be good at
everything. A fine-tuned 7B is optimized to be excellent at one thing.
On the specific task of synthesizing enterprise account data into structured
JSON briefs across six defined surfaces, I expect the fine-tuned model to
outperform GPT-4 — not because it's smarter, but because it's been trained
on exactly this task, with exactly this schema, evaluated against exactly
these metrics.

That's the thesis. I'm testing it now. The methodology is open at
[elm-research](https://github.com/jeffgeiser/elm-research).
