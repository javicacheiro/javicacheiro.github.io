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
  --bad: #c2410c;
  --ref: hsl(var(--muted-foreground));
  --grid: hsl(var(--border));
  --ink: hsl(var(--foreground));
  --dim: hsl(var(--muted-foreground));
  margin: 2rem 0;
}
.dark .qz { --ok: #2f9e8f; --bad: #cf6238; }
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
