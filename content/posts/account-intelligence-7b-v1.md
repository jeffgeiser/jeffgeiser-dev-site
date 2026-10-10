---
title: "account-intelligence-7b-v1: A Small Model That Writes Account Briefs"
date: 2026-10-10
draft: true
toc: false
description: "I fine-tuned a 7B model to turn raw account data into structured JSON briefs.. it's up on Hugging Face, here's how it went and what I still haven't measured."
tags: ["elm", "fine-tuning", "small-models", "local-inference", "sovereign-ai"]
---

**I fine-tuned a 7B model to turn raw enterprise account data into structured briefs. It's on Hugging Face now, with the numbers and the gaps.**

*By Jeff Geiser — October 2026*

---

The model's up on Hugging Face now: [jgeiser/account-intelligence-7b-v1](https://huggingface.co/jgeiser/account-intelligence-7b-v1).

You hand it a bundle of account signals (support tickets, usage metrics, contract terms, stakeholder notes, etc..) and it gives you back a structured JSON brief across six surfaces: meeting prep, QBR pack, handoff, renewal alert, onboarding, and escalation context. Nobody is supposed to read the output like an essay.. it's JSON, and downstream systems consume it directly.

## Why fine-tune instead of just prompting

Prompting a frontier model works. I've done it. But it's ~$0.04 per brief at current API rates, it adds 8-12 seconds of latency per call, and your account data goes out to a third-party API every single time.

The fine-tuned 7B running locally has no per-call API fee, comes back in under 3 seconds on an A100, and the data stays on your infrastructure. You still pay for the hardware and the power, obviously.. but for a task with predictable inputs and outputs that runs hundreds of times a day, I think the math works out pretty clearly in favor of local.

This is what I've been calling the ELM (Expert Language Model) pattern: one small model, one task, structured output. I wrote more about the why in [Why I fine-tune small models instead of prompting big ones](/posts/why-i-fine-tune-small-models/).

## How I trained it

The setup:

- **Base model:** Qwen2.5-7B-Instruct
- **Method:** LoRA (rank 16) via Unsloth + TRL SFTTrainer
- **Data:** 1,127 (source bundle → structured brief) pairs
- **Sequence length:** 16,384 tokens
- **Training:** 10 rounds on Lambda A6000/A100 instances (~$40 total compute)

The training pairs are source bundle in, brief out.. there's no prompt-to-prose in there. I used completion-only loss masking, so the model only trains on the JSON output tokens and never on the input. The idea is it has to learn to actually pull the bundle together into a brief, instead of getting good at repeating what it was handed.

The thing that bit me early was structure. With no output constraints, schema adherence on the early rounds was ~3%. The JSON would drift, required enums would get hallucinated, arrays would overflow.. pretty much a mess.

What fixed it was constrained decoding: [Outlines](https://github.com/dottxt-ai/outlines) for fp16 inference, and llama.cpp's JSON schema grammar for the GGUF. Both build an FSM from the JSON schema and mask the logits at each step, so the model literally can't emit a token that breaks the schema. M1 (schema adherence) went to 100% and stayed there.

## Results

I evaluated on 60 held-out examples.

| Model | M1 Schema | M3 Coverage | M5 Attribution |
|---|---|---|---|
| Base Qwen2.5-7B (untuned) | 100% | 28.1% | 69.2% |
| Round 10 adapter (fp16) | 100% | 95.8% | 92.7% |
| Round 10 Q4_K_M GGUF | 100% | 99.2% | 96.1% |

M1 is 100% for every row because constrained decoding forces it, so treat it as a floor and don't read much into it. The actual fine-tuning gain shows up in M3 and M5.

**M3 (section coverage).** The base model knows the shape of the schema, but it writes skeletal 2-3KB briefs. The fine-tuned model writes complete 14-16KB briefs. 28% → 99%.

**M5 (source attribution).** Of the sources in the input bundle, how many get correctly cited in the output. 69% → 96%.

One thing I noticed: the GGUF slightly beats the fp16 adapter on both. Q4_K_M quantization sometimes regularizes outputs at the edges, so maybe that's it.. or it's just eval variance across 60 examples. I'm not claiming it as a finding.

What I haven't measured yet is M2, hallucination rate.. whether the model invents plausible-sounding content the source bundle doesn't support. Honestly that's the metric that matters most, and it's not in this release. Measuring it needs an LLM judge pass over the outputs, and that's on the roadmap.

If you want to try it, there are two files:

- `adapter_model.safetensors`: the LoRA adapter, ~100MB, needs a GPU + Unsloth
- `account-intelligence-7b-v1-Q4_K_M.gguf`: 4.68GB, runs on CPU or GPU, Apple Silicon via Metal

Both are at [huggingface.co/jgeiser/account-intelligence-7b-v1](https://huggingface.co/jgeiser/account-intelligence-7b-v1). If you run it against your own data I'd love to hear how it does..

---

*Jeff Geiser writes about operator-model, sovereignty-first agentic infrastructure. He builds [Wicklee](https://wicklee.dev).*
