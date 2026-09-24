---
title: 'Serving LLMs on a single 8×H200 server'
description: 'Measuring the throughput of five large models on one 8xH200 server.'
publishDate: 2026-09-24
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

Comparison of the performance of four medium-size models (GLM-5.3, GLM-5.3-Flash, DeepSeek-V4.1-Flash and Qwen3.8-Flash-Next)
on a H200 node with 8 GPUs, measured on the same seven
workload shapes I used for the [B300 comparison](/blog/throughput-benchmarking-on-b300/).

## Method

One node with 8× NVIDIA H200 (SM90, 141 GiB each). TP8 throughout except Qwen's
SGLang cells, which are TP4. The flags used correspond to the ones from each model's published
recipe in vLLM and SGlang docs. There are only three forced deviations: `--max-model-len 262144` for
GLM-5.3 under vLLM (the published command will not boot otherwise),
`--max-num-seqs 512` for GLM-5.3-Flash under data parallelism, and a raised
engine-start timeout.

GuideLLM 0.7.1, with seven workload shapes from 2,048 to 131,072 input tokens, runs
bounded by duration rather than prompt count. Same workloads as the ones used for the [B300 comparison](/blog/throughput-benchmarking-on-b300/).

The 256-stream runs are provisional due to some issues in the benchmarking procedure.

## Single request performance

<figure class="qz">
  <svg viewBox="0 0 640 528" role="img" aria-label="Single-stream output throughput for six serving configurations. vLLM latency 294, SGLang low-latency 195, SGLang low-latency 184, SGLang low-latency 174, vLLM throughput 167, SGLang low-latency 164, vLLM latency 150, vLLM latency 147, vLLM balanced 143, vLLM balanced 140, vLLM balanced 137, SGLang high-throughput 137, SGLang high-throughput 137, SGLang high-throughput 113, vLLM throughput 99, SGLang high-throughput 96, SGLang low-latency 68, SGLang high-throughput 68.">
    <line x1="306" y1="20" x2="306" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="306" y="16" text-anchor="middle">100</text>
    <line x1="415" y1="20" x2="415" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="415" y="16" text-anchor="middle">200</text>
    <line x1="525" y1="20" x2="525" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="525" y="16" text-anchor="middle">300</text>
    <text class="ax" x="196" y="523" text-anchor="start">One request at a time · CHAT-S · output tok/s</text>
    <text class="lab" x="188" y="40" text-anchor="end">vLLM latency</text>
    <text class="sub" x="188" y="50" text-anchor="end">DeepSeek-V4.1 · 3.32 ms / token</text>
    <rect x="196" y="30" width="321" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="525" y="41">294</text>
    <text class="lab" x="188" y="66" text-anchor="end">SGLang low-latency</text>
    <text class="sub" x="188" y="76" text-anchor="end">GLM-5.3 · 4.71 ms / token</text>
    <rect x="196" y="56" width="213" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="417" y="67">195</text>
    <text class="lab" x="188" y="92" text-anchor="end">SGLang low-latency</text>
    <text class="sub" x="188" y="102" text-anchor="end">Qwen3.8 FP8 · 5.03 ms / token</text>
    <rect x="196" y="82" width="202" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="406" y="93">184</text>
    <text class="lab" x="188" y="118" text-anchor="end">SGLang low-latency</text>
    <text class="sub" x="188" y="128" text-anchor="end">GLM-5.3-Flash · 5.31 ms / token</text>
    <rect x="196" y="108" width="191" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="395" y="119">174</text>
    <text class="lab" x="188" y="144" text-anchor="end">vLLM throughput</text>
    <text class="sub" x="188" y="154" text-anchor="end">DeepSeek-V4.1 · 5.68 ms / token</text>
    <rect x="196" y="134" width="183" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="387" y="145">167</text>
    <text class="lab" x="188" y="170" text-anchor="end">SGLang low-latency</text>
    <text class="sub" x="188" y="180" text-anchor="end">Qwen3.8 bf16 · 5.76 ms / token</text>
    <rect x="196" y="160" width="179" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="383" y="171">164</text>
    <text class="lab" x="188" y="196" text-anchor="end">vLLM latency</text>
    <text class="sub" x="188" y="206" text-anchor="end">GLM-5.3-Flash · 6.10 ms / token</text>
    <rect x="196" y="186" width="164" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="368" y="197">150</text>
    <text class="lab" x="188" y="222" text-anchor="end">vLLM latency</text>
    <text class="sub" x="188" y="232" text-anchor="end">GLM-5.3 · 6.41 ms / token</text>
    <rect x="196" y="212" width="161" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="365" y="223">147</text>
    <text class="lab" x="188" y="248" text-anchor="end">vLLM balanced</text>
    <text class="sub" x="188" y="258" text-anchor="end">GLM-5.3-Flash · 6.44 ms / token</text>
    <rect x="196" y="238" width="157" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="361" y="249">143</text>
    <text class="lab" x="188" y="274" text-anchor="end">vLLM balanced</text>
    <text class="sub" x="188" y="284" text-anchor="end">Qwen3.8 FP8 · 7.02 ms / token</text>
    <rect x="196" y="264" width="153" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="357" y="275">140</text>
    <text class="lab" x="188" y="300" text-anchor="end">vLLM balanced</text>
    <text class="sub" x="188" y="310" text-anchor="end">GLM-5.3 · 7.02 ms / token</text>
    <rect x="196" y="290" width="150" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="354" y="301">137</text>
    <text class="lab" x="188" y="326" text-anchor="end">SGLang high-throughput</text>
    <text class="sub" x="188" y="336" text-anchor="end">Qwen3.8 bf16 · 7.16 ms / token</text>
    <rect x="196" y="316" width="150" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="354" y="327">137</text>
    <text class="lab" x="188" y="352" text-anchor="end">SGLang high-throughput</text>
    <text class="sub" x="188" y="362" text-anchor="end">Qwen3.8 FP8 · 7.07 ms / token</text>
    <rect x="196" y="342" width="150" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="354" y="353">137</text>
    <text class="lab" x="188" y="378" text-anchor="end">SGLang high-throughput</text>
    <text class="sub" x="188" y="388" text-anchor="end">GLM-5.3-Flash · 8.64 ms / token</text>
    <rect x="196" y="368" width="123" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="327" y="379">113</text>
    <text class="lab" x="188" y="404" text-anchor="end">vLLM throughput</text>
    <text class="sub" x="188" y="414" text-anchor="end">GLM-5.3-Flash · 9.95 ms / token</text>
    <rect x="196" y="394" width="108" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="312" y="405">99</text>
    <text class="lab" x="188" y="430" text-anchor="end">SGLang high-throughput</text>
    <text class="sub" x="188" y="440" text-anchor="end">GLM-5.3 · 10.21 ms / token</text>
    <rect x="196" y="420" width="105" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="309" y="431">96</text>
    <text class="lab" x="188" y="456" text-anchor="end">SGLang low-latency</text>
    <text class="sub" x="188" y="466" text-anchor="end">DeepSeek-V4.1 · 14.97 ms / token</text>
    <rect x="196" y="446" width="75" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="279" y="457">68</text>
    <text class="lab" x="188" y="482" text-anchor="end">SGLang high-throughput</text>
    <text class="sub" x="188" y="492" text-anchor="end">DeepSeek-V4.1 · 15.00 ms / token</text>
    <rect x="196" y="472" width="75" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="279" y="483">68</text>
  </svg>
  <figcaption>
    A single request, CHAT-S shape (2,048 in → 512 out). The sublabel is the
    inter-token latency — how long a reader waits between words.
  </figcaption>
</figure>

DeepSeek-V4.1-Flash under vLLM is the fastest single stream in the set at
**294 tok/s and 3.3 ms between tokens** — and the same model under SGLang's
published H200 cell is the *slowest* at 68 tok/s. More on that below, because it
is the one result here that is not a tuning trade-off.

Otherwise the pattern is the one you would expect: low-latency recipes on top,
high-throughput recipes at the bottom, and the largest model (GLM-5.3, 743B)
holding its own at 195 tok/s because its EAGLE 5-step draft is nearly free at
batch 1.

## Concurrent request performance

<figure class="qz">
  <svg viewBox="0 0 640 528" role="img" aria-label="Peak batch output throughput for six serving configurations. SGLang high-throughput 14,210, vLLM throughput 11,805, vLLM throughput 10,490, SGLang high-throughput 8,737, SGLang low-latency 8,737, SGLang high-throughput 8,412, vLLM balanced 8,412, SGLang high-throughput 8,176, SGLang high-throughput 6,500, vLLM balanced 5,787, vLLM latency 5,343, SGLang low-latency 5,219, vLLM latency 5,097, SGLang low-latency 4,457, SGLang low-latency 4,149, vLLM balanced 2,051, vLLM latency 2,014, SGLang low-latency 1,636.">
    <line x1="309" y1="20" x2="309" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="309" y="16" text-anchor="middle">5,000</text>
    <line x1="422" y1="20" x2="422" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="422" y="16" text-anchor="middle">10,000</text>
    <line x1="535" y1="20" x2="535" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="535" y="16" text-anchor="middle">15,000</text>
    <text class="ax" x="196" y="523" text-anchor="start">Peak under load · BATCH-D · output tok/s</text>
    <text class="lab" x="188" y="40" text-anchor="end">SGLang high-throughput</text>
    <text class="sub" x="188" y="50" text-anchor="end">GLM-5.3-Flash · peak of c1–c256</text>
    <rect x="196" y="30" width="321" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="525" y="41">14,210</text>
    <text class="lab" x="188" y="66" text-anchor="end">vLLM throughput</text>
    <text class="sub" x="188" y="76" text-anchor="end">GLM-5.3-Flash · peak of c1–c256</text>
    <rect x="196" y="56" width="267" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="471" y="67">11,805</text>
    <text class="lab" x="188" y="92" text-anchor="end">vLLM throughput</text>
    <text class="sub" x="188" y="102" text-anchor="end">DeepSeek-V4.1 · peak of c1–c256</text>
    <rect x="196" y="82" width="237" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="441" y="93">10,490</text>
    <text class="lab" x="188" y="118" text-anchor="end">SGLang high-throughput</text>
    <text class="sub" x="188" y="128" text-anchor="end">DeepSeek-V4.1 · peak of c1–c256</text>
    <rect x="196" y="108" width="198" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="402" y="119">8,737</text>
    <text class="lab" x="188" y="144" text-anchor="end">SGLang low-latency</text>
    <text class="sub" x="188" y="154" text-anchor="end">DeepSeek-V4.1 · peak of c1–c256</text>
    <rect x="196" y="134" width="198" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="402" y="145">8,737</text>
    <text class="lab" x="188" y="170" text-anchor="end">SGLang high-throughput</text>
    <text class="sub" x="188" y="180" text-anchor="end">Qwen3.8 FP8 · peak of c1–c256</text>
    <rect x="196" y="160" width="190" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="394" y="171">8,412</text>
    <text class="lab" x="188" y="196" text-anchor="end">vLLM balanced</text>
    <text class="sub" x="188" y="206" text-anchor="end">Qwen3.8 FP8 · peak of c1–c256</text>
    <rect x="196" y="186" width="190" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="394" y="197">8,412</text>
    <text class="lab" x="188" y="222" text-anchor="end">SGLang high-throughput</text>
    <text class="sub" x="188" y="232" text-anchor="end">Qwen3.8 bf16 · peak of c1–c256</text>
    <rect x="196" y="212" width="185" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="389" y="223">8,176</text>
    <text class="lab" x="188" y="248" text-anchor="end">SGLang high-throughput</text>
    <text class="sub" x="188" y="258" text-anchor="end">GLM-5.3 · peak of c1–c256</text>
    <rect x="196" y="238" width="147" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="351" y="249">6,500</text>
    <text class="lab" x="188" y="274" text-anchor="end">vLLM balanced</text>
    <text class="sub" x="188" y="284" text-anchor="end">GLM-5.3-Flash · peak of c1–c256</text>
    <rect x="196" y="264" width="131" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="335" y="275">5,787</text>
    <text class="lab" x="188" y="300" text-anchor="end">vLLM latency</text>
    <text class="sub" x="188" y="310" text-anchor="end">DeepSeek-V4.1 · peak of c1–c256</text>
    <rect x="196" y="290" width="121" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="325" y="301">5,343</text>
    <text class="lab" x="188" y="326" text-anchor="end">SGLang low-latency</text>
    <text class="sub" x="188" y="336" text-anchor="end">Qwen3.8 bf16 · peak of c1–c256</text>
    <rect x="196" y="316" width="118" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="322" y="327">5,219</text>
    <text class="lab" x="188" y="352" text-anchor="end">vLLM latency</text>
    <text class="sub" x="188" y="362" text-anchor="end">GLM-5.3-Flash · peak of c1–c256</text>
    <rect x="196" y="342" width="115" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="319" y="353">5,097</text>
    <text class="lab" x="188" y="378" text-anchor="end">SGLang low-latency</text>
    <text class="sub" x="188" y="388" text-anchor="end">Qwen3.8 FP8 · peak of c1–c256</text>
    <rect x="196" y="368" width="101" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="305" y="379">4,457</text>
    <text class="lab" x="188" y="404" text-anchor="end">SGLang low-latency</text>
    <text class="sub" x="188" y="414" text-anchor="end">GLM-5.3-Flash · peak of c1–c256</text>
    <rect x="196" y="394" width="94" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="298" y="405">4,149</text>
    <text class="lab" x="188" y="430" text-anchor="end">vLLM balanced</text>
    <text class="sub" x="188" y="440" text-anchor="end">GLM-5.3 · peak of c1–c256</text>
    <rect x="196" y="420" width="46" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="250" y="431">2,051</text>
    <text class="lab" x="188" y="456" text-anchor="end">vLLM latency</text>
    <text class="sub" x="188" y="466" text-anchor="end">GLM-5.3 · peak of c1–c256</text>
    <rect x="196" y="446" width="46" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="250" y="457">2,014</text>
    <text class="lab" x="188" y="482" text-anchor="end">SGLang low-latency</text>
    <text class="sub" x="188" y="492" text-anchor="end">GLM-5.3 · peak of c1–c256</text>
    <rect x="196" y="472" width="37" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="241" y="483">1,636</text>
  </svg>
  <figcaption>
    Peak sustained output on BATCH-D (4,096 in → 8,192 out), each configuration at
    whichever concurrency maximised it.
  </figcaption>
</figure>

The order inverts almost completely. GLM-5.3-Flash on SGLang's high-throughput
cell tops the chart, and the low-latency recipes fall to the bottom.


## Complete results

Aggregate output tok/s at 64 concurrent requests — the operating point where a
second repetition agreed within 6%, so these are the soundest numbers here.

- LL: Low latency tuning
- HF: High througput tuning

| configuration | API-S | CHAT-S | CODE-I | CHAT-L | CODE-A | DOC-L | BATCH-D |
|---|--:|--:|--:|--:|--:|--:|--:|
| **GLM-5.3-Flash** | | | | | | | |
| SGLang LL | 759 | 1,675 | 1,608 | 964 | 957 | 264 | 4,093 |
| SGLang HT | 1,529 | 2,840 | 1,873 | 1,248 | 868 | 222 | 4,369 |
| vLLM latency | 1,193 | 2,038 | 2,003 | 1,305 | 1,225 | 386 | 5,025 |
| vLLM balanced | 1,257 | 2,238 | 2,097 | 1,308 | 1,216 | 388 | 5,433 |
| vLLM throughput | 1,478 | 2,091 | 1,845 | 1,317 | 1,699 | 587 | 4,490 |
| **GLM-5.3** | | | | | | | |
| SGLang LL | 891 | 1,139 | 616 | 354 | 289 | 94 | 1,516 |
| SGLang HT | 1,090 | 1,521 | 930 | 468 | 392 | *rej* | 4,009 |
| vLLM latency | 760 | 1,238 | 569 | 320 | 300 | 95 | 2,014 |
| vLLM balanced | 801 | 1,239 | 599 | 328 | 287 | 99 | 2,051 |
| vLLM throughput | — | — | — | — | — | — | — |
| **DeepSeek-V4.1-Flash** | | | | | | | |
| SGLang LL | 437 | 655 | 624 | 624 | *n/c* | *n/c* | 2,184 |
| SGLang HT | 437 | 655 | 624 | *n/c* | *n/c* | *n/c* | 2,184 |
| vLLM latency | 1,620 | 2,927 | 1,731 | 1,006 | 965 | *n/c* | 5,343 |
| vLLM throughput | 1,331 | 2,095 | 1,333 | 746 | 992 | *rej* | 5,700 |
| **Qwen3.8-Flash-Next bf16** | | | | | | | |
| SGLang LL | 1,032 | 1,685 | 1,936 | 1,189 | 1,155 | 353 | 4,446 |
| SGLang HT | 1,857 | 2,403 | 1,873 | 1,100 | 1,100 | 359 | 4,327 |
| **Qwen3.8-Flash-Next FP8** | | | | | | | |
| SGLang LL | 1,170 | 1,681 | 1,917 | 1,198 | 1,390 | 342 | 4,453 |
| SGLang HT | 2,075 | 2,840 | 2,496 | 1,249 | 1,039 | 289 | 4,276 |
| vLLM balanced | 2,435 | 3,398 | 3,102 | 1,838 | 2,082 | 378 | 6,475 |
| vLLM throughput | — | — | — | — | — | — | — |

*rej* — the server refused every request. *n/c* — no request finished inside the window.

**Qwen3.8-Flash-Next FP8 under vLLM wins six of the seven benchmarks**, and by
wide margins on the short shapes — 2,435 tok/s on API-S against 1,529 for the
best GLM-5.3-Flash cell. The exception is DOC-L, the 128k-token shape, where
GLM-5.3-Flash's vLLM throughput cell leads at 587.

No configuration is good at everything. The best API-S cell is mid-table on
DOC-L; the best DOC-L cell is mid-table on API-S. If your traffic is one shape,
benchmark that shape.

## Throughtput vs Concurrency

<figure class="qz">
  <svg viewBox="0 0 640 318" role="img" aria-label="Output throughput against concurrency on BATCH-D for three GLM-5.3-Flash configurations. SGLang low-latency: 205 at c1, 2,158 at c16, 4,093 at c64, 4,149 at c256; SGLang high-throughput: 171 at c1, 1,638 at c16, 4,369 at c64, 14,210 at c256; vLLM throughput: 137 at c1, 1,403 at c16, 4,490 at c64, 11,805 at c256.">
    <rect x="58" y="8" width="13" height="3" rx="1.5" fill="var(--ok)"/>
    <text class="sub" x="76" y="12.5">SGLang low-latency</text>
    <rect x="186" y="8" width="13" height="3" rx="1.5" fill="var(--bad)"/>
    <text class="sub" x="204" y="12.5">SGLang high-throughput</text>
    <rect x="337" y="8" width="13" height="3" rx="1.5" fill="var(--vl)"/>
    <text class="sub" x="355" y="12.5">vLLM throughput</text>
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
    <polyline points="58,270 227,252 397,207 566,100" fill="none" stroke="var(--vl)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
    <circle cx="58" cy="270" r="3.5" fill="var(--vl)"/>
    <circle cx="227" cy="252" r="3.5" fill="var(--vl)"/>
    <circle cx="397" cy="207" r="3.5" fill="var(--vl)"/>
    <circle cx="566" cy="100" r="3.5" fill="var(--vl)"/>
    <text class="val" x="574" y="68" fill="var(--bad)">14,210</text>
    <text class="val" x="574" y="103" fill="var(--vl)">11,805</text>
    <text class="val" x="574" y="215" fill="var(--ok)">4,149</text>
  </svg>
  <figcaption>
    GLM-5.3-Flash on BATCH-D. The low-latency recipe leads at low concurrency and
    is overtaken between 16 and 64 concurrent requests.
  </figcaption>
</figure>

Every model shows this shape. What is **not** portable is where the lines cross:

| model | crossover |
|---|---|
| DeepSeek-V4.1-Flash | below 16 concurrent requests |
| GLM-5.3 | between 16 and 64 |
| GLM-5.3-Flash | between 16 and 64 |
| Qwen3.8-Flash-Next (bf16 and FP8) | above 64 |

A rule of thumb learned on one model picks the wrong recipe for another. If you
serve Qwen at 32 concurrent requests, the low-latency recipe is still the faster
choice; at the same load DeepSeek has already crossed over.

## Qwen3.8-Flash-Next Quantization comparison

Comparison of Qwen3.8-Flash-Next BF16 checkpoint (336 GB) and FP8 (173 GB) checkpoints:

| Qwen3.8-Flash-Next | BATCH-D c64 | BATCH-D peak |
|---|--:|--:|
| BF16, SGLang high-throughput | 4,327 | 8,176 ‡ |
| FP8, SGLang high-throughput | 4,276 | 7,705 † |

On this node both perform the same, even if BF16 is larger.

## Conclusion
It is important to use the right tuning for the type of workload expected, considering the balance between throughtput and latency.

The engine use is not so important but it is important to do some bechmarking before moving to production.
