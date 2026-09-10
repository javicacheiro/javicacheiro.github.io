---
title: 'Throughput is not a fair fight'
description: 'Seven large models on one 8-GPU node. The fastest is nine times smaller than the slowest, which changes what the ranking means.'
publishDate: 2026-09-05
draft: true
tags:
  - ai
  - llm
  - gpu
  - benchmarking
  - inference
---

<style>
.qz {
  --ok: #0e9384;
  --ok2: #9dd3cb;
  --bad: #c2410c;
  --ref: hsl(var(--muted-foreground));
  --grid: hsl(var(--border));
  --ink: hsl(var(--foreground));
  --dim: hsl(var(--muted-foreground));
  margin: 2rem 0;
}
.dark .qz { --ok: #2f9e8f; --ok2: #2a5f5a; --bad: #cf6238; }
.qz svg { width: 100%; height: auto; display: block; overflow: visible; }
.qz figcaption { font-size: .875rem; line-height: 1.55; color: var(--dim); margin-top: .8rem; }
.qz .lab { font-size: 12px; fill: var(--ink); }
.qz .sub { font-size: 10.5px; fill: var(--dim); }
.qz .val { font-size: 12px; font-weight: 600; fill: var(--ink); }
.qz .ax  { font-size: 11px; fill: var(--dim); }
</style>

I benchmarked seven large models on a single 8×B300 node — 414 runs, seven
workload shapes, concurrency swept from 1 to 1024 — and the headline was easy to
write: **DeepSeek-V4-Flash wins every workload, by a lot.**

It is also close to meaningless on its own, because DeepSeek-V4-Flash is
**nine times smaller than Kimi-K3**. Ranking them on tokens per second is like
ranking vehicles by top speed without mentioning that one is a motorbike.

So I have taken it out. Every chart in this post compares the five models that
are genuinely in the same class — 714 GiB to 1,532 GiB of weights on the node —
and Flash gets its own section at the end, on its own axis, where a 170 GiB
model belongs. The five-model comparison is the interesting one; the Flash
section is the fun one.

## How big these models actually are

Parameter counts are awkward for this comparison: several of these are
mixture-of-experts, published counts mix total and active parameters, and two of
the checkpoints are 4-bit. So instead of a number from a model card, I use one I
measured — **what vLLM reported loading**, summed across the eight GPUs:

<figure class="qz">
  <svg viewBox="0 0 640 201" role="img" aria-label="Weight footprint. GLM-5.2 FP8 714 GiB, DeepSeek-V4-Pro 850 GiB, Qwen3.8-2.4T 1,340 GiB, GLM-5.2 bf16 1,412 GiB, Kimi-K3 1,532 GiB.">
    <line x1="279" y1="22" x2="279" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="279" y="18" text-anchor="middle">500 GiB</text>
    <line x1="402" y1="22" x2="402" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="402" y="18" text-anchor="middle">1,000 GiB</text>
    <line x1="525" y1="22" x2="525" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="525" y="18" text-anchor="middle">1,500 GiB</text>
    <text class="ax" x="156" y="195" text-anchor="start">Weight footprint — measured across the 8-GPU node, precision as served</text>
    <text class="lab" x="148" y="45" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="56" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="34" width="176" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="340" y="46">714 GiB</text>
    <text class="lab" x="148" y="72" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="83" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="61" width="209" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="373" y="73">850 GiB</text>
    <text class="lab" x="148" y="99" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="110" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="88" width="330" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="494" y="100">1,340 GiB</text>
    <text class="lab" x="148" y="126" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="137" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="115" width="348" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="512" y="127">1,412 GiB</text>
    <text class="lab" x="148" y="153" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="164" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="142" width="377" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="541" y="154">1,532 GiB</text>
  </svg>
  <figcaption>
    From vLLM's own <code>Model loading took N GiB</code> line per rank, times
    eight ranks. This is the honest size axis for a serving comparison: it
    already accounts for quantization, and it is what determines how much KV
    cache fits alongside the weights.
  </figcaption>
</figure>

A useful sanity check falls out of this. GLM-5.2 appears twice — bf16 at
1,412 GiB and FP8 at 714 GiB, almost exactly half. Same model, half the bits,
half the footprint. That the arithmetic works is a small confirmation that the
measurement means what I think it does.

## The seven workloads

Every number in this post comes from one of seven synthetic workload shapes.
They differ only in how many tokens go in and how many come out, but that is
enough to change the answer completely — a model that leads on one can be
mid-table on another, and the same hardware produces a 24× spread in tokens per
second depending purely on which shape you run.

| workload | in → out | what it stands for |
|---|--:|---|
| **API-S** | 2,048 → 256 | Short structured request: extraction, routing, JSON generation, metadata tagging. High request rate, no reasoning. |
| **CHAT-S** | 2,048 → 512 | Ordinary interactive chat — short assistant turns. The latency-sensitive default. |
| **CODE-I** | 16,384 → 2,048 | Interactive coding assistant: explain this, fix this, review this. Bursty, a few files of context. |
| **CHAT-L** | 32,768 → 2,048 | A long technical conversation with accumulated history and detailed answers. |
| **CODE-A** | 65,536 → 4,096 | A coding-agent call with a repository in context — one model call, not the full tool loop. |
| **DOC-L** | 131,072 → 2,048 | Long-document analysis and RAG: reports, logs, retrieved context. Prefill-dominated. |
| **BATCH-D** | 4,096 → 8,192 | Decode-heavy batch generation, EOS suppressed. Approximates maximum sustained output capacity. |

Two of these are deliberately artificial. **BATCH-D** forces `ignore_eos` so the
model cannot stop early — it is a capacity probe, not a workload anyone runs,
and it is the one that produces the biggest tok/s numbers in every vendor
benchmark you will ever read. **API-S** and BATCH-D run with reasoning disabled;
the other five leave it on, which is why their output token counts are
realistic rather than minimal.

Everything else in this post is `/v1/chat/completions` against synthetic text at
the stated shape, swept across concurrency and taking each model's best run.

## The raw ranking

<figure class="qz">
  <svg viewBox="0 0 640 201" role="img" aria-label="Peak batch throughput. GLM-5.2 FP8 7,204, DeepSeek-V4-Pro 9,183, Qwen3.8-2.4T 6,168, GLM-5.2 bf16 4,360, Kimi-K3 4,422.">
    <line x1="259" y1="22" x2="259" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="259" y="18" text-anchor="middle">2,500</text>
    <line x1="361" y1="22" x2="361" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="361" y="18" text-anchor="middle">5,000</text>
    <line x1="464" y1="22" x2="464" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="464" y="18" text-anchor="middle">7,500</text>
    <text class="ax" x="156" y="195" text-anchor="start">Peak batch throughput — BATCH-D, best concurrency per model</text>
    <text class="lab" x="148" y="45" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="56" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="34" width="296" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="460" y="46">7,204</text>
    <text class="lab" x="148" y="72" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="83" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="61" width="377" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="541" y="73">9,183</text>
    <text class="lab" x="148" y="99" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="110" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="88" width="253" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="417" y="100">6,168</text>
    <text class="lab" x="148" y="126" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="137" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="115" width="179" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="343" y="127">4,360</text>
    <text class="lab" x="148" y="153" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="164" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="142" width="182" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="346" y="154">4,422</text>
  </svg>
  <figcaption>
    Peak sustained output on BATCH-D, each model at whichever concurrency
    maximised it. Headline metric is <code>server_output_tps</code> as reported
    by vLLM — what an operator sees on their own dashboard.
  </figcaption>
</figure>

**DeepSeek-V4-Pro leads at 9,183 tok/s**, 27% clear of GLM-5.2 FP8 — and it is
19% *larger*. That is the shape of a result worth having: the winner is not the
smallest model in the chart, so the lead is not a size effect.

At the other end, Kimi-K3 and GLM-5.2 bf16 finish within 1.4% of each other,
which is inside run-to-run noise. The two largest checkpoints in the set are
tied.

## Normalising by size

<figure class="qz">
  <svg viewBox="0 0 640 201" role="img" aria-label="Throughput per GiB of weights. GLM-5.2 FP8 10.1, DeepSeek-V4-Pro 10.8, Qwen3.8-2.4T 4.6, GLM-5.2 bf16 3.1, Kimi-K3 2.9.">
    <line x1="331" y1="22" x2="331" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="331" y="18" text-anchor="middle">5.0</text>
    <line x1="505" y1="22" x2="505" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="505" y="18" text-anchor="middle">10.0</text>
    <text class="ax" x="156" y="195" text-anchor="start">Throughput per GiB of weights — peak BATCH-D tok/s ÷ node weight footprint</text>
    <text class="lab" x="148" y="45" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="56" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="34" width="352" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="516" y="46">10.1</text>
    <text class="lab" x="148" y="72" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="83" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="61" width="377" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="541" y="73">10.8</text>
    <text class="lab" x="148" y="99" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="110" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="88" width="161" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="325" y="100">4.6</text>
    <text class="lab" x="148" y="126" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="137" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="115" width="108" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="272" y="127">3.1</text>
    <text class="lab" x="148" y="153" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="164" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="142" width="101" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="265" y="154">2.9</text>
  </svg>
  <figcaption>
    Peak BATCH-D throughput divided by node weight footprint. A crude
    normalisation — it assumes throughput should scale inversely with model
    size, which is only roughly true — but it separates "fast because small"
    from "fast because well-built".
  </figcaption>
</figure>

If throughput were purely a size effect, this chart would rank the five models
in reverse order of footprint. It does not. **GLM-5.2 FP8 is the smallest of the
five and finishes second**, behind DeepSeek-V4-Pro, which carries 19% more
weight and still extracts more throughput from every GiB of it. Whatever Pro is
doing well, it is not being small.

The gap between the top two (10.8 and 10.1) and the bottom three (4.6, 3.1, 2.9)
is much larger than the gap within either group, and it does not track precision:
the group of two is FP8, the group of three is NVFP4, bf16 and MXFP4. Footprint
and format together explain the split better than either does alone.

I would not push this metric far. Dividing by size assumes a linear relationship
that does not really hold, and it flatters small models. It is a lens, not a
verdict — which is the other reason Flash is not on it.

## The fair fight: comparing within size tiers

The most useful comparison is between models someone would actually be choosing
between — ones of comparable size, competing for the same GPUs.

| tier | model | footprint | BATCH-D | CHAT-S |
|---|---|--:|--:|--:|
| **mid** | **DeepSeek-V4-Pro** | 850 GiB | **9,183** | **5,493** |
| | GLM-5.2 FP8 | 714 GiB | 7,204 | 3,989 |
| **large** | **Qwen3.8-2.4T** | 1,340 GiB | **6,168** | **4,115** |
| | Kimi-K3 | 1,532 GiB | 4,422 | 2,880 |
| | GLM-5.2 bf16 | 1,412 GiB | 4,360 | 3,658 |

Read this way the result is different, and more useful:

- **In the mid tier, DeepSeek-V4-Pro beats GLM-5.2 FP8 by 27%** on batch
  throughput while being 19% *larger*. That is a genuine engineering win, not a
  size artefact.
- **In the large tier, Qwen3.8-2.4T leads by ~40%** over both Kimi-K3 and
  GLM-5.2 bf16, despite sitting between them in footprint.
- **Both tiers are won by the model in the middle of its group**, not the
  smallest one. Across five comparable models, footprint predicts the broad
  band but not the order within it.

## Workload by workload

The tier table above uses two workloads. Here are all seven, in order of how
much context each one has to chew through before it can emit anything.

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="API-S peak output throughput, short structured API request. GLM-5.2 FP8 2,684, DeepSeek-V4-Pro 3,759, Qwen3.8-2.4T 2,443, GLM-5.2 bf16 2,617, Kimi-K3 1,651.">
    <line x1="346" y1="14" x2="346" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="346" y="10" text-anchor="middle">2,000</text>
    <line x1="535" y1="14" x2="535" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="535" y="10" text-anchor="middle">4,000</text>
    <text class="ax" x="156" y="169" text-anchor="start">API-S · 2,048 → 256 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="24" width="254" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="418" y="35">2,684</text>
    <text class="sub" x="467" y="35">c256</text>
    <text class="lab" x="148" y="58" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="48" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="59">3,759</text>
    <text class="sub" x="569" y="59">c512</text>
    <text class="lab" x="148" y="82" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="92" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="72" width="232" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="396" y="83">2,443</text>
    <text class="sub" x="445" y="83">c512</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="116" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="96" width="248" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="412" y="107">2,617</text>
    <text class="sub" x="461" y="107">c256</text>
    <text class="lab" x="148" y="130" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="140" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="120" width="157" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="321" y="131">1,651</text>
    <text class="sub" x="370" y="131">c256</text>
  </svg>
  <figcaption>
    <strong>API-S — 2,048 in → 256 out.</strong> Short structured requests:
    extraction, routing, JSON generation, metadata. Reasoning off, so the
    output is genuinely short. Together with CHAT-S it is the shape that
    sustains the most concurrent requests: DeepSeek-V4-Pro and Qwen3.8-2.4T
    both peak at 512 here, the other three at 256.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="CHAT-S peak output throughput, ordinary interactive chat. GLM-5.2 FP8 3,989, DeepSeek-V4-Pro 5,493, Qwen3.8-2.4T 4,115, GLM-5.2 bf16 3,658, Kimi-K3 2,880.">
    <line x1="286" y1="14" x2="286" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="286" y="10" text-anchor="middle">2,000</text>
    <line x1="416" y1="14" x2="416" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="416" y="10" text-anchor="middle">4,000</text>
    <line x1="545" y1="14" x2="545" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="545" y="10" text-anchor="middle">6,000</text>
    <text class="ax" x="156" y="169" text-anchor="start">CHAT-S · 2,048 → 512 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="24" width="259" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="423" y="35">3,989</text>
    <text class="sub" x="472" y="35">c256</text>
    <text class="lab" x="148" y="58" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="48" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="59">5,493</text>
    <text class="sub" x="569" y="59">c512</text>
    <text class="lab" x="148" y="82" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="92" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="72" width="267" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="431" y="83">4,115</text>
    <text class="sub" x="480" y="83">c512</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="116" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="96" width="237" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="401" y="107">3,658</text>
    <text class="sub" x="450" y="107">c256</text>
    <text class="lab" x="148" y="130" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="140" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="120" width="187" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="351" y="131">2,880</text>
    <text class="sub" x="400" y="131">c512</text>
  </svg>
  <figcaption>
    <strong>CHAT-S — 2,048 in → 512 out.</strong> Ordinary interactive chat.
    The default shape for anything a person is waiting on, and the one used
    for the latency comparison later in this post.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="CODE-I peak output throughput, interactive coding assistant. GLM-5.2 FP8 1,814, DeepSeek-V4-Pro 2,335, Qwen3.8-2.4T 2,004, GLM-5.2 bf16 1,458, Kimi-K3 1,390.">
    <line x1="309" y1="14" x2="309" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="309" y="10" text-anchor="middle">1,000</text>
    <line x1="461" y1="14" x2="461" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="461" y="10" text-anchor="middle">2,000</text>
    <text class="ax" x="156" y="169" text-anchor="start">CODE-I · 16,384 → 2,048 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="24" width="277" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="441" y="35">1,814</text>
    <text class="sub" x="490" y="35">c256</text>
    <text class="lab" x="148" y="58" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="48" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="59">2,335</text>
    <text class="sub" x="569" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="92" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="72" width="306" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="470" y="83">2,004</text>
    <text class="sub" x="519" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="116" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="96" width="222" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="386" y="107">1,458</text>
    <text class="sub" x="435" y="107">c192</text>
    <text class="lab" x="148" y="130" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="140" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="120" width="212" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="376" y="131">1,390</text>
    <text class="sub" x="425" y="131">c256</text>
  </svg>
  <figcaption>
    <strong>CODE-I — 16,384 in → 2,048 out.</strong> An interactive coding
    assistant: explain this function, make this edit, review this diff. A few
    files of context, a bounded answer, and a user watching.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="CHAT-L peak output throughput, long technical conversation. GLM-5.2 FP8 1,071, DeepSeek-V4-Pro 1,535, Qwen3.8-2.4T 1,039, GLM-5.2 bf16 855, Kimi-K3 774.">
    <line x1="272" y1="14" x2="272" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="272" y="10" text-anchor="middle">500</text>
    <line x1="388" y1="14" x2="388" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="388" y="10" text-anchor="middle">1,000</text>
    <line x1="504" y1="14" x2="504" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="504" y="10" text-anchor="middle">1,500</text>
    <text class="ax" x="156" y="169" text-anchor="start">CHAT-L · 32,768 → 2,048 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="24" width="249" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="413" y="35">1,071</text>
    <text class="sub" x="462" y="35">c64</text>
    <text class="lab" x="148" y="58" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="48" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="59">1,535</text>
    <text class="sub" x="569" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="92" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="72" width="241" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="405" y="83">1,039</text>
    <text class="sub" x="454" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="116" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="96" width="198" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="362" y="107">855</text>
    <text class="sub" x="395" y="107">c64</text>
    <text class="lab" x="148" y="130" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="140" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="120" width="180" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="344" y="131">774</text>
    <text class="sub" x="377" y="131">c256</text>
  </svg>
  <figcaption>
    <strong>CHAT-L — 32,768 in → 2,048 out.</strong> A long technical
    conversation carrying accumulated history. Sixteen times the input of
    CHAT-S for four times the output, and throughput falls by roughly the
    same factor as the input grows.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="CODE-A peak output throughput, coding-agent call, repository in context. GLM-5.2 FP8 839, DeepSeek-V4-Pro 1,328, Qwen3.8-2.4T 997, GLM-5.2 bf16 719, Kimi-K3 691.">
    <line x1="290" y1="14" x2="290" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="290" y="10" text-anchor="middle">500</text>
    <line x1="424" y1="14" x2="424" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="424" y="10" text-anchor="middle">1,000</text>
    <text class="ax" x="156" y="169" text-anchor="start">CODE-A · 65,536 → 4,096 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="24" width="225" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="389" y="35">839</text>
    <text class="sub" x="422" y="35">c64</text>
    <text class="lab" x="148" y="58" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="48" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="59">1,328</text>
    <text class="sub" x="569" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="92" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="72" width="268" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="432" y="83">997</text>
    <text class="sub" x="465" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="116" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="96" width="193" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="357" y="107">719</text>
    <text class="sub" x="390" y="107">c64</text>
    <text class="lab" x="148" y="130" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="140" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="120" width="186" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="350" y="131">691</text>
    <text class="sub" x="383" y="131">c64</text>
  </svg>
  <figcaption>
    <strong>CODE-A — 65,536 in → 4,096 out.</strong> A coding-agent model call
    with a repository in context. One call, not a full multi-step tool loop —
    real agents issue many of these back to back, so treat this as the cost of
    a single step.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="DOC-L peak output throughput, long-document analysis and RAG. GLM-5.2 FP8 258, DeepSeek-V4-Pro 384, Qwen3.8-2.4T 293, GLM-5.2 bf16 260, Kimi-K3 212.">
    <line x1="341" y1="14" x2="341" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="341" y="10" text-anchor="middle">200</text>
    <line x1="527" y1="14" x2="527" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="527" y="10" text-anchor="middle">400</text>
    <text class="ax" x="156" y="169" text-anchor="start">DOC-L · 131,072 → 2,048 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="24" width="239" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="403" y="35">258</text>
    <text class="sub" x="436" y="35">c16</text>
    <text class="lab" x="148" y="58" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="48" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="59">384</text>
    <text class="sub" x="553" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="92" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="72" width="272" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="436" y="83">293</text>
    <text class="sub" x="469" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="116" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="96" width="241" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="405" y="107">260</text>
    <text class="sub" x="438" y="107">c96</text>
    <text class="lab" x="148" y="130" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="140" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="120" width="197" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="361" y="131">212</text>
    <text class="sub" x="394" y="131">c256</text>
  </svg>
  <figcaption>
    <strong>DOC-L — 131,072 in → 2,048 out.</strong> Long-document analysis
    and RAG. The lowest output throughput in the campaign by a wide margin, and
    the most misleading row in the post if read on its own — see below. It is
    also the least reproducible: run-to-run variation reaches 24%, so
    differences under about 25% here are not real.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="BATCH-D peak output throughput, decode-heavy batch generation. GLM-5.2 FP8 7,204, DeepSeek-V4-Pro 9,183, Qwen3.8-2.4T 6,168, GLM-5.2 bf16 4,360, Kimi-K3 4,422.">
    <line x1="350" y1="14" x2="350" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="350" y="10" text-anchor="middle">5,000</text>
    <line x1="544" y1="14" x2="544" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="544" y="10" text-anchor="middle">10,000</text>
    <text class="ax" x="156" y="169" text-anchor="start">BATCH-D · 4,096 → 8,192 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="24" width="280" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="444" y="35">7,204</text>
    <text class="sub" x="493" y="35">c256</text>
    <text class="lab" x="148" y="58" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="48" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="59">9,183</text>
    <text class="sub" x="569" y="59">c512</text>
    <text class="lab" x="148" y="82" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="92" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="72" width="239" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="403" y="83">6,168</text>
    <text class="sub" x="452" y="83">c512</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="116" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="96" width="169" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="333" y="107">4,360</text>
    <text class="sub" x="382" y="107">c256</text>
    <text class="lab" x="148" y="130" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="140" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="120" width="172" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="336" y="131">4,422</text>
    <text class="sub" x="385" y="131">c1024</text>
  </svg>
  <figcaption>
    <strong>BATCH-D — 4,096 in → 8,192 out.</strong> Decode-heavy batch
    generation with EOS suppressed, so the model cannot stop early. A capacity
    probe rather than a workload anyone runs — and the source of essentially
    every headline throughput number in the industry, including the one at the
    top of this post and the one in the final section.
  </figcaption>
</figure>

### What stays the same, and what moves

One thing holds across all seven shapes: **DeepSeek-V4-Pro is first
everywhere.** No workload in this set dislodges it, which is worth stating
plainly because it is the only clean result in the comparison. At the other end,
Kimi-K3 is last everywhere except BATCH-D, where it edges GLM-5.2 bf16 by 1.4% —
inside run-to-run noise, so the bottom two are effectively tied.

The middle moves. Qwen3.8-2.4T sits second or third on six of the seven shapes
but drops to **fourth on API-S**, behind both GLM-5.2 builds. API-S is the shortest shape
in the set and the only one with reasoning disabled, so it is the closest thing
here to a pure request-rate test — and it is the one place Qwen's ranking does
not hold. If short structured calls are your traffic, the tier table earlier in
this post gives you the wrong answer.

The other thing that moves is **where each model saturates**, and how far it
falls depends on how much KV cache it has to spare:

| | API-S | CHAT-S | CODE-I | CHAT-L | CODE-A | DOC-L |
|---|--:|--:|--:|--:|--:|--:|
| input tokens | 2,048 | 2,048 | 16,384 | 32,768 | 65,536 | 131,072 |
| GLM-5.2 FP8 peaks at | c256 | c256 | c256 | c64 | c64 | **c16** |
| DeepSeek-V4-Pro peaks at | c512 | c512 | c256 | c256 | c256 | **c256** |

GLM-5.2 FP8 saturates at sixteen concurrent requests on DOC-L; DeepSeek-V4-Pro
still runs 256 on the same shape. The difference is headroom — at its BATCH-D
peak GLM-5.2 FP8 is already at 88% KV cache occupancy against Pro's 34%, so
every extra thousand tokens of context costs it a request slot much sooner.

Either way, capacity planning from a BATCH-D number would overstate what the
server holds on long context: for GLM-5.2 FP8, by a factor of sixteen.

### DOC-L is not slow, it is doing different work

Read on output tokens per second, DOC-L looks catastrophic: 24× below BATCH-D
for the same model on the same hardware. That comparison is an artefact of
measuring only half the tokens.

<figure class="qz">
  <svg viewBox="0 0 640 252" role="img" aria-label="Prefill and decode token rates for DeepSeek-V4-Pro across the seven workloads. API-S 31,793 total tokens per second, CHAT-S 24,468 total tokens per second, CODE-I 26,284 total tokens per second, CHAT-L 29,150 total tokens per second, CODE-A 28,527 total tokens per second, DOC-L 28,873 total tokens per second, BATCH-D 9,894 total tokens per second.">
  <rect x="128" y="4" width="9" height="9" rx="2" fill="var(--ok)"/>
  <text class="sub" x="141" y="12">output (decode)</text>
  <rect x="228" y="4" width="9" height="9" rx="2" fill="var(--ok2)"/>
  <text class="sub" x="241" y="12">input (prefill)</text>
  <line x1="248" y1="32" x2="248" y2="213" stroke="var(--grid)"/>
  <text class="sub" x="248" y="28" text-anchor="middle">10k</text>
  <line x1="368" y1="32" x2="368" y2="213" stroke="var(--grid)"/>
  <text class="sub" x="368" y="28" text-anchor="middle">20k</text>
  <line x1="488" y1="32" x2="488" y2="213" stroke="var(--grid)"/>
  <text class="sub" x="488" y="28" text-anchor="middle">30k</text>
  <text class="ax" x="128" y="247" text-anchor="start">Total token rate, prefill + decode — DeepSeek-V4-Pro</text>
  <text class="lab" x="120" y="51" text-anchor="end">API-S</text>
  <text class="sub" x="120" y="61" text-anchor="end">2,048→256</text>
  <rect x="128" y="40" width="45" height="15" rx="3" fill="var(--ok)"/>
  <rect x="175" y="40" width="335" height="15" rx="3" fill="var(--ok2)"/>
  <text class="val" x="518" y="52">32k</text>
  <text class="lab" x="120" y="77" text-anchor="end">CHAT-S</text>
  <text class="sub" x="120" y="87" text-anchor="end">2,048→512</text>
  <rect x="128" y="66" width="66" height="15" rx="3" fill="var(--ok)"/>
  <rect x="196" y="66" width="226" height="15" rx="3" fill="var(--ok2)"/>
  <text class="val" x="430" y="78">24k</text>
  <text class="lab" x="120" y="103" text-anchor="end">CODE-I</text>
  <text class="sub" x="120" y="113" text-anchor="end">16,384→2,048</text>
  <rect x="128" y="92" width="28" height="15" rx="3" fill="var(--ok)"/>
  <rect x="158" y="92" width="286" height="15" rx="3" fill="var(--ok2)"/>
  <text class="val" x="452" y="104">26k</text>
  <text class="lab" x="120" y="129" text-anchor="end">CHAT-L</text>
  <text class="sub" x="120" y="139" text-anchor="end">32,768→2,048</text>
  <rect x="128" y="118" width="18" height="15" rx="3" fill="var(--ok)"/>
  <rect x="148" y="118" width="330" height="15" rx="3" fill="var(--ok2)"/>
  <text class="val" x="486" y="130">29k</text>
  <text class="lab" x="120" y="155" text-anchor="end">CODE-A</text>
  <text class="sub" x="120" y="165" text-anchor="end">65,536→4,096</text>
  <rect x="128" y="144" width="16" height="15" rx="3" fill="var(--ok)"/>
  <rect x="146" y="144" width="325" height="15" rx="3" fill="var(--ok2)"/>
  <text class="val" x="479" y="156">29k</text>
  <text class="lab" x="120" y="181" text-anchor="end">DOC-L</text>
  <text class="sub" x="120" y="191" text-anchor="end">131,072→2,048</text>
  <rect x="128" y="170" width="5" height="15" rx="3" fill="var(--ok)"/>
  <rect x="135" y="170" width="340" height="15" rx="3" fill="var(--ok2)"/>
  <text class="val" x="483" y="182">29k</text>
  <text class="lab" x="120" y="207" text-anchor="end">BATCH-D</text>
  <text class="sub" x="120" y="217" text-anchor="end">4,096→8,192</text>
  <rect x="128" y="196" width="110" height="15" rx="3" fill="var(--ok)"/>
  <rect x="240" y="196" width="7" height="15" rx="3" fill="var(--ok2)"/>
  <text class="val" x="255" y="208">10k</text>
  </svg>
  <figcaption>
    The same runs, counting input tokens as well as output. Across the six
    realistic workloads the total token rate varies by only 1.3× — from 24k to
    32k tok/s — while output-only throughput varies by 10×. DOC-L is not a slow
    workload; it is a workload whose tokens are almost entirely prefill.
  </figcaption>
</figure>

The pattern holds for every model in the set: total token rate varies by
1.3–1.6× across the six realistic workloads, against a 10–28× spread in output
throughput alone. GLM-5.2 bf16 is the one exception at 2.6×.

BATCH-D goes the other way. It is the *lowest* total token rate of the seven —
about a third of the others — because it is nearly pure decode, and decode is
memory-bandwidth-bound in a way prefill is not. The workload that produces the
largest headline number is the one moving the fewest tokens per second overall.

So "tokens per second" is not one metric. It is at least two, and which one you
are quoting is decided by the input:output ratio of the shape you chose. That is
worth knowing before comparing your number to someone else's.

## Latency separates these models far more than throughput

One more reason not to rank on tok/s alone. At a fixed load of 256 concurrent
requests, time-to-first-token p95 spreads over a factor of **2.4** across these
five models, against a factor of 1.7 in throughput at the same operating point.
Latency is the more discriminating axis, and it is the one nobody puts in the
headline.

| model | TTFT p95 @ c256 | tok/s @ c256 | tok/kJ |
|---|--:|--:|--:|
| DeepSeek-V4-Pro | 9,852 ms | 4,237 | 1,423 |
| GLM-5.2 bf16 | 13,155 ms | 3,658 | 576 |
| GLM-5.2 FP8 | 13,214 ms | 3,989 | 1,003 |
| Qwen3.8-2.4T | 16,394 ms | 3,241 | 783 |
| Kimi-K3 | 23,918 ms | 2,460 | 523 |

Kimi-K3 at 23.9 seconds to first token is a batch engine, whatever its tokens
per second say. A model can be respectable on throughput and still be unusable
interactively, and the throughput chart will never tell you that.

The two GLM-5.2 builds make the other half of the point. They are within 0.5% of
each other on first-token latency — the same model, so the same amount of
prefill work — but the FP8 build gets **74% more output per kilojoule**, because
half the weight bits means half the memory traffic per forward pass. Precision
barely moves latency here and moves energy a great deal.

`tok/kJ` — output tokens per kilojoule of total GPU power, measured at each
model's BATCH-D peak — is the column that reaches an electricity bill, and it
spans 2.7× across five models whose throughput spans 2.1×.

## What I would do differently

Publishing the ranking without the size column was the mistake. It is not that
the numbers were wrong — every figure above comes from the same 414 runs — but
that the framing invited a comparison the data does not support. "Fastest model"
and "fastest model of this size" are different claims, and only the second one
helps someone choose.

If you are running a comparison like this: **record the weight footprint from
your serving logs and put it in the table from the start.** It costs nothing,
it is more honest than a parameter count for quantized models, and it stops the
headline from being a size effect in disguise.

## Caveats

- Two models were only swept to concurrency 256 rather than 1024, so their peaks
  are lower bounds. GLM-5.2 was separately confirmed flat from 256 to 1024, so
  this does not affect it; GLM-5.2 FP8 was never extended.
- Two vLLM versions are mixed across the campaign (0.24.0 and 0.27.x). I
  measured the difference rather than assuming it away: re-running GLM-5.2 on
  both gave a +1.8% mean delta, inside run-to-run noise.
- The size normalisation assumes throughput scales inversely with footprint. It
  does not, exactly. Treat that chart as a lens rather than a metric.

## The other kind of fast: DeepSeek-V4-Flash

Everything above compares five models between 714 GiB and 1,532 GiB. This is the
sixth, and it does not belong on any of those charts: **DeepSeek-V4-Flash loads
170 GiB across the node** — 4.2× smaller than the smallest of the five, 9×
smaller than Kimi-K3. Putting it on a shared axis is the mistake this whole post
is about, so here it is on its own.

<figure class="qz">
  <svg viewBox="0 0 640 242" role="img" aria-label="DeepSeek-V4-Flash peak output throughput across all seven workloads. API-S 6,404, CHAT-S 9,010, CODE-I 4,825, CHAT-L 2,482, CODE-A 2,471, DOC-L 634, BATCH-D 15,484.">
    <line x1="265" y1="20" x2="265" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="265" y="16" text-anchor="middle">5,000</text>
    <line x1="379" y1="20" x2="379" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="379" y="16" text-anchor="middle">10,000</text>
    <line x1="494" y1="20" x2="494" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="494" y="16" text-anchor="middle">15,000</text>
    <text class="ax" x="150" y="237" text-anchor="start">DeepSeek-V4-Flash · peak output tok/s by workload</text>
    <text class="lab" x="142" y="41" text-anchor="end">API-S</text>
    <text class="sub" x="142" y="51" text-anchor="end">2,048→256</text>
    <rect x="150" y="30" width="147" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="305" y="42">6,404</text>
    <text class="sub" x="354" y="42">c1024</text>
    <text class="lab" x="142" y="67" text-anchor="end">CHAT-S</text>
    <text class="sub" x="142" y="77" text-anchor="end">2,048→512</text>
    <rect x="150" y="56" width="207" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="365" y="68">9,010</text>
    <text class="sub" x="414" y="68">c512</text>
    <text class="lab" x="142" y="93" text-anchor="end">CODE-I</text>
    <text class="sub" x="142" y="103" text-anchor="end">16,384→2,048</text>
    <rect x="150" y="82" width="111" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="269" y="94">4,825</text>
    <text class="sub" x="318" y="94">c256</text>
    <text class="lab" x="142" y="119" text-anchor="end">CHAT-L</text>
    <text class="sub" x="142" y="129" text-anchor="end">32,768→2,048</text>
    <rect x="150" y="108" width="57" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="215" y="120">2,482</text>
    <text class="sub" x="264" y="120">c256</text>
    <text class="lab" x="142" y="145" text-anchor="end">CODE-A</text>
    <text class="sub" x="142" y="155" text-anchor="end">65,536→4,096</text>
    <rect x="150" y="134" width="57" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="215" y="146">2,471</text>
    <text class="sub" x="264" y="146">c256</text>
    <text class="lab" x="142" y="171" text-anchor="end">DOC-L</text>
    <text class="sub" x="142" y="181" text-anchor="end">131,072→2,048</text>
    <rect x="150" y="160" width="15" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="173" y="172">634</text>
    <text class="sub" x="206" y="172">c64</text>
    <text class="lab" x="142" y="197" text-anchor="end">BATCH-D</text>
    <text class="sub" x="142" y="207" text-anchor="end">4,096→8,192</text>
    <rect x="150" y="186" width="355" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="513" y="198">15,484</text>
    <text class="sub" x="570" y="198">c512</text>
  </svg>
  <figcaption>
    DeepSeek-V4-Flash across all seven workloads, peak output tok/s with the
    concurrency that produced it. Same node, same vLLM, same sweep as every
    other number in this post — MXFP4 weights, speculative decoding disabled.
  </figcaption>
</figure>

15,484 tok/s on BATCH-D, and 634 on DOC-L — it obeys the same workload physics
as everything else, falling by 24× between the decode-heavy probe and the
prefill-heavy one. What changes is the level.

<figure class="qz">
  <svg viewBox="0 0 640 242" role="img" aria-label="DeepSeek-V4-Flash speed-up over the best of the five-model field, by workload. API-S 1.70×, CHAT-S 1.64×, CODE-I 2.07×, CHAT-L 1.62×, CODE-A 1.86×, DOC-L 1.65×, BATCH-D 1.69×.">
    <line x1="322" y1="20" x2="322" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="322" y="16" text-anchor="middle">1.00×</text>
    <line x1="494" y1="20" x2="494" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="494" y="16" text-anchor="middle">2.00×</text>
    <line x1="322" y1="20" x2="322" y2="205" stroke="var(--ref)" stroke-width="1.5"/>
    <text class="ax" x="150" y="237" text-anchor="start">DeepSeek-V4-Flash ÷ best of the five-model field</text>
    <text class="lab" x="142" y="41" text-anchor="end">API-S</text>
    <text class="sub" x="142" y="51" text-anchor="end">2,048→256</text>
    <rect x="150" y="30" width="293" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="451" y="42">1.70×</text>
    <text class="sub" x="500" y="42">3,759 tok/s</text>
    <text class="lab" x="142" y="67" text-anchor="end">CHAT-S</text>
    <text class="sub" x="142" y="77" text-anchor="end">2,048→512</text>
    <rect x="150" y="56" width="282" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="440" y="68">1.64×</text>
    <text class="sub" x="489" y="68">5,493 tok/s</text>
    <text class="lab" x="142" y="93" text-anchor="end">CODE-I</text>
    <text class="sub" x="142" y="103" text-anchor="end">16,384→2,048</text>
    <rect x="150" y="82" width="355" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="513" y="94">2.07×</text>
    <text class="sub" x="562" y="94">2,335 tok/s</text>
    <text class="lab" x="142" y="119" text-anchor="end">CHAT-L</text>
    <text class="sub" x="142" y="129" text-anchor="end">32,768→2,048</text>
    <rect x="150" y="108" width="278" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="436" y="120">1.62×</text>
    <text class="sub" x="485" y="120">1,535 tok/s</text>
    <text class="lab" x="142" y="145" text-anchor="end">CODE-A</text>
    <text class="sub" x="142" y="155" text-anchor="end">65,536→4,096</text>
    <rect x="150" y="134" width="320" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="478" y="146">1.86×</text>
    <text class="sub" x="527" y="146">1,328 tok/s</text>
    <text class="lab" x="142" y="171" text-anchor="end">DOC-L</text>
    <text class="sub" x="142" y="181" text-anchor="end">131,072→2,048</text>
    <rect x="150" y="160" width="284" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="442" y="172">1.65×</text>
    <text class="sub" x="491" y="172">384 tok/s</text>
    <text class="lab" x="142" y="197" text-anchor="end">BATCH-D</text>
    <text class="sub" x="142" y="207" text-anchor="end">4,096→8,192</text>
    <rect x="150" y="186" width="290" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="448" y="198">1.69×</text>
    <text class="sub" x="497" y="198">9,183 tok/s</text>
  </svg>
  <figcaption>
    Flash divided by the best of the five-model field on each workload — which
    is DeepSeek-V4-Pro on all seven. The vertical rule is parity. The striking
    thing is how flat this is: Flash is not winning one shape, it is winning
    every shape by roughly the same margin.
  </figcaption>
</figure>

**Flash beats the best of the field by 1.6× to 2.1× on every workload**, and
the spread is narrow — 1.62× on CHAT-L, 2.07× on CODE-I. It is not exploiting
one shape; it is simply operating at a different level throughout.

It is even further ahead on the axes nobody puts in a headline:

| | Flash | best of the five | |
|---|--:|--:|--:|
| TTFT p95 @ c256 | **577 ms** | 9,852 ms | 17× faster |
| output per kilojoule | **3,507 tok/kJ** | 1,423 | 2.5× |
| GPU power at BATCH-D peak | **4,416 W** | 6,453 W | lowest in the set |
| KV cache used at that peak | **16%** | 34% | three of the five sit at 100% |

Sub-second time to first token at 256 concurrent requests is a different product
from nine seconds, not a faster version of the same one. And Flash draws the
least power of any model here while producing the most tokens, so the energy gap
compounds the throughput gap rather than trading against it.

### The part that argues against itself

Here is the number that keeps this honest. Flash is **5.0× smaller** than
DeepSeek-V4-Pro and delivers **1.7× the throughput**. Against GLM-5.2 FP8:
4.2× smaller, 2.1× the throughput.

If throughput scaled inversely with size, a 5× smaller model would be 5× faster.
It is not — it is a third of the way there. Shrinking the model buys much less
than proportional throughput, which is exactly why the per-GiB chart earlier
carries a warning label. That chart credits Flash with 91 tok/s per GiB against
Pro's 11, an 8.4× lead, and that ratio flatters it badly.

So the fair summary is narrower than the headline: on this hardware a model a
fifth of the size delivers a bit under twice the throughput, an order of
magnitude better first-token latency, and two and a half times the energy
efficiency. That is a real and useful trade — it is just not the 5× that
"nine times smaller" invites you to imagine, and it says nothing about how Flash
compares to another model *its own size*, because there isn't one in this set.

Two caveats specific to this section. **Speculative decoding is disabled** — the
stock configuration crashed three times with illegal-instruction errors inside
DeepGEMM, and the spec-decode path is 2–3× faster below concurrency 16 but up to
33% *slower* at 256, so it is the wrong configuration for a throughput
comparison in either case. And **Flash is the only model in its size class
here**, so every comparison on this page is against models it was never really
competing with.
