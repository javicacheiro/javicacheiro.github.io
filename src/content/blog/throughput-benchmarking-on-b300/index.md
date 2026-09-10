---
title: 'Serving LLMs on a single 8×B300 server'
description: 'Measuring the throughput of five large models on one 8xB300 server.'
publishDate: 2026-09-10
draft: false
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

Lately, I have been playing a lot with really large local models for different use cases where it is very important to keep full control of the data.

There are really good open-weight models right now that offer state-of-the-art performance, but to run them you need really powerful GPUs with a lot of VRAM.

One of the most versatile nodes for this task, as well as for training, is the B300 node: eight NVIDIA B300 GPUs, each one with 268 GiB of VRAM, which means 2.1 TB of VRAM in a single node.

This high amount of VRAM in just one node is one of the reasons why B300 nodes are in such high demand nowadays.

If you compare with an equivalent B200 node (the next GPU), it has 1.4 TB of VRAM, so roughly 700 GB less.

The great advantage of having 2 TB of VRAM is that with just one node you can run some of the frontier open-weight models right now without having to enter the wonderful world of multi-node inference and the fun of debugging the interconnection when you do not get the expected throughput (I can talk about that in a different post).

To measure how much one of these nodes can offer, I benchmarked five large models to see how much performance we can get for local inference.

## Models tested

Parameter counts complicate this comparison because several of the models that I benchmarked are mixture-of-experts, and published counts mix total and active parameters. So instead of the number from the model card, I use the measured VRAM consumption from **what vLLM reported during loading**, summed across the eight GPUs:

<figure class="qz">
  <svg viewBox="0 0 640 201" role="img" aria-label="Weight footprint. Kimi-K3 1,532 GiB, GLM-5.2 BF16 1,412 GiB, Qwen3.8-2.4T 1,340 GiB, DeepSeek-V4-Pro 850 GiB, GLM-5.2 FP8 714 GiB.">
    <line x1="279" y1="22" x2="279" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="279" y="18" text-anchor="middle">500 GiB</text>
    <line x1="402" y1="22" x2="402" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="402" y="18" text-anchor="middle">1,000 GiB</text>
    <line x1="525" y1="22" x2="525" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="525" y="18" text-anchor="middle">1,500 GiB</text>
    <text class="ax" x="156" y="195" text-anchor="start">Memory usage: Weight footprint</text>
    <text class="lab" x="148" y="45" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="56" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="34" width="377" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="541" y="46">1,532 GiB</text>
    <text class="lab" x="148" y="72" text-anchor="end">GLM-5.2 BF16</text>
    <text class="sub" x="148" y="83" text-anchor="end">BF16 · 1,412 GiB</text>
    <rect x="156" y="61" width="348" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="512" y="73">1,412 GiB</text>
    <text class="lab" x="148" y="99" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="110" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="88" width="330" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="494" y="100">1,340 GiB</text>
    <text class="lab" x="148" y="126" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="137" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="115" width="209" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="373" y="127">850 GiB</text>
    <text class="lab" x="148" y="153" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="164" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="142" width="176" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="340" y="154">714 GiB</text>
  </svg>
  <figcaption>
    Memory usage for the model weights calculated from vLLM's log.
    It already accounts for quantization, and it is what determines how much space is left for the KV cache.
  </figcaption>
</figure>

## Workloads

Using the great [GuideLLM](https://github.com/vllm-project/guidellm) framework I created seven different workloads, representing different types of usage.
They differ only in how many tokens go in and how many come out, but those numbers are very important because they affect performance a lot.

| workload | in → out | use case description |
|---|--:|---|
| **CHAT-S** | 2,048 → 512 | Ordinary interactive chat: short assistant turns. Latency-dominated. |
| **CHAT-L** | 32,768 → 2,048 | A long technical conversation with accumulated history and detailed answers. |
| **CODE-I** | 16,384 → 2,048 | Interactive coding assistant: explain this, fix this, review this. |
| **CODE-A** | 65,536 → 4,096 | A coding-agent call with a repository in context. |
| **API-S** | 2,048 → 256 | Short structured request: extraction, routing, JSON generation, metadata tagging. High request rate, no reasoning. |
| **DOC-L** | 131,072 → 2,048 | Long-document analysis and RAG: reports, logs, retrieved context. Prefill-dominated. |
| **BATCH-D** | 4,096 → 8,192 | Decode-heavy batch generation. Approximates maximum sustained output capacity. |

**BATCH-D** runs with reasoning disabled and forces `ignore_eos` so the model cannot stop early, which makes it a maximum throughput probe.
**API-S** also runs with reasoning disabled; the other five have reasoning enabled.

## Results

The metric reported here is <code>server_output_tps</code>, as reported by vLLM.

Let's start with the raw ranking using the BATCH-D workload (the one where we can get the most throughput):

<figure class="qz">
  <svg viewBox="0 0 640 201" role="img" aria-label="Peak batch throughput. DeepSeek-V4-Pro 9,183, GLM-5.2 FP8 7,204, Qwen3.8-2.4T 6,168, Kimi-K3 4,422, GLM-5.2 BF16 4,360.">
    <line x1="259" y1="22" x2="259" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="259" y="18" text-anchor="middle">2,500</text>
    <line x1="361" y1="22" x2="361" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="361" y="18" text-anchor="middle">5,000</text>
    <line x1="464" y1="22" x2="464" y2="161" stroke="var(--grid)"/>
    <text class="sub" x="464" y="18" text-anchor="middle">7,500</text>
    <text class="ax" x="156" y="195" text-anchor="start">Peak batch throughput — BATCH-D, best concurrency per model</text>
    <text class="lab" x="148" y="45" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="56" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="34" width="377" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="541" y="46">9,183</text>
    <text class="lab" x="148" y="72" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="83" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="61" width="296" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="460" y="73">7,204</text>
    <text class="lab" x="148" y="99" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="110" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="88" width="253" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="417" y="100">6,168</text>
    <text class="lab" x="148" y="126" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="137" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="115" width="182" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="346" y="127">4,422</text>
    <text class="lab" x="148" y="153" text-anchor="end">GLM-5.2 BF16</text>
    <text class="sub" x="148" y="164" text-anchor="end">BF16 · 1,412 GiB</text>
    <rect x="156" y="142" width="179" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="343" y="154">4,360</text>
  </svg>
  <figcaption>
    Peak sustained output on BATCH-D, each model at the concurrency that
    maximised it. Headline metric is <code>server_output_tps</code> as reported
    by vLLM — what an operator sees on their own dashboard.
  </figcaption>
</figure>

**DeepSeek-V4-Pro leads at 9,183 tok/s**, 27% ahead of GLM-5.2 FP8 even though it is 19% larger.

Kimi-K3 and GLM-5.2 BF16 are very close, but **the great surprise is Qwen3.8-2.4T at 6,168 tok/s**, which is of a similar size to Kimi-K3 and GLM-5.2 BF16 but performs much faster.

## Comparing within size tiers

Even though they are all state-of-the-art models, the difference in size is important, so it is fairer to compare them by establishing two tiers, mid and large, depending on the size:

| tier | model | footprint | BATCH-D | CHAT-S |
|---|---|--:|--:|--:|
| **mid** | **DeepSeek-V4-Pro** | 850 GiB | **9,183** | **5,493** |
| | GLM-5.2 FP8 | 714 GiB | 7,204 | 3,989 |
| **large** | **Qwen3.8-2.4T** | 1,340 GiB | **6,168** | **4,115** |
| | Kimi-K3 | 1,532 GiB | 4,422 | 2,880 |
| | GLM-5.2 BF16 | 1,412 GiB | 4,360 | 3,658 |

Read this way:

- **In the mid tier, DeepSeek-V4-Pro beats GLM-5.2 FP8 by 27%** on batch
  throughput while being 19% *larger*.
- **In the large tier, Qwen3.8-2.4T leads by ~40%** over both Kimi-K3 and
  GLM-5.2 BF16, despite sitting between them in footprint.

## Individual workload results

Here are the individual results for the seven workloads, with the concurrency numbers (how many clients run at the same time) reported to the right of each bar:

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="API-S peak output throughput, short structured API request. DeepSeek-V4-Pro 3,759, GLM-5.2 FP8 2,684, GLM-5.2 BF16 2,617, Qwen3.8-2.4T 2,443, Kimi-K3 1,651.">
    <line x1="346" y1="14" x2="346" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="346" y="10" text-anchor="middle">2,000</text>
    <line x1="535" y1="14" x2="535" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="535" y="10" text-anchor="middle">4,000</text>
    <text class="ax" x="156" y="169" text-anchor="start">API-S · 2,048 → 256 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="35">3,759</text>
    <text class="sub" x="569" y="35">c512</text>
    <text class="lab" x="148" y="58" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="48" width="254" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="418" y="59">2,684</text>
    <text class="sub" x="467" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">GLM-5.2 BF16</text>
    <text class="sub" x="148" y="92" text-anchor="end">BF16 · 1,412 GiB</text>
    <rect x="156" y="72" width="248" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="412" y="83">2,617</text>
    <text class="sub" x="461" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="116" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="96" width="232" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="396" y="107">2,443</text>
    <text class="sub" x="445" y="107">c512</text>
    <text class="lab" x="148" y="130" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="140" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="120" width="157" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="321" y="131">1,651</text>
    <text class="sub" x="370" y="131">c256</text>
  </svg>
  <figcaption>
    <strong>API-S — 2,048 in → 256 out.</strong> Short structured requests:
    extraction, routing, JSON generation, metadata. Reasoning off.
    DeepSeek-V4-Pro and Qwen3.8-2.4T
    both peak at 512 concurrent requests, the other three at 256.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="CHAT-S peak output throughput, ordinary interactive chat. DeepSeek-V4-Pro 5,493, Qwen3.8-2.4T 4,115, GLM-5.2 FP8 3,989, GLM-5.2 BF16 3,658, Kimi-K3 2,880.">
    <line x1="286" y1="14" x2="286" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="286" y="10" text-anchor="middle">2,000</text>
    <line x1="416" y1="14" x2="416" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="416" y="10" text-anchor="middle">4,000</text>
    <line x1="545" y1="14" x2="545" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="545" y="10" text-anchor="middle">6,000</text>
    <text class="ax" x="156" y="169" text-anchor="start">CHAT-S · 2,048 → 512 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="35">5,493</text>
    <text class="sub" x="569" y="35">c512</text>
    <text class="lab" x="148" y="58" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="68" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="48" width="267" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="431" y="59">4,115</text>
    <text class="sub" x="480" y="59">c512</text>
    <text class="lab" x="148" y="82" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="92" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="72" width="259" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="423" y="83">3,989</text>
    <text class="sub" x="472" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 BF16</text>
    <text class="sub" x="148" y="116" text-anchor="end">BF16 · 1,412 GiB</text>
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
    The most common use case.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="CODE-I peak output throughput, interactive coding assistant. DeepSeek-V4-Pro 2,335, Qwen3.8-2.4T 2,004, GLM-5.2 FP8 1,814, GLM-5.2 BF16 1,458, Kimi-K3 1,390.">
    <line x1="309" y1="14" x2="309" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="309" y="10" text-anchor="middle">1,000</text>
    <line x1="461" y1="14" x2="461" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="461" y="10" text-anchor="middle">2,000</text>
    <text class="ax" x="156" y="169" text-anchor="start">CODE-I · 16,384 → 2,048 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="35">2,335</text>
    <text class="sub" x="569" y="35">c256</text>
    <text class="lab" x="148" y="58" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="68" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="48" width="306" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="470" y="59">2,004</text>
    <text class="sub" x="519" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="92" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="72" width="277" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="441" y="83">1,814</text>
    <text class="sub" x="490" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 BF16</text>
    <text class="sub" x="148" y="116" text-anchor="end">BF16 · 1,412 GiB</text>
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
    assistant: explain this function, make this edit, review this diff.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="CHAT-L peak output throughput, long technical conversation. DeepSeek-V4-Pro 1,535, GLM-5.2 FP8 1,071, Qwen3.8-2.4T 1,039, GLM-5.2 BF16 855, Kimi-K3 774.">
    <line x1="272" y1="14" x2="272" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="272" y="10" text-anchor="middle">500</text>
    <line x1="388" y1="14" x2="388" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="388" y="10" text-anchor="middle">1,000</text>
    <line x1="504" y1="14" x2="504" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="504" y="10" text-anchor="middle">1,500</text>
    <text class="ax" x="156" y="169" text-anchor="start">CHAT-L · 32,768 → 2,048 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="35">1,535</text>
    <text class="sub" x="569" y="35">c256</text>
    <text class="lab" x="148" y="58" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="48" width="249" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="413" y="59">1,071</text>
    <text class="sub" x="462" y="59">c64</text>
    <text class="lab" x="148" y="82" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="92" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="72" width="241" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="405" y="83">1,039</text>
    <text class="sub" x="454" y="83">c256</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 BF16</text>
    <text class="sub" x="148" y="116" text-anchor="end">BF16 · 1,412 GiB</text>
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
    conversation carrying accumulated history.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="CODE-A peak output throughput, coding-agent call, repository in context. DeepSeek-V4-Pro 1,328, Qwen3.8-2.4T 997, GLM-5.2 FP8 839, GLM-5.2 BF16 719, Kimi-K3 691.">
    <line x1="290" y1="14" x2="290" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="290" y="10" text-anchor="middle">500</text>
    <line x1="424" y1="14" x2="424" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="424" y="10" text-anchor="middle">1,000</text>
    <text class="ax" x="156" y="169" text-anchor="start">CODE-A · 65,536 → 4,096 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="35">1,328</text>
    <text class="sub" x="569" y="35">c256</text>
    <text class="lab" x="148" y="58" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="68" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="48" width="268" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="432" y="59">997</text>
    <text class="sub" x="465" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="92" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="72" width="225" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="389" y="83">839</text>
    <text class="sub" x="422" y="83">c64</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 BF16</text>
    <text class="sub" x="148" y="116" text-anchor="end">BF16 · 1,412 GiB</text>
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
    with a repository in context.
    Real agents will issue many of these requests.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="DOC-L peak output throughput, long-document analysis and RAG. DeepSeek-V4-Pro 384, Qwen3.8-2.4T 293, GLM-5.2 BF16 260, GLM-5.2 FP8 258, Kimi-K3 212.">
    <line x1="341" y1="14" x2="341" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="341" y="10" text-anchor="middle">200</text>
    <line x1="527" y1="14" x2="527" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="527" y="10" text-anchor="middle">400</text>
    <text class="ax" x="156" y="169" text-anchor="start">DOC-L · 131,072 → 2,048 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="35">384</text>
    <text class="sub" x="553" y="35">c256</text>
    <text class="lab" x="148" y="58" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="68" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="48" width="272" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="436" y="59">293</text>
    <text class="sub" x="469" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">GLM-5.2 BF16</text>
    <text class="sub" x="148" y="92" text-anchor="end">BF16 · 1,412 GiB</text>
    <rect x="156" y="72" width="241" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="405" y="83">260</text>
    <text class="sub" x="438" y="83">c96</text>
    <text class="lab" x="148" y="106" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="116" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="96" width="239" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="403" y="107">258</text>
    <text class="sub" x="436" y="107">c16</text>
    <text class="lab" x="148" y="130" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="140" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="120" width="197" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="361" y="131">212</text>
    <text class="sub" x="394" y="131">c256</text>
  </svg>
  <figcaption>
    <strong>DOC-L — 131,072 in → 2,048 out.</strong> Long-document analysis
    and RAG.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 174" role="img" aria-label="BATCH-D peak output throughput, decode-heavy batch generation. DeepSeek-V4-Pro 9,183, GLM-5.2 FP8 7,204, Qwen3.8-2.4T 6,168, Kimi-K3 4,422, GLM-5.2 BF16 4,360.">
    <line x1="350" y1="14" x2="350" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="350" y="10" text-anchor="middle">5,000</text>
    <line x1="544" y1="14" x2="544" y2="137" stroke="var(--grid)"/>
    <text class="sub" x="544" y="10" text-anchor="middle">10,000</text>
    <text class="ax" x="156" y="169" text-anchor="start">BATCH-D · 4,096 → 8,192 · peak output tok/s</text>
    <text class="lab" x="148" y="34" text-anchor="end">DeepSeek-V4-Pro</text>
    <text class="sub" x="148" y="44" text-anchor="end">FP8 · 850 GiB</text>
    <rect x="156" y="24" width="356" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="520" y="35">9,183</text>
    <text class="sub" x="569" y="35">c512</text>
    <text class="lab" x="148" y="58" text-anchor="end">GLM-5.2 FP8</text>
    <text class="sub" x="148" y="68" text-anchor="end">FP8 · 714 GiB</text>
    <rect x="156" y="48" width="280" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="444" y="59">7,204</text>
    <text class="sub" x="493" y="59">c256</text>
    <text class="lab" x="148" y="82" text-anchor="end">Qwen3.8-2.4T</text>
    <text class="sub" x="148" y="92" text-anchor="end">NVFP4 · 1,340 GiB</text>
    <rect x="156" y="72" width="239" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="403" y="83">6,168</text>
    <text class="sub" x="452" y="83">c512</text>
    <text class="lab" x="148" y="106" text-anchor="end">Kimi-K3</text>
    <text class="sub" x="148" y="116" text-anchor="end">MXFP4 · 1,532 GiB</text>
    <rect x="156" y="96" width="172" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="336" y="107">4,422</text>
    <text class="sub" x="385" y="107">c1024</text>
    <text class="lab" x="148" y="130" text-anchor="end">GLM-5.2 BF16</text>
    <text class="sub" x="148" y="140" text-anchor="end">BF16 · 1,412 GiB</text>
    <rect x="156" y="120" width="169" height="13" rx="3" fill="var(--ok)"/>
    <text class="val" x="333" y="131">4,360</text>
    <text class="sub" x="382" y="131">c256</text>
  </svg>
  <figcaption>
    <strong>BATCH-D — 4,096 in → 8,192 out.</strong> Decode-heavy batch
    generation with EOS suppressed, so the model cannot stop early.
    It allows us to measure peak performance.
  </figcaption>
</figure>

## Latency

In this benchmark we focus mainly on throughput, but there is another important aspect to measure: latency (how long you have to wait for the first token to appear).

So I measured time-to-first-token p95 at a fixed load of 256 concurrent
requests. It spreads over a factor of **2.4** across the five models, against a
factor of 1.7 in throughput at the same level of concurrency.

| model | TTFT p95 @ c256 | tok/s @ c256 |
|---|--:|--:|
| DeepSeek-V4-Pro | 9,852 ms | 4,237 |
| GLM-5.2 BF16 | 13,155 ms | 3,658 |
| GLM-5.2 FP8 | 13,214 ms | 3,989 |
| Qwen3.8-2.4T | 16,394 ms | 3,241 |
| Kimi-K3 | 23,918 ms | 2,460 |

Kimi-K3 at 23.9 seconds to first token is by far the slowest, so you have to balance throughput against latency.

## DeepSeek-V4-Flash

Everything above compares five models between 714 GiB and 1,532 GiB. But just for measuring how much performance we can get on this server with a more modest model,
I also measured **DeepSeek-V4-Flash**, which is much smaller: 170 GiB.

<figure class="qz">
  <svg viewBox="0 0 640 242" role="img" aria-label="DeepSeek-V4-Flash peak output throughput across all seven workloads. BATCH-D 15,484, CHAT-S 9,010, API-S 6,404, CODE-I 4,825, CHAT-L 2,482, CODE-A 2,471, DOC-L 634.">
    <line x1="265" y1="20" x2="265" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="265" y="16" text-anchor="middle">5,000</text>
    <line x1="379" y1="20" x2="379" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="379" y="16" text-anchor="middle">10,000</text>
    <line x1="494" y1="20" x2="494" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="494" y="16" text-anchor="middle">15,000</text>
    <text class="ax" x="150" y="237" text-anchor="start">DeepSeek-V4-Flash · peak output tok/s by workload</text>
    <text class="lab" x="142" y="41" text-anchor="end">BATCH-D</text>
    <text class="sub" x="142" y="51" text-anchor="end">4,096→8,192</text>
    <rect x="150" y="30" width="355" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="513" y="42">15,484</text>
    <text class="sub" x="570" y="42">c512</text>
    <text class="lab" x="142" y="67" text-anchor="end">CHAT-S</text>
    <text class="sub" x="142" y="77" text-anchor="end">2,048→512</text>
    <rect x="150" y="56" width="207" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="365" y="68">9,010</text>
    <text class="sub" x="414" y="68">c512</text>
    <text class="lab" x="142" y="93" text-anchor="end">API-S</text>
    <text class="sub" x="142" y="103" text-anchor="end">2,048→256</text>
    <rect x="150" y="82" width="147" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="305" y="94">6,404</text>
    <text class="sub" x="354" y="94">c1024</text>
    <text class="lab" x="142" y="119" text-anchor="end">CODE-I</text>
    <text class="sub" x="142" y="129" text-anchor="end">16,384→2,048</text>
    <rect x="150" y="108" width="111" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="269" y="120">4,825</text>
    <text class="sub" x="318" y="120">c256</text>
    <text class="lab" x="142" y="145" text-anchor="end">CHAT-L</text>
    <text class="sub" x="142" y="155" text-anchor="end">32,768→2,048</text>
    <rect x="150" y="134" width="57" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="215" y="146">2,482</text>
    <text class="sub" x="264" y="146">c256</text>
    <text class="lab" x="142" y="171" text-anchor="end">CODE-A</text>
    <text class="sub" x="142" y="181" text-anchor="end">65,536→4,096</text>
    <rect x="150" y="160" width="57" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="215" y="172">2,471</text>
    <text class="sub" x="264" y="172">c256</text>
    <text class="lab" x="142" y="197" text-anchor="end">DOC-L</text>
    <text class="sub" x="142" y="207" text-anchor="end">131,072→2,048</text>
    <rect x="150" y="186" width="15" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="173" y="198">634</text>
    <text class="sub" x="206" y="198">c64</text>
  </svg>
  <figcaption>
    DeepSeek-V4-Flash across all seven workloads, peak output tok/s with the
    concurrency that produced it. MXFP4 weights, speculative decoding disabled.
  </figcaption>
</figure>

## Conclusions

With just one B300 node we can serve lots of concurrent requests, even for state-of-the-art models.
