---
title: 'Six ways to serve GLM-5.3 on one 8×H200 server'
description: 'Two models, two engines, two tuning strategies. The recipe you pick matters more than the engine.'
publishDate: 2026-09-20
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
  --vl: #4f5bd5;
  --grid: hsl(var(--border));
  --ink: hsl(var(--foreground));
  --dim: hsl(var(--muted-foreground));
  margin: 2rem 0;
}
.dark .qz { --ok: #2f9e8f; --bad: #cf6238; --vl: #8b93e8; }
.qz svg { width: 100%; height: auto; display: block; overflow: visible; }
.qz figcaption { font-size: .875rem; line-height: 1.55; color: var(--dim); margin-top: .8rem; }
.qz .lab { font-size: 12px; fill: var(--ink); }
.qz .sub { font-size: 10.5px; fill: var(--dim); }
.qz .val { font-size: 12px; font-weight: 600; fill: var(--ink); }
.qz .ax  { font-size: 11px; fill: var(--dim); }
</style>

Z-AI released GLM-5.3 and GLM-5.3-Flash with published serving recipes for both
vLLM and SGLang, and each recipe comes in a low-latency and a high-throughput
variant. That is six ways to serve two models on the same hardware, so I ran all
six on one 8×H200 node and measured them on the same seven workload shapes I used
for the [B300 comparison](/blog/throughput-benchmarking-on-b300/).

The short version: **which recipe you pick matters more than which engine you
pick**, and the right answer flips somewhere between 16 and 64 concurrent
requests.

## One user at a time

<figure class="qz">
  <svg viewBox="0 0 640 216" role="img" aria-label="Single-stream output throughput for six serving configurations. SGLang low-latency 195, SGLang low-latency 174, vLLM 150, vLLM 147, SGLang high-throughput 113, SGLang high-throughput 96.">
      <line x1="361" y1="20" x2="361" y2="179" stroke="var(--grid)"/>
      <text class="sub" x="361" y="16" text-anchor="middle">100</text>
      <line x1="526" y1="20" x2="526" y2="179" stroke="var(--grid)"/>
      <text class="sub" x="526" y="16" text-anchor="middle">200</text>
      <text class="ax" x="196" y="211" text-anchor="start">One request at a time · CHAT-S · output tok/s</text>
      <text class="lab" x="188" y="40" text-anchor="end">SGLang low-latency</text>
      <text class="sub" x="188" y="50" text-anchor="end">GLM-5.3 · 4.71 ms / token</text>
      <rect x="196" y="30" width="321" height="14" rx="3" fill="var(--ok)"/>
      <text class="val" x="525" y="41">195</text>
      <text class="lab" x="188" y="66" text-anchor="end">SGLang low-latency</text>
      <text class="sub" x="188" y="76" text-anchor="end">Flash · 5.31 ms / token</text>
      <rect x="196" y="56" width="288" height="14" rx="3" fill="var(--ok)"/>
      <text class="val" x="492" y="67">174</text>
      <text class="lab" x="188" y="92" text-anchor="end">vLLM</text>
      <text class="sub" x="188" y="102" text-anchor="end">Flash · 6.10 ms / token</text>
      <rect x="196" y="82" width="248" height="14" rx="3" fill="var(--vl)"/>
      <text class="val" x="452" y="93">150</text>
      <text class="lab" x="188" y="118" text-anchor="end">vLLM</text>
      <text class="sub" x="188" y="128" text-anchor="end">GLM-5.3 · 6.41 ms / token</text>
      <rect x="196" y="108" width="242" height="14" rx="3" fill="var(--vl)"/>
      <text class="val" x="446" y="119">147</text>
      <text class="lab" x="188" y="144" text-anchor="end">SGLang high-throughput</text>
      <text class="sub" x="188" y="154" text-anchor="end">Flash · 8.64 ms / token</text>
      <rect x="196" y="134" width="186" height="14" rx="3" fill="var(--bad)"/>
      <text class="val" x="390" y="145">113</text>
      <text class="lab" x="188" y="170" text-anchor="end">SGLang high-throughput</text>
      <text class="sub" x="188" y="180" text-anchor="end">GLM-5.3 · 10.21 ms / token</text>
      <rect x="196" y="160" width="158" height="14" rx="3" fill="var(--bad)"/>
      <text class="val" x="362" y="171">96</text>
  </svg>
  <figcaption>
    A single request, CHAT-S shape (2,048 in → 512 out). The sublabel is the
    inter-token latency — how long a reader waits between words.
  </figcaption>
</figure>

If one person is talking to the model, the low-latency recipes win and the bigger
model wins. GLM-5.3 on SGLang's low-latency cell emits **195 tok/s with 4.71 ms
between tokens**, which is faster than the much smaller Flash model on any
configuration. Both low-latency cells run EAGLE speculative decoding with a
5-step draft, and at batch 1 that draft is nearly free.

Note what happens to the high-throughput cells here: they are the *slowest* two
rows. GLM-5.3 high-throughput manages 96 tok/s, half of its low-latency sibling.

## Under load

<figure class="qz">
  <svg viewBox="0 0 640 216" role="img" aria-label="Peak batch output throughput for six serving configurations. SGLang high-throughput 14,210, SGLang high-throughput 6,500, vLLM 5,097, SGLang low-latency 4,149, vLLM 2,014, SGLang low-latency 1,636.">
      <line x1="309" y1="20" x2="309" y2="179" stroke="var(--grid)"/>
      <text class="sub" x="309" y="16" text-anchor="middle">5,000</text>
      <line x1="422" y1="20" x2="422" y2="179" stroke="var(--grid)"/>
      <text class="sub" x="422" y="16" text-anchor="middle">10,000</text>
      <line x1="535" y1="20" x2="535" y2="179" stroke="var(--grid)"/>
      <text class="sub" x="535" y="16" text-anchor="middle">15,000</text>
      <text class="ax" x="196" y="211" text-anchor="start">Peak under load · BATCH-D · output tok/s</text>
      <text class="lab" x="188" y="40" text-anchor="end">SGLang high-throughput</text>
      <text class="sub" x="188" y="50" text-anchor="end">Flash · peak of c1–c256</text>
      <rect x="196" y="30" width="321" height="14" rx="3" fill="var(--bad)"/>
      <text class="val" x="525" y="41">14,210</text>
      <text class="lab" x="188" y="66" text-anchor="end">SGLang high-throughput</text>
      <text class="sub" x="188" y="76" text-anchor="end">GLM-5.3 · peak of c1–c256</text>
      <rect x="196" y="56" width="147" height="14" rx="3" fill="var(--bad)"/>
      <text class="val" x="351" y="67">6,500</text>
      <text class="lab" x="188" y="92" text-anchor="end">vLLM</text>
      <text class="sub" x="188" y="102" text-anchor="end">Flash · peak of c1–c256</text>
      <rect x="196" y="82" width="115" height="14" rx="3" fill="var(--vl)"/>
      <text class="val" x="319" y="93">5,097</text>
      <text class="lab" x="188" y="118" text-anchor="end">SGLang low-latency</text>
      <text class="sub" x="188" y="128" text-anchor="end">Flash · peak of c1–c256</text>
      <rect x="196" y="108" width="94" height="14" rx="3" fill="var(--ok)"/>
      <text class="val" x="298" y="119">4,149</text>
      <text class="lab" x="188" y="144" text-anchor="end">vLLM</text>
      <text class="sub" x="188" y="154" text-anchor="end">GLM-5.3 · peak of c1–c256</text>
      <rect x="196" y="134" width="46" height="14" rx="3" fill="var(--vl)"/>
      <text class="val" x="250" y="145">2,014</text>
      <text class="lab" x="188" y="170" text-anchor="end">SGLang low-latency</text>
      <text class="sub" x="188" y="180" text-anchor="end">GLM-5.3 · peak of c1–c256</text>
      <rect x="196" y="160" width="37" height="14" rx="3" fill="var(--ok)"/>
      <text class="val" x="241" y="171">1,636</text>
  </svg>
  <figcaption>
    Peak sustained output on BATCH-D (4,096 in → 8,192 out), each configuration at
    whichever concurrency maximised it.
  </figcaption>
</figure>

The order inverts almost exactly. SGLang's high-throughput cell on Flash reaches
**14,210 tok/s**, 3.4× its own low-latency sibling and nine times what it managed
single-stream. The configuration that was last is now first by a factor of three.

## Where they trade places

<figure class="qz">
  <svg viewBox="0 0 640 318" role="img" aria-label="Output throughput against concurrency on BATCH-D for three GLM-5.3-Flash configurations. SGLang low-latency: 205 at c1, 2,158 at c16, 4,093 at c64, 4,149 at c256; SGLang high-throughput: 171 at c1, 1,638 at c16, 4,369 at c64, 14,210 at c256; vLLM: 205 at c1, 1,946 at c16, 5,025 at c64, 5,097 at c256.">
      <rect x="58" y="8" width="13" height="3" rx="1.5" fill="var(--ok)"/>
      <text class="sub" x="76" y="12.5">SGLang low-latency</text>
      <rect x="186" y="8" width="13" height="3" rx="1.5" fill="var(--bad)"/>
      <text class="sub" x="204" y="12.5">SGLang high-throughput</text>
      <rect x="337" y="8" width="13" height="3" rx="1.5" fill="var(--vl)"/>
      <text class="sub" x="355" y="12.5">vLLM</text>
      <line x1="58" y1="214" x2="566" y2="214" stroke="var(--grid)"/>
      <text class="sub" x="49" y="217" text-anchor="end">4k</text>
      <line x1="58" y1="155" x2="566" y2="155" stroke="var(--grid)"/>
      <text class="sub" x="49" y="159" text-anchor="end">8k</text>
      <line x1="58" y1="97" x2="566" y2="97" stroke="var(--grid)"/>
      <text class="sub" x="49" y="100" text-anchor="end">12k</text>
      <text class="sub" x="58" y="290" text-anchor="middle">1</text>
      <text class="sub" x="227" y="290" text-anchor="middle">16</text>
      <text class="sub" x="397" y="290" text-anchor="middle">64</text>
      <text class="sub" x="566" y="290" text-anchor="middle">256</text>
      <text class="ax" x="312" y="306" text-anchor="middle">concurrent requests</text>
      <polyline points="58,269 227,241 397,212 566,211" fill="none" stroke="var(--ok)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
      <circle cx="58" cy="269" r="3.5" fill="var(--ok)"/>
      <circle cx="227" cy="241" r="3.5" fill="var(--ok)"/>
      <circle cx="397" cy="212" r="3.5" fill="var(--ok)"/>
      <circle cx="566" cy="211" r="3.5" fill="var(--ok)"/>
      <polyline points="58,270 227,248 397,208 566,65" fill="none" stroke="var(--bad)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
      <circle cx="58" cy="270" r="3.5" fill="var(--bad)"/>
      <circle cx="227" cy="248" r="3.5" fill="var(--bad)"/>
      <circle cx="397" cy="208" r="3.5" fill="var(--bad)"/>
      <circle cx="566" cy="65" r="3.5" fill="var(--bad)"/>
      <polyline points="58,269 227,244 397,199 566,198" fill="none" stroke="var(--vl)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
      <circle cx="58" cy="269" r="3.5" fill="var(--vl)"/>
      <circle cx="227" cy="244" r="3.5" fill="var(--vl)"/>
      <circle cx="397" cy="199" r="3.5" fill="var(--vl)"/>
      <circle cx="566" cy="198" r="3.5" fill="var(--vl)"/>
      <text class="val" x="574" y="68" fill="var(--bad)">14,210</text>
      <text class="val" x="574" y="201" fill="var(--vl)">5,097</text>
      <text class="val" x="574" y="215" fill="var(--ok)">4,149</text>
  </svg>
  <figcaption>
    GLM-5.3-Flash on BATCH-D, the three configurations across the concurrency
    sweep. The lines cross between 16 and 64 concurrent requests.
  </figcaption>
</figure>

Below about 16 concurrent requests the low-latency cell leads. Above 64 the
high-throughput cell runs away with it. In between, they are within noise of each
other and vLLM is briefly ahead of both.

This is not a subtle effect to get wrong. If you are sizing a coding assistant for
a handful of engineers and you deploy the high-throughput recipe because it has
the bigger headline number, you will hand them a model that is **35% slower** than
the alternative. Deploy the low-latency recipe behind a batch pipeline and you
leave **70%** of the node's output on the floor.

## Why the low-latency cells stop scaling

They cap themselves. SGLang prints this at startup and then never exceeds it:

```
Max running requests is reset to 48 for speculative decoding.
You can override this by explicitly setting --max-running-requests.
```

Speculative decoding needs scratch space proportional to the number of in-flight
sequences, so the scheduler trades concurrency for the draft. That is why both
low-latency configurations actually *regress* from 64 to 256 offered concurrency —
the extra 208 requests only queue. Any figure above 48 for those cells is measured
against that ceiling, not against the hardware.

## Every workload, at a common operating point

Aggregate output tok/s at 64 concurrent requests. `LL` is the low-latency recipe,
`HT` high-throughput.

| workload | in → out | Flash LL | Flash HT | Flash vLLM | 5.3 LL | 5.3 HT | 5.3 vLLM |
|---|--:|--:|--:|--:|--:|--:|--:|
| API-S | 2,048 → 256 | 759 | **1,529** | 1,193 | 891 | 1,090 | 760 |
| CHAT-S | 2,048 → 512 | 1,675 | **2,840** | 2,038 | 1,139 | 1,521 | 1,238 |
| CODE-I | 16,384 → 2,048 | 1,608 | 1,873 | **2,003** | 616 | 930 | 569 |
| CHAT-L | 32,768 → 2,048 | 964 | 1,248 | **1,305** | 354 | 468 | 320 |
| CODE-A | 65,536 → 4,096 | 957 | 868 | **1,225** | 289 | 392 | 300 |
| DOC-L | 131,072 → 2,048 | 264 | 222 | **386** | 94 | *rejected* | 95 |
| BATCH-D | 4,096 → 8,192 | 4,093 | 4,369 | **5,025** | 1,516 | 4,009 | 2,014 |

At this particular operating point vLLM leads on five of seven shapes — but 64 is
close to the crossover, and by 256 concurrent requests SGLang's high-throughput
cell is 53% ahead on CHAT-S and 2.8× ahead on BATCH-D. A single-concurrency
comparison of two engines is not a comparison of two engines.

GLM-5.3 costs roughly two to three times the throughput of GLM-5.3-Flash for 2.3×
the parameters — 743B against 321B, 704 GB of weights against 306 GB.

## Three things that were not about speed

**NVFP4 does not run on this hardware.** I had intended to measure FP8 against
NVFP4. These H200s report compute capability 9.0 and NVFP4 needs a Blackwell FP4
tensor core, which is 10.0 and up. Both SGLang cookbooks publish NVFP4 cells only
for B200, B300, GB200 and GB300; both vLLM recipes say Blackwell-only outright.
Half the intended matrix needs different silicon.

**The GLM-5.3 high-throughput recipe cannot read a long document.** Every one of
460 DOC-L requests was rejected:

```
Input length (131084 tokens) exceeds the maximum allowed length (111546 tokens).
```

`--dp 8 --enable-dp-attention` replicates the KV pool per data-parallel rank,
cutting it to 111,552 tokens against the low-latency recipe's 224,768. The
low-latency recipe serves the same workload without complaint. GLM-5.3 declares a
1M-token context window; on this node, in this configuration, you get about a
tenth of it.

**The published vLLM command for GLM-5.3 does not start.** vLLM refuses to boot
unless one request's worth of KV cache fits, which for the full 1M window is
53.45 GiB — and 704 GB of weights leave 23.2 GiB. Adding `--max-model-len 262144`
is what made it run. That is still well above anything I measured, so it
constrains none of these numbers, but the recipe as published is missing it.

## The metric that nearly made this post wrong

GuideLLM reports `output_tokens_per_second` averaged over **completed** requests.
On BATCH-D at 256 concurrent streams, each request generating 8,192 tokens, only
2 of 257 requests finish inside the measurement window. That metric duly reported
**53 tok/s** while the engine was sustaining about **5,100**.

Every number here is instead the aggregate — all output tokens produced, including
by requests still streaming when the clock stopped, divided by the measured
window. It is monotonic in concurrency and it agrees with what the engine says
about itself. If you are benchmarking long-output workloads, check which of the
two your tool is giving you.

## Method

Single node, 8× NVIDIA H200 (SM90), TP8 throughout, FP8 weights. Flags copied
verbatim from the published `hw=h200` cells — the SGLang cookbook and the vLLM
recipes — with the one deviation noted above. GuideLLM 0.7.1, seven workload
shapes from 2,048 to 131,072 input tokens, runs bounded by duration rather than
prompt count. 78 runs in total.

Every configuration was checked for silent Triton `w8a8_block_fp8_matmul`
fallbacks, which are the usual cause of a quietly bad serving measurement. All six
came back clean. One repetition per cell, so there is no run-to-run spread to
quote — treat differences under about 10% as unresolved.
