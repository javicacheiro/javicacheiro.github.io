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

Here is the same data with size put back in.

## How big these models actually are

Parameter counts are awkward for this comparison: several of these are
mixture-of-experts, published counts mix total and active parameters, and two of
the checkpoints are 4-bit. So instead of a number from a model card, I use one I
measured — **what vLLM reported loading**, summed across the eight GPUs:

<figure class="qz">
  <svg viewBox="0 0 640 220" role="img" aria-label="Weight footprint. DeepSeek-V4-Flash 170 GiB, GLM-5.2 FP8 714 GiB, DeepSeek-V4-Pro 850 GiB, Qwen3.8-2.4T 1,340 GiB, GLM-5.2 bf16 1,412 GiB, Kimi-K3 1,532 GiB.">
    <line x1="279" y1="14" x2="279" y2="180" stroke="var(--grid)"/>
    <line x1="402" y1="14" x2="402" y2="180" stroke="var(--grid)"/>
    <line x1="525" y1="14" x2="525" y2="180" stroke="var(--grid)"/>
    <text class="ax" x="156" y="214" text-anchor="start">Weight footprint — measured across the 8-GPU node, precision as served</text>
    <text class="lab" x="148" y="37" text-anchor="end">DeepSeek-V4-Flash</text>
    <text class="sub" x="148" y="48" text-anchor="end">MXFP4 · 170 GiB</text>
    <rect x="156" y="26" width="42" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="206" y="38">170 GiB</text>
    <text class="lab" x="148" y="64" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="75" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="53" width="176" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="340" y="65">714 GiB</text>
    <text class="lab" x="148" y="91" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="102" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="80" width="209" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="373" y="92">850 GiB</text>
    <text class="lab" x="148" y="118" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="129" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="107" width="330" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="494" y="119">1,340 GiB</text>
    <text class="lab" x="148" y="145" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="156" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="134" width="348" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="512" y="146">1,412 GiB</text>
    <text class="lab" x="148" y="172" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="183" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="161" width="377" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="541" y="173">1,532 GiB</text>
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
  <svg viewBox="0 0 640 220" role="img" aria-label="Peak batch throughput. DeepSeek-V4-Flash 15,484, GLM-5.2 FP8 7,204, DeepSeek-V4-Pro 9,183, Qwen3.8-2.4T 6,168, GLM-5.2 bf16 4,360, Kimi-K3 4,422.">
    <line x1="278" y1="14" x2="278" y2="180" stroke="var(--grid)"/>
    <line x1="400" y1="14" x2="400" y2="180" stroke="var(--grid)"/>
    <line x1="522" y1="14" x2="522" y2="180" stroke="var(--grid)"/>
    <text class="ax" x="156" y="214" text-anchor="start">Peak batch throughput — BATCH-D, best concurrency per model</text>
    <text class="lab" x="148" y="37" text-anchor="end">DeepSeek-V4-Flash</text>
    <text class="sub" x="148" y="48" text-anchor="end">MXFP4 · 170 GiB</text>
    <rect x="156" y="26" width="377" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="541" y="38">15,484</text>
    <text class="lab" x="148" y="64" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="75" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="53" width="176" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="340" y="65">7,204</text>
    <text class="lab" x="148" y="91" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="102" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="80" width="224" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="388" y="92">9,183</text>
    <text class="lab" x="148" y="118" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="129" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="107" width="150" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="314" y="119">6,168</text>
    <text class="lab" x="148" y="145" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="156" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="134" width="106" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="270" y="146">4,360</text>
    <text class="lab" x="148" y="172" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="183" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="161" width="108" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="272" y="173">4,422</text>
  </svg>
  <figcaption>
    Peak sustained output on BATCH-D, each model at whichever concurrency
    maximised it. Headline metric is <code>server_output_tps</code> as reported
    by vLLM — what an operator sees on their own dashboard.
  </figcaption>
</figure>

DeepSeek-V4-Flash produces **15,484 tok/s**, 1.7× the next-best model, while
occupying a fifth of the memory. Read alongside the previous chart, most of that
lead is explained before you reach any question of engineering quality: it is
simply a much smaller model.

## Normalising by size

<figure class="qz">
  <svg viewBox="0 0 640 220" role="img" aria-label="Throughput per GiB of weights. DeepSeek-V4-Flash 91.1, GLM-5.2 FP8 10.1, DeepSeek-V4-Pro 10.8, Qwen3.8-2.4T 4.6, GLM-5.2 bf16 3.1, Kimi-K3 2.9.">
    <line x1="363" y1="14" x2="363" y2="180" stroke="var(--grid)"/>
    <text class="ax" x="156" y="214" text-anchor="start">Throughput per GiB of weights — peak BATCH-D tok/s ÷ node weight footprint</text>
    <text class="lab" x="148" y="37" text-anchor="end">DeepSeek-V4-Flash</text>
    <text class="sub" x="148" y="48" text-anchor="end">MXFP4 · 170 GiB</text>
    <rect x="156" y="26" width="377" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="541" y="38">91.1</text>
    <text class="lab" x="148" y="64" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="75" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="53" width="42" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="206" y="65">10.1</text>
    <text class="lab" x="148" y="91" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="102" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="80" width="45" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="209" y="92">10.8</text>
    <text class="lab" x="148" y="118" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="129" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="107" width="19" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="183" y="119">4.6</text>
    <text class="lab" x="148" y="145" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="156" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="134" width="13" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="177" y="146">3.1</text>
    <text class="lab" x="148" y="172" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="183" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="161" width="12" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="176" y="173">2.9</text>
  </svg>
  <figcaption>
    Peak BATCH-D throughput divided by node weight footprint. A crude
    normalisation — it assumes throughput should scale inversely with model
    size, which is only roughly true — but it separates "fast because small"
    from "fast because well-built".
  </figcaption>
</figure>

Flash still leads, and now by **8×** rather than 1.7×. That is more than its
size advantage alone explains, so something else is going on: sparser expert
activation, a smaller KV footprint per token, and MXFP4 weights that move less
memory per forward pass.

I would not push this metric far. Dividing by size assumes a linear relationship
that does not really hold, and it flatters small models. It is a lens, not a
verdict.

## The fair fight: comparing within size tiers

The most useful comparison is between models someone would actually be choosing
between — ones of comparable size, competing for the same GPUs.

| tier | model | footprint | BATCH-D | CHAT-S |
|---|---|--:|--:|--:|
| **small** | DeepSeek-V4-Flash | 170 GiB | 15,484 | 9,010 |
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
- **DeepSeek-V4-Flash has no peer in this set.** It is alone in its tier, so
  "it wins" says nothing about how it compares to another model of its size —
  only that a much smaller model is much faster, which we knew.

## Workload by workload

The tier table above uses two workloads. Here are all seven, in order of how
much context each one has to chew through before it can emit anything.

<figure class="qz">
  <svg viewBox="0 0 640 198" role="img" aria-label="API-S peak output throughput, short structured API request. DeepSeek-V4-Flash 6,404, GLM-5.2 FP8 2,684, DeepSeek-V4-Pro 3,759, Qwen3.8-2.4T 2,443, GLM-5.2 bf16 2,617, Kimi-K3 1,651.">
    <line x1="267" y1="14" x2="267" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="267" y="10" text-anchor="middle">2,000</text>
    <line x1="379" y1="14" x2="379" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="379" y="10" text-anchor="middle">4,000</text>
    <line x1="490" y1="14" x2="490" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="490" y="10" text-anchor="middle">6,000</text>
    <text class="ax" x="156" y="193" text-anchor="start">API-S · 2,048 → 256 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Flash</text>
    <text class="sub" x="148" y="44" text-anchor="end">MXFP4 · 170 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--bad)"/>
    <text class="val" x="520" y="35">6,404</text>
    <text class="sub" x="569" y="35">c1024</text>
    <text class="lab" x="148" y="58" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="48" width="149" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="313" y="59">2,684</text>
    <text class="sub" x="362" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="92" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="72" width="209" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="373" y="83">3,759</text>
    <text class="sub" x="422" y="83">c512</text>
    <text class="lab" x="148" y="106" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="116" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="96" width="136" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="300" y="107">2,443</text>
    <text class="sub" x="349" y="107">c512</text>
    <text class="lab" x="148" y="130" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="140" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="120" width="146" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="310" y="131">2,617</text>
    <text class="sub" x="359" y="131">c256</text>
    <text class="lab" x="148" y="154" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="164" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="144" width="92" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="256" y="155">1,651</text>
    <text class="sub" x="305" y="155">c256</text>
  </svg>
  <figcaption>
    <strong>API-S — 2,048 in → 256 out.</strong> Short structured requests:
    extraction, routing, JSON generation, metadata. Reasoning off, so the
    output is genuinely short. This is the shape that scales to the highest
    concurrency of any workload here — Flash peaks at 1,024 concurrent
    requests, further than any other cell in the campaign.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 198" role="img" aria-label="CHAT-S peak output throughput, ordinary interactive chat. DeepSeek-V4-Flash 9,010, GLM-5.2 FP8 3,989, DeepSeek-V4-Pro 5,493, Qwen3.8-2.4T 4,115, GLM-5.2 bf16 3,658, Kimi-K3 2,880.">
    <line x1="255" y1="14" x2="255" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="255" y="10" text-anchor="middle">2,500</text>
    <line x1="354" y1="14" x2="354" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="354" y="10" text-anchor="middle">5,000</text>
    <line x1="453" y1="14" x2="453" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="453" y="10" text-anchor="middle">7,500</text>
    <text class="ax" x="156" y="193" text-anchor="start">CHAT-S · 2,048 → 512 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Flash</text>
    <text class="sub" x="148" y="44" text-anchor="end">MXFP4 · 170 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--bad)"/>
    <text class="val" x="520" y="35">9,010</text>
    <text class="sub" x="569" y="35">c512</text>
    <text class="lab" x="148" y="58" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="48" width="158" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="322" y="59">3,989</text>
    <text class="sub" x="371" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="92" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="72" width="217" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="381" y="83">5,493</text>
    <text class="sub" x="430" y="83">c512</text>
    <text class="lab" x="148" y="106" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="116" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="96" width="163" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="327" y="107">4,115</text>
    <text class="sub" x="376" y="107">c512</text>
    <text class="lab" x="148" y="130" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="140" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="120" width="145" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="309" y="131">3,658</text>
    <text class="sub" x="358" y="131">c256</text>
    <text class="lab" x="148" y="154" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="164" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="144" width="114" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="278" y="155">2,880</text>
    <text class="sub" x="327" y="155">c512</text>
  </svg>
  <figcaption>
    <strong>CHAT-S — 2,048 in → 512 out.</strong> Ordinary interactive chat.
    The default shape for anything a person is waiting on, and the one used
    for the latency comparison later in this post.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 198" role="img" aria-label="CODE-I peak output throughput, interactive coding assistant. DeepSeek-V4-Flash 4,825, GLM-5.2 FP8 1,814, DeepSeek-V4-Pro 2,335, Qwen3.8-2.4T 2,004, GLM-5.2 bf16 1,458, Kimi-K3 1,390.">
    <line x1="304" y1="14" x2="304" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="304" y="10" text-anchor="middle">2,000</text>
    <line x1="451" y1="14" x2="451" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="451" y="10" text-anchor="middle">4,000</text>
    <text class="ax" x="156" y="193" text-anchor="start">CODE-I · 16,384 → 2,048 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Flash</text>
    <text class="sub" x="148" y="44" text-anchor="end">MXFP4 · 170 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--bad)"/>
    <text class="val" x="520" y="35">4,825</text>
    <text class="sub" x="569" y="35">c256</text>
    <text class="lab" x="148" y="58" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="48" width="134" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="298" y="59">1,814</text>
    <text class="sub" x="347" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="92" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="72" width="172" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="336" y="83">2,335</text>
    <text class="sub" x="385" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="116" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="96" width="148" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="312" y="107">2,004</text>
    <text class="sub" x="361" y="107">c256</text>
    <text class="lab" x="148" y="130" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="140" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="120" width="108" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="272" y="131">1,458</text>
    <text class="sub" x="321" y="131">c192</text>
    <text class="lab" x="148" y="154" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="164" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="144" width="103" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="267" y="155">1,390</text>
    <text class="sub" x="316" y="155">c256</text>
  </svg>
  <figcaption>
    <strong>CODE-I — 16,384 in → 2,048 out.</strong> An interactive coding
    assistant: explain this function, make this edit, review this diff. A few
    files of context, a bounded answer, and a user watching.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 198" role="img" aria-label="CHAT-L peak output throughput, long technical conversation. DeepSeek-V4-Flash 2,482, GLM-5.2 FP8 1,071, DeepSeek-V4-Pro 1,535, Qwen3.8-2.4T 1,039, GLM-5.2 bf16 855, Kimi-K3 774.">
    <line x1="300" y1="14" x2="300" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="300" y="10" text-anchor="middle">1,000</text>
    <line x1="443" y1="14" x2="443" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="443" y="10" text-anchor="middle">2,000</text>
    <text class="ax" x="156" y="193" text-anchor="start">CHAT-L · 32,768 → 2,048 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Flash</text>
    <text class="sub" x="148" y="44" text-anchor="end">MXFP4 · 170 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--bad)"/>
    <text class="val" x="520" y="35">2,482</text>
    <text class="sub" x="569" y="35">c256</text>
    <text class="lab" x="148" y="58" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="48" width="154" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="318" y="59">1,071</text>
    <text class="sub" x="367" y="59">c64</text>
    <text class="lab" x="148" y="82" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="92" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="72" width="220" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="384" y="83">1,535</text>
    <text class="sub" x="433" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="116" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="96" width="149" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="313" y="107">1,039</text>
    <text class="sub" x="362" y="107">c256</text>
    <text class="lab" x="148" y="130" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="140" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="120" width="123" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="287" y="131">855</text>
    <text class="sub" x="320" y="131">c64</text>
    <text class="lab" x="148" y="154" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="164" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="144" width="111" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="275" y="155">774</text>
    <text class="sub" x="308" y="155">c256</text>
  </svg>
  <figcaption>
    <strong>CHAT-L — 32,768 in → 2,048 out.</strong> A long technical
    conversation carrying accumulated history. Sixteen times the input of
    CHAT-S for four times the output, and throughput falls by roughly the
    same factor as the input grows.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 198" role="img" aria-label="CODE-A peak output throughput, coding-agent call, repository in context. DeepSeek-V4-Flash 2,471, GLM-5.2 FP8 839, DeepSeek-V4-Pro 1,328, Qwen3.8-2.4T 997, GLM-5.2 bf16 719, Kimi-K3 691.">
    <line x1="300" y1="14" x2="300" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="300" y="10" text-anchor="middle">1,000</text>
    <line x1="444" y1="14" x2="444" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="444" y="10" text-anchor="middle">2,000</text>
    <text class="ax" x="156" y="193" text-anchor="start">CODE-A · 65,536 → 4,096 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Flash</text>
    <text class="sub" x="148" y="44" text-anchor="end">MXFP4 · 170 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--bad)"/>
    <text class="val" x="520" y="35">2,471</text>
    <text class="sub" x="569" y="35">c256</text>
    <text class="lab" x="148" y="58" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="48" width="121" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="285" y="59">839</text>
    <text class="sub" x="318" y="59">c64</text>
    <text class="lab" x="148" y="82" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="92" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="72" width="191" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="355" y="83">1,328</text>
    <text class="sub" x="404" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="116" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="96" width="144" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="308" y="107">997</text>
    <text class="sub" x="341" y="107">c256</text>
    <text class="lab" x="148" y="130" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="140" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="120" width="104" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="268" y="131">719</text>
    <text class="sub" x="301" y="131">c64</text>
    <text class="lab" x="148" y="154" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="164" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="144" width="100" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="264" y="155">691</text>
    <text class="sub" x="297" y="155">c64</text>
  </svg>
  <figcaption>
    <strong>CODE-A — 65,536 in → 4,096 out.</strong> A coding-agent model call
    with a repository in context. One call, not a full multi-step tool loop —
    real agents issue many of these back to back, so treat this as the cost of
    a single step.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 198" role="img" aria-label="DOC-L peak output throughput, long-document analysis and RAG. DeepSeek-V4-Flash 634, GLM-5.2 FP8 258, DeepSeek-V4-Pro 384, Qwen3.8-2.4T 293, GLM-5.2 bf16 260, Kimi-K3 212.">
    <line x1="268" y1="14" x2="268" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="268" y="10" text-anchor="middle">200</text>
    <line x1="381" y1="14" x2="381" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="381" y="10" text-anchor="middle">400</text>
    <line x1="493" y1="14" x2="493" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="493" y="10" text-anchor="middle">600</text>
    <text class="ax" x="156" y="193" text-anchor="start">DOC-L · 131,072 → 2,048 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Flash</text>
    <text class="sub" x="148" y="44" text-anchor="end">MXFP4 · 170 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--bad)"/>
    <text class="val" x="520" y="35">634</text>
    <text class="sub" x="553" y="35">c64</text>
    <text class="lab" x="148" y="58" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="48" width="145" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="309" y="59">258</text>
    <text class="sub" x="342" y="59">c16</text>
    <text class="lab" x="148" y="82" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="92" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="72" width="216" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="380" y="83">384</text>
    <text class="sub" x="413" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="116" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="96" width="165" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="329" y="107">293</text>
    <text class="sub" x="362" y="107">c256</text>
    <text class="lab" x="148" y="130" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="140" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="120" width="146" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="310" y="131">260</text>
    <text class="sub" x="343" y="131">c96</text>
    <text class="lab" x="148" y="154" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="164" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="144" width="119" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="283" y="155">212</text>
    <text class="sub" x="316" y="155">c256</text>
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
  <svg viewBox="0 0 640 198" role="img" aria-label="BATCH-D peak output throughput, decode-heavy batch generation. DeepSeek-V4-Flash 15,484, GLM-5.2 FP8 7,204, DeepSeek-V4-Pro 9,183, Qwen3.8-2.4T 6,168, GLM-5.2 bf16 4,360, Kimi-K3 4,422.">
    <line x1="271" y1="14" x2="271" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="271" y="10" text-anchor="middle">5,000</text>
    <line x1="386" y1="14" x2="386" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="386" y="10" text-anchor="middle">10,000</text>
    <line x1="501" y1="14" x2="501" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="501" y="10" text-anchor="middle">15,000</text>
    <text class="ax" x="156" y="193" text-anchor="start">BATCH-D · 4,096 → 8,192 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Flash</text>
    <text class="sub" x="148" y="44" text-anchor="end">MXFP4 · 170 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--bad)"/>
    <text class="val" x="520" y="35">15,484</text>
    <text class="sub" x="577" y="35">c512</text>
    <text class="lab" x="148" y="58" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="48" width="166" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="330" y="59">7,204</text>
    <text class="sub" x="379" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="92" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="72" width="211" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="375" y="83">9,183</text>
    <text class="sub" x="424" y="83">c512</text>
    <text class="lab" x="148" y="106" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="116" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="96" width="142" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="306" y="107">6,168</text>
    <text class="sub" x="355" y="107">c512</text>
    <text class="lab" x="148" y="130" text-anchor="end">GLM-5.2 bf16</text>
    <text class="sub" x="148" y="140" text-anchor="end">bf16 · 1,412 GiB</text>
    <rect x="156" y="120" width="100" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="264" y="131">4,360</text>
    <text class="sub" x="313" y="131">c256</text>
    <text class="lab" x="148" y="154" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="164" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="144" width="102" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="266" y="155">4,422</text>
    <text class="sub" x="315" y="155">c1024</text>
  </svg>
  <figcaption>
    <strong>BATCH-D — 4,096 in → 8,192 out.</strong> Decode-heavy batch
    generation with EOS suppressed, so the model cannot stop early. A capacity
    probe rather than a workload anyone runs — and the source of essentially
    every headline throughput number in the industry, including the one at the
    top of this post.
  </figcaption>
</figure>

### What stays the same, and what moves

Two things hold across all seven shapes. **DeepSeek-V4-Flash is first
everywhere and DeepSeek-V4-Pro is second everywhere** — no workload in this set
reorders the top two. And Kimi-K3 is last everywhere except BATCH-D, where it
edges GLM-5.2 bf16 by 1.4%, which is inside run-to-run noise.

The middle does move. Qwen3.8-2.4T sits third or fourth on five workloads but
drops to **fifth on API-S**, behind both GLM-5.2 builds. API-S is the shortest
shape in the set and the only one with reasoning disabled, so it is the closest
thing here to a pure request-rate test — and it is the one place Qwen's ranking
does not hold. If short structured calls are your traffic, the tier table
earlier in this post gives you the wrong answer.

The other thing that moves is **where each model saturates**. Peak concurrency
falls monotonically as input length grows, because a long-context request holds
far more KV cache while it waits:

| | API-S | CHAT-S | CODE-I | CHAT-L | CODE-A | DOC-L |
|---|--:|--:|--:|--:|--:|--:|
| input tokens | 2,048 | 2,048 | 16,384 | 32,768 | 65,536 | 131,072 |
| GLM-5.2 FP8 peaks at | c256 | c256 | c256 | c64 | c64 | **c16** |
| DeepSeek-V4-Flash peaks at | c1024 | c512 | c256 | c256 | c256 | **c64** |

GLM-5.2 FP8 saturates at sixteen concurrent requests on DOC-L. Capacity planning
from a BATCH-D number — where the same model happily runs 256 — would overstate
what that server can hold by more than an order of magnitude.

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
requests, time-to-first-token p95 spans **577 ms to 23,918 ms** — a factor of
41, against a factor of 3.5 in throughput over the same models.

| model | TTFT p95 @ c256 | tok/kJ |
|---|--:|--:|
| DeepSeek-V4-Flash | 577 ms | 3,507 |
| DeepSeek-V4-Pro | 9,852 ms | 1,423 |
| GLM-5.2 bf16 | 13,155 ms | 576 |
| GLM-5.2 FP8 | 13,214 ms | 1,003 |
| Qwen3.8-2.4T | 16,394 ms | 783 |
| Kimi-K3 | 23,918 ms | 523 |

Kimi-K3 at 23.9 seconds to first token is a batch engine, whatever its tokens
per second say. A model can be respectable on throughput and still be unusable
interactively, and the throughput chart will never tell you that.

`tok/kJ` — output tokens per kilojoule of total GPU power — is the column that
reaches an electricity bill. Flash draws the *least* power of any model at its
peak while producing the most tokens, which compounds its advantage well beyond
the throughput number.

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
- DeepSeek-V4-Flash is shown with speculative decoding **disabled**. Its stock
  configuration crashed three times with illegal-instruction errors inside
  DeepGEMM; the spec-decode path is 2–3× faster below concurrency 16 and up to
  33% *slower* at 256, so it is the wrong configuration for a throughput
  comparison anyway.
- The size normalisation assumes throughput scales inversely with footprint. It
  does not, exactly. Treat that chart as a lens rather than a metric.
