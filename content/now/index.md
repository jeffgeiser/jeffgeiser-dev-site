---
title: "Now"
showpagemeta: false
---

*Updated May 2026 · [what is a now page?](https://nownownow.com/about)*

**ELM work** — finishing `account-intelligence-7b-v1`. Fine-tuned Qwen2.5-7B
on a synthetic dataset of 568 examples across six surfaces: meeting prep, QBR,
handoff, renewal alert, onboarding, escalation triage. LoRA training on DGX
Spark. Eval harness running DeepEval + LLM-as-judge with 8 locked metrics.
Q4_K_M GGUF release when it clears eval.

**Taarn** — a personal context and trust layer for AI agents. It keeps a
portable profile of my goals, decisions, and voice that any agent runtime can
read via MCP, and gates anything that touches the outside world through an
approval queue. The idea: swap runtimes freely, your context and trust rules
travel with you. Building in public at [taarn.ai](https://taarn.ai).

**Wicklee** — v0.9.0 just landed with Runtime Config Surface and a dedicated
Models tab. → [wicklee.dev](https://wicklee.dev) If you run Ollama, vLLM, or
llama.cpp in production and want to see WES (tokens-per-watt with thermal
honesty), I'd love feedback.

**compass-md** — an open format for portable AI context. A directory of
markdown files — your facts, voice, goals, decisions, sources — that any agent
can read. The idea is that your personal context shouldn't be locked inside any
one tool. README live at
[github.com/jeffgeiser/compass-md](https://github.com/jeffgeiser/compass-md).

**Running** — base building. No race on the calendar yet. Keeping it consistent.
