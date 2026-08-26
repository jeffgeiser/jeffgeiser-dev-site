---
title: "The Layer Below MCP"
date: 2026-08-26
draft: false
description: "Connection isn't the hard part anymore. Reasoning about resources is — the resource-decision gap the current protocol stack leaves wide open."
tags: ["inference-infrastructure", "decision-loop", "agentic-infrastructure"]
---

**Connection isn't the hard part anymore. Reasoning about resources is.**

*By Jeff Geiser — April 2026*

---

Today, most agents run on a single machine or call a public API. Routing is trivial — there's one option. The question "which GPU should this go to" doesn't come up because there's nothing to pick between.

That changes the moment an agent has a fleet under it. One GPU is cool and idle, with plenty of free VRAM. Another is thermally throttled, busy, near OOM. The question of where to send the work becomes real but the agent isn't the one answering it. A layer underneath is.

NVIDIA Dynamo. vLLM's scheduler. Ray Serve. A humble load balancer in front of a handful of inference endpoints. These systems route workloads based on what they can see: queue depth, latency, available memory, GPU utilization. They route well, for what they know. They are good at their job.

What they don't see is what the agent is trying to do. Whether a 200ms slower answer is fine or fatal. Whether it's the first step in a chain that will reuse the same KV cache six more times, or a one-shot call that will never touch this model again. Whether the user is waiting or it's background work. Whether the operator has a cost envelope for the whole task, or only for this individual call. The agent has all of that context. The routing layer has none of it.

The agent isn't picking the wrong GPU. It isn't picking at all. It's been left out of a decision it has context for, because the protocol stack doesn't have a slot for it to participate.

This is the gap I want to talk about, because it's the gap everyone in agentic infrastructure is starting to feel..

---

## Three protocols

The agentic infrastructure stack, as of mid 2026, looks roughly like this:

```
┌──────────────────────────────────────────┐
│  Agent Application (Claude, GPT, custom) │
├──────────────────────────────────────────┤
│  MCP — tool connection                   │
│  A2A — agent-to-agent discovery          │
│  ACP — commerce / payments               │
├──────────────────────────────────────────┤
│  ???  — resource execution layer         │  ← missing
├──────────────────────────────────────────┤
│  Hardware (GPU, CPU, memory, thermals)   │
└──────────────────────────────────────────┘
```

MCP, from Anthropic, lets agents connect to tools and data sources. It standardizes the "how do I call this thing" layer. It's the part of the stack that makes Claude (or any other LLM) able to actually reach into the world.

A2A, from Google, lets agents discover and delegate to other agents. It's about federation.. how do I find another agent and ask it to do something on my behalf.

ACP, from OpenAI and Stripe, handles commerce. Catalogs, checkout, payment tokens. It's the protocol for when an agent has to buy something.

Each of these is genuinely useful. None of them addresses the question I started this essay with. MCP gets the agent connected to the inference API. It does not tell the agent which inference API. A2A lets the agent find other agents that might have GPUs. It does not tell the agent whether those GPUs are thermally healthy. ACP can settle a payment for compute. It cannot answer "should I run this 70B model on this specific node right now."

**MCP, A2A, and ACP get agents to the door. None of them tells the agent what to do once it walks through.**

---

## What does the gap actually look like

Here are four things an agent in 2026 cannot do, even with the entire current protocol stack at its disposal:

**1. Ask a node whether it's thermally healthy before sending it work.** The MCP tool surface for inference is "call this endpoint." The endpoint is opaque. If the node behind the endpoint is throttled, the agent finds out by experiencing slow tokens — not by querying the state.

**2. Declare intent before consuming resources.** Every infrastructure system that has scaled has needed some form of intent declaration: cgroups, Kubernetes resource requests, AWS reservations, SQL transaction scopes. Agentic infrastructure has none of this. Agents just start jobs. The system finds out what the agent wanted by watching what it did. This makes oversubscription prevention impossible and makes "why did this fail" forensics expensive.

**3. Compare two equivalent endpoints on dimensions other than success rate.** Two GPUs both run 70B models. One is reliably 9.1 tokens-per-watt at 41°C. The other is variably 6–9 tokens-per-watt at 78°C with thermal swings. The agent has no protocol-level way to know this. It just knows that sometimes calls succeed and sometimes they don't.

**4. Reason about predictability as a first-class property.** The most expensive thing in serious agentic systems isn't compute time — it's variance. An agent that runs a 60-second job in 60 seconds is cheap. An agent that runs a 60-second job in 30–300 seconds, depending on conditions it cannot see, is operationally untenable. There is no protocol for predictability.

These are not exotic edge cases. They are the everyday reality of running agents on real GPUs. The protocol layer that was supposed to make agents first-class infrastructure citizens treats them as if they're calling a black-box API in someone else's data center.. even when the data center belongs to the same operator running the agent.

---

## I came at this sideways

The way I noticed this gap was not by sitting down to design a protocol. I was building [Wicklee](https://wicklee.dev) — an open-source fleet monitor for GPU inference. Rust binary on each node, installs in 30 seconds, dashboard at `localhost:7700` that shows you what the hardware is doing. The natural next step after "monitor your fleet" is "aggregate the data and show you a unified view." So I built that.

Then I sat there looking at the dashboard and realized I was building for the wrong consumer.

The dashboard exists because a human needs to look at fleet state and decide what to do with it. But in an agentic system, the consumer of fleet state is not a human. It's an LLM making routing decisions. And LLMs don't read dashboards. They read structured payloads with status assessments and suggested actions. The thing the agent needed from me wasn't a better chart. It was a contract — a protocol for handing fleet state over in a form the agent could actually act on.

That shift — from "the customer of inference observability is a human reading a dashboard" to "the customer is an LLM making tool calls" — is the moment I stopped adding panels and started writing a spec. Once you see the consumer change, every design decision in the observability layer changes with it. You don't need pretty charts. You need machine-readable status fields with confidence scores and recommended actions. You don't need historical graphs. You need a structured query for "is this node a good fit for this workload right now, and if not, why not."

Most of the conversation about agentic infrastructure is happening at the model layer and the tooling layer. The layer that needs the most rethinking, the one that no one is rethinking, is the one underneath both: how do agents reason about the physical compute they're running on.

---

## The marketplace assumption is the wrong starting point

Most thinking on "how do agents pick GPUs" has been framed as a marketplace problem. The future, the story goes, is agents shopping a market of GPU time. Akash, Render, io.net, Vast.ai, and a long tail of decentralized compute marketplaces are all betting on this thesis. So is most of the venture capital flowing into agentic compute right now.

I think the marketplace framing is a distraction. Not because it's wrong (there will be marketplaces) but because it puts the interesting case in the wrong place.

The interesting case isn't agents shopping a market they don't control. It's agents reasoning about infrastructure they already own.

Call this the **operator model**. The operator owns the hardware. The agents run locally. The data stays sovereign. The trust model is internal, not external. The feedback loop is fast because every agent run produces telemetry the operator can see immediately. There's no marketplace between the agent and the metal.

The operator model is more defensible than the marketplace model in three specific ways.

**Sovereignty.** Many serious users of agentic infrastructure — defense, healthcare, finance, regulated enterprise — cannot put their data on a third-party marketplace. They are going to run agents on hardware they own. They need a protocol for that. The marketplace plays cannot serve them at all.

**Locality.** The feedback loop in a marketplace is slow because the marketplace doesn't see what the agent did with the compute. It only sees the transaction. In an operator-owned fleet, every agent run produces telemetry that immediately feeds back into routing decisions. The flywheel runs faster because the loop is shorter.

**Speed of iteration.** An operator can change how their fleet decides things in an afternoon. A marketplace has to coordinate change across thousands of independent providers. The protocol layer for operator-owned infrastructure can move faster, get smarter faster, and converge on real working semantics faster than any marketplace standard.

The marketplace framing is the cloud-era assumption applied to a post-cloud problem. Most agentic compute is going to live behind an operator's existing infrastructure boundary, not in a market. The protocol that wins is the one designed for that case.

---

## ARP: four phases, one flywheel

I've been writing a specification for this layer. It's called the Agentic Resource Protocol — ARP — and it lives at [github.com/jeffgeiser/arp-spec](https://github.com/jeffgeiser/arp-spec) under MIT license. The current version is 0.1.0. It's a draft. It's probably wrong in interesting ways.

ARP defines four phases. Each one is independently useful. Each one makes the previous phases more valuable. The whole thing is designed so an implementor can ship Sense without ever touching Reconcile and still get value.

### Sense — context, not metrics

The first phase is Sense. The agent calls `get_fleet_context()` and gets back a structured description of the fleet: nodes, their states, their thermal conditions, their inference state, their efficiency scores. The key design decision is that **every numeric field comes with a status assessment.** Not just `wes: 11.8`, but `wes: 11.8, wes_status: "good", confidence: 0.84`. The agent should not have to reason about raw numbers. It should reason about pre-digested decisions.

If a node is in a degraded state, Sense includes a `suggested_action` field. If thermal headroom is collapsing, Sense suggests reducing batch size. If a node is offline, Sense suggests investigation. The protocol is opinionated. It treats the agent as the customer of the data, and customers need decision-ready output, not raw telemetry.

You can think of Sense as DNS for your own infrastructure — a standardized way to ask "what's out there and what shape is it in" that returns something you can act on without parsing.

### Score — workload-specific prediction with explanation

The second phase is Score. The agent describes a workload (a model family, a parameter count, a quantization, a context length) and a specific node, and Score returns a 0.0–1.0 fit scalar with a human-readable explanation.

The agent doesn't get back `0.73`. It gets back: *"0.73. Primary constraint: thermal headroom (78°C, 12°C from throttle threshold). Secondary: VRAM margin tight at this context length. Recommendation: try the cooler node with more thermal margin."* Plus a per-factor breakdown showing exactly which dimensions contributed and how.

This is non-negotiable for a protocol designed for agents. An agent that can explain its routing decision to a human operator is dramatically more trustworthy than one that just emits a number. The explainability requirement is in the spec as a MUST, not a SHOULD.

The v0.1 Score formula is a weighted combination of thermal headroom (30%), VRAM fit (30%), historical efficiency on this node for this model class (25%), and recent reliability (15%). The weights are starting heuristics — the per-factor formulas are documented in the spec so two implementations produce comparable scores. A learned model will replace them once Reconcile data accumulates. The point of v0.1 is not to be the right Score formula. The point is to define the contract precisely enough that two implementations can disagree about the math while agreeing about the interface.

### Commit — intent declaration with credit escrow

The third phase is Commit. Before an agent uses a resource, it declares what it expects to use, for how long, and how many credits it stakes on that prediction. The system validates the declaration, holds the credits in escrow, and returns a reservation ID.

This is uncomfortable. It adds latency. It adds complexity. Implementors will resist it. I require it because of three things you cannot get any other way:

**The declared-vs-actual delta is the training signal for Score.** Without a declaration, you can measure what happened, but you cannot measure how well the prediction worked. The intent log is the ground truth that lets the protocol learn.

**Oversubscription prevention requires knowing what's coming, not just what's running.** Two agents that both want to load 70B models on the same 32GB node need to find out about the conflict at Commit time, not at OOM time.

**Agent reputation requires a baseline.** An agent that consistently declares accurately is worth more than an agent that wildly over-declares to be safe. There is no way to compute this without intent.

The credit model is the part that gets the most pushback, so I want to be explicit. **Credits in ARP are not money.** They are a coordination primitive — a way to enforce a resource budget across the fleet without centralized scheduling. The fleet owner sets the pool. Agents draw from it. The credit price encodes predictability (node reliability × agent reputation × predicted efficiency × thermal headroom), not compute time.

You are pricing the cost of producing a predictable outcome. Predictability has a real cost, and now it has a real price.

### Reconcile — the three artifacts and the flywheel

The fourth phase is Reconcile. Every completed job produces three artifacts:

**ActualResourceReceipt** — what was promised vs. what was delivered. The financial settlement, in the operator's local currency of credits.

**AgentAccuracyDelta** — how well the agent predicted its own resource consumption. Duration predicted vs. actual. VRAM predicted vs. measured. This feeds the agent's reputation.

**NodeReliabilityDelta** — whether the node honored its commitment. Predicted efficiency vs. actual. Thermal stability during the run. This feeds Score directly. Unreliable nodes get lower Score predictions for future workloads.

These three artifacts are the most important objects in the entire spec. They are the training data. Every reconciled job makes Score's predictions more accurate. Every cycle makes the next routing decision better. This compounds without any new features shipping.

The settlement rules are deliberately asymmetric. When a node under-delivers, the agent gets a pro-rated refund. When an agent over-consumes, the surcharge is bounded — capped, not punitive. Over-penalizing trains agents to over-declare defensively, which corrupts the intent log. The system should incentivize honest declaration, not conservative declaration. This sounds like a small detail. It is the load-bearing detail.

---

## Wait — isn't this too much for an agent to care about?

The reasonable objection at this point: you're asking a lot of an agent. Four phases, credit escrow, declared intent, reputation tracking, per-factor score explanations — that's a meaningful protocol surface. The entire history of good infrastructure design is the consumer not having to think about the layer below. Developers don't pick nodes in Kubernetes. Web apps don't pick DNS servers. Browsers don't pick TCP paths. Every abstraction that got adopted did so because it hid complexity, not exposed it. A code-generation agent should not have to know what a thermal penalty is. And this is exactly the role Dynamo and Ray Serve and vLLM are playing today.. they absorb routing so the caller doesn't have to.

The objection is correct. The answer is that **ARP's consumer is not the primary agent doing domain work.** It's a specialized routing sub-agent the primary agent delegates to.

The primary agent — the one writing code, doing research, answering the user, orchestrating the long-running task — stays simple. It says "I need to run inference on this workload, here's my goal, here's my constraint envelope" and hands off to a sub-agent whose entire job is reasoning about compute. The sub-agent speaks ARP. The primary agent never sees a credit escrow or a per-factor score breakdown. It sees an answer: "run it on node B, here's the reservation, here's why."

This is the same pattern every mature infrastructure layer eventually produces. DNS is not consumed by every application directly — it's consumed by a resolver that applications call through `getaddrinfo`. You don't complicate your web app with DNS logic. You call a library. ARP is designed to be consumed the same way: not by the primary agent, but by a specialized routing agent acting on its behalf.

Once you adopt that framing, the stack clicks into a cleaner shape:

| Layer | What it does | Protocol |
|---|---|---|
| Primary agent | Domain work — code, research, answering the user | MCP (for tool calls) |
| Routing sub-agent | "Where should this inference run, and under what terms?" | **ARP** |
| Inference backend | Actually runs the model | Dynamo / vLLM / Ray Serve / custom |
| Network orchestration | Shapes bandwidth, path selection, QoS for LLM traffic | (active area; closed-loop systems emerging) |
| Hardware | GPUs, thermals, memory | Platform telemetry |

Each layer does one thing well. Each consumes the layer below through a contract. The primary agent stays simple because the routing sub-agent absorbs the complexity. The routing sub-agent stays effective because ARP gives it a structured contract to reason about. The inference backend stays focused on running the model fast. Nobody is doing anyone else's job.

The network-orchestration row deserves a brief note. Distributed inference traffic is unusually spiky — gradient sync bursts during training, KV cache streaming during inference, bursty token generation under load — and the systems that shape bandwidth, queue priority, and path selection for those workloads are their own active problem space, distinct from anything the agent layer touches. Closed-loop orchestrators that observe LLM performance metrics and dynamically adjust the underlying network are starting to appear, largely from the operators who run the hardware and feel the cost of generic QoS most acutely. ARP is intentionally silent on that layer. It's complementary, not subordinate, and the contract for talking to it cleanly is future work — the kind of thing that makes sense to define once both the agent layer and the network layer have stabilized enough to know what a clean handshake between them should look like.

This also resolves the obvious competitive question: "why not just let Dynamo handle the routing?" Dynamo is a very good closed-loop router. It does exactly what closed-loop routers should do. But a closed-loop router cannot know things that only exist at the agent layer — what the primary task is ultimately trying to accomplish, what the acceptable fallback options are, whether this is step 6 of a 10-step chain that should lock the same node for KV cache reuse, whether the user is waiting or this is background work, what the cost envelope is for the whole task rather than the individual call. Those are not Dynamo's problem to solve. They're the sub-agent's problem. ARP is the protocol that lets the sub-agent solve them without re-implementing Dynamo from scratch.

In fact, the cleanest version of the stack is one where Dynamo *implements ARP Sense* — exposing its internal fleet state through `get_fleet_context()` so a routing sub-agent can query it alongside other inference backends. Dynamo doesn't have to give up routing control. It just has to expose what it knows in a structured form. The sub-agent takes that structured data, combines it with what the primary agent is trying to do, and makes the decision at the right layer. That's not competition. That's a stack.

There's a second-order benefit to the sub-agent framing that I didn't appreciate at first: **the routing sub-agent becomes portable across primary agents.** You build one good sub-agent that knows your fleet, and every primary agent you spin up inherits the routing intelligence by delegating to it. The code-generation primary agent and the customer-research primary agent both delegate to the same routing sub-agent. The reputation that sub-agent earns over time — the historical record of good decisions, accurate predictions, clean settlements — is itself an operational asset that compounds. It gets smarter with every job, independent of whatever primary task is running above it.

The objection "don't complicate the agent" is correct. The answer is not "simplify ARP until any agent can consume it directly." The answer is "the consumer of ARP is a specialist, and ARP's existence is what makes that specialization possible in the first place."

---

## Reconcile is the moat

If you read the spec and ask "where's the actual differentiation," the answer is not the four phases. The four phases are the marketing. The differentiation is the portable Reconcile artifacts.

ARP is the first protocol I'm aware of where **hardware can rate the agents running on it.** Every other infrastructure protocol I've worked with has been one-directional: humans rate hardware (uptime, latency, throughput). ARP is bidirectional. The hardware learns which agents are predictable, which agents crash, which agents over-declare. That data lives in the protocol, not in any individual implementation. It's portable across ARP-conformant fleets.

Why does that matter? Two reasons.

First, agent reputation is the actuarially correct way to price unpredictable behavior. An agent that consistently over-declares VRAM is a real cost to the fleet — you can't pack other workloads on the same node when you don't know what's actually being used. Today that cost is invisible. With ARP, it's a credit-price multiplier. You don't punish the agent — you accurately price the unpredictability so the operator can decide what to do about it.

Second, agent reputation is portable. An agent that earns a good reputation on one ARP fleet can carry it to another. (This isn't in v0.1 — multi-fleet portability is planned for v1.0 — but the artifact format is designed to support it.) That portability is what turns ARP from a single-vendor protocol into a real standard. The day an agent can show up at a new fleet with a verifiable history of predictable behavior is the day this stops being a spec and starts being infrastructure.

The flywheel is simple to draw and harder to internalize:

```
More Commits
   → Richer Reconcile data
       → Better Score predictions
           → Agents trust the protocol more
               → More Commits
```

The product gets better without any new features shipping after Reconcile exists. Every reconciled job improves Score accuracy. This runs automatically. It's the only part of the protocol that compounds.

---

## Why this matters now and not in 2027

Three things are converging to make this the right moment to define the resource layer.

**Agents are starting to do real work on real GPUs.** Until about six months ago, most "agents on GPUs" content was demos and toy loops. That has shifted. There are now serious systems running long-horizon agentic workloads on operator-owned hardware. The volume is small but the rate of change is fast. The people building these systems are starting to feel the gap I described at the top of this essay. They are going to build something to fill it. The question is whether they each build their own private version or whether they coordinate around a standard.

**The economics matter.** GPU time is expensive enough that getting the routing decision wrong costs real money. This was not true two years ago when GPUs were primarily used for model training, where getting the routing wrong just meant the job took longer. With inference workloads driving most GPU spend, the cost of variance is now visible in the bill.

**The MCP wave is finally building the bottom of the stack.** MCP adoption has gone from "interesting idea" to "table stakes" in about a year. You can now assume the connection layer exists. That assumption was not safe in 2024. It is safe now. Which means the next layer up — the resource layer — is the next natural conversation.

The window to define the resource layer is open right now. It will close in 12 to 18 months when one of the major players ships their version. NVIDIA could ship a protocol on top of Dynamo. Anthropic could extend MCP with resource hints. Cloudflare could ship an agent-facing surface for Workers AI. Any of these would become the de facto standard immediately because of the distribution behind it.

The opportunity for an open spec is to be the thing those players adopt rather than the thing they replace. That is the MCP playbook: ship the open spec early enough that the closed alternatives don't get traction. It's worked before. It's worth trying again.

---

## Who should be paying attention

The strongest signal you should keep reading is that you're in one of these camps.

**CDN and network operators putting inference at the edge.** You already run distributed infrastructure across heterogeneous hardware at hundreds or thousands of locations. Your routing logic today is driven by latency, geography, and capacity. The next iteration of that logic has to account for per-node workload fit — thermal state, model residency, KV cache warmth, agent reputation — and there is no protocol for it yet. Adding a structured resource layer on top of what you already run is a natural extension, not a rewrite.

**Teams building distributed inference across heterogeneous hardware.** If you've ever written a routing heuristic that tried to account for mixed GPU generations, driver versions, thermal profiles, or differing quantization support across nodes, you already know why Score needs to exist. You've built a worse version of it by hand. ARP gives you the contract so you stop re-inventing it per project and start comparing notes with everyone else doing the same work.

**Operators running on-prem GPU fleets who don't want a vendor owning routing.** The alternative to an open resource protocol isn't "no resource protocol." It's a resource protocol shipped by whichever hyperscaler or model vendor gets there first. If you care about not having your fleet's routing decisions dictated by someone else's closed loop — especially when that loop has incentives you don't share — now is the time to care about an open spec.

**Anyone designing agentic systems that will outgrow a single endpoint.** Today you're calling one inference API. Tomorrow you'll have four, and then forty. The routing sub-agent pattern described above is what lets you scale from one endpoint to forty without the primary agent's code touching any of it. ARP is what the sub-agent uses to reason. If that scaling curve is in your next 12 months, the protocol decision is in your next three.

If none of these are you, ARP is probably not urgent for you yet. Come back in a year.

---

## Where this goes

I set out to write a spec. I care less about the spec now than the shape underneath it.

Sense, Score, Commit, Reconcile isn't really a protocol you adopt. It's a loop. Look at the state, predict the fit, claim the resource, learn from what actually happened. Name it once and you start seeing it everywhere. An agent picking a GPU runs it. vLLM's scheduler runs a dumber version of it. And the network orchestration row from that table, the one I said to hold.. a network deciding where inference runs and over which path is running the same loop, one scale up, with the wire folded into the state.

That's the thread I want to pull next, and honestly it's the one that matters more. The GPU version you can build and measure today. The network version nobody's really built yet, and I think it's worth more.

So I'm not going to ask you to help me bless a standard. I asked for that in the first LinkedIn post and I've come around. The value was never the committee. It was noticing the loop is real, and being the one who measures it.

If you run a fleet, even one node, here's the cheap test. Look at your own state the way a router would have to, and count how much of the placement decision you're currently making blind. That number is most of the argument.

More on the network layer soon.

---

*Jeff Geiser writes about operator-model, sovereignty-first agentic infrastructure. He builds [Wicklee](https://wicklee.dev).*
