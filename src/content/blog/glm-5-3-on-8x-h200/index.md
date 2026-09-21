---
title: 'Four frontier models on one 8×H200 server'
description: 'Twenty serving configurations across vLLM and SGLang. The tuning recipe matters more than the engine.'
publishDate: 2026-09-21
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

Four models — GLM-5.3, GLM-5.3-Flash, DeepSeek-V4.1-Flash and Qwen3.8-Flash-Next —
served twenty different ways on the same 8×H200 node, measured on the same seven
workload shapes I used for the [B300 comparison](/blog/throughput-benchmarking-on-b300/).
232 benchmark runs.

The finding that survived everything else: **the tuning recipe moves throughput
further than the engine does, and usually further than the model does.** Peak
batch throughput across the twenty configurations spans 8.7×, and most of that
spread is recipe, not hardware and not model choice.

## One request at a time

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

## Under load

<figure class="qz">
  <svg viewBox="0 0 640 528" role="img" aria-label="Peak batch output throughput for six serving configurations. SGLang high-throughput 14,210, vLLM throughput 11,805, vLLM throughput 10,490, SGLang high-throughput 8,412, vLLM balanced 8,412, SGLang high-throughput 8,176, SGLang high-throughput 6,500, vLLM balanced 5,787, vLLM latency 5,343, SGLang low-latency 5,219, vLLM latency 5,097, SGLang low-latency 4,457, SGLang low-latency 4,149, SGLang high-throughput 2,184, SGLang low-latency 2,184, vLLM balanced 2,051, vLLM latency 2,014, SGLang low-latency 1,636.">
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
      <text class="sub" x="188" y="128" text-anchor="end">Qwen3.8 FP8 · peak of c1–c256</text>
      <rect x="196" y="108" width="190" height="14" rx="3" fill="var(--bad)"/>
      <text class="val" x="394" y="119">8,412</text>
      <text class="lab" x="188" y="144" text-anchor="end">vLLM balanced</text>
      <text class="sub" x="188" y="154" text-anchor="end">Qwen3.8 FP8 · peak of c1–c256</text>
      <rect x="196" y="134" width="190" height="14" rx="3" fill="var(--vl)"/>
      <text class="val" x="394" y="145">8,412</text>
      <text class="lab" x="188" y="170" text-anchor="end">SGLang high-throughput</text>
      <text class="sub" x="188" y="180" text-anchor="end">Qwen3.8 bf16 · peak of c1–c256</text>
      <rect x="196" y="160" width="185" height="14" rx="3" fill="var(--bad)"/>
      <text class="val" x="389" y="171">8,176</text>
      <text class="lab" x="188" y="196" text-anchor="end">SGLang high-throughput</text>
      <text class="sub" x="188" y="206" text-anchor="end">GLM-5.3 · peak of c1–c256</text>
      <rect x="196" y="186" width="147" height="14" rx="3" fill="var(--bad)"/>
      <text class="val" x="351" y="197">6,500</text>
      <text class="lab" x="188" y="222" text-anchor="end">vLLM balanced</text>
      <text class="sub" x="188" y="232" text-anchor="end">GLM-5.3-Flash · peak of c1–c256</text>
      <rect x="196" y="212" width="131" height="14" rx="3" fill="var(--vl)"/>
      <text class="val" x="335" y="223">5,787</text>
      <text class="lab" x="188" y="248" text-anchor="end">vLLM latency</text>
      <text class="sub" x="188" y="258" text-anchor="end">DeepSeek-V4.1 · peak of c1–c256</text>
      <rect x="196" y="238" width="121" height="14" rx="3" fill="var(--ok)"/>
      <text class="val" x="325" y="249">5,343</text>
      <text class="lab" x="188" y="274" text-anchor="end">SGLang low-latency</text>
      <text class="sub" x="188" y="284" text-anchor="end">Qwen3.8 bf16 · peak of c1–c256</text>
      <rect x="196" y="264" width="118" height="14" rx="3" fill="var(--ok)"/>
      <text class="val" x="322" y="275">5,219</text>
      <text class="lab" x="188" y="300" text-anchor="end">vLLM latency</text>
      <text class="sub" x="188" y="310" text-anchor="end">GLM-5.3-Flash · peak of c1–c256</text>
      <rect x="196" y="290" width="115" height="14" rx="3" fill="var(--ok)"/>
      <text class="val" x="319" y="301">5,097</text>
      <text class="lab" x="188" y="326" text-anchor="end">SGLang low-latency</text>
      <text class="sub" x="188" y="336" text-anchor="end">Qwen3.8 FP8 · peak of c1–c256</text>
      <rect x="196" y="316" width="101" height="14" rx="3" fill="var(--ok)"/>
      <text class="val" x="305" y="327">4,457</text>
      <text class="lab" x="188" y="352" text-anchor="end">SGLang low-latency</text>
      <text class="sub" x="188" y="362" text-anchor="end">GLM-5.3-Flash · peak of c1–c256</text>
      <rect x="196" y="342" width="94" height="14" rx="3" fill="var(--ok)"/>
      <text class="val" x="298" y="353">4,149</text>
      <text class="lab" x="188" y="378" text-anchor="end">SGLang high-throughput</text>
      <text class="sub" x="188" y="388" text-anchor="end">DeepSeek-V4.1 · peak of c1–c256</text>
      <rect x="196" y="368" width="49" height="14" rx="3" fill="var(--bad)"/>
      <text class="val" x="253" y="379">2,184</text>
      <text class="lab" x="188" y="404" text-anchor="end">SGLang low-latency</text>
      <text class="sub" x="188" y="414" text-anchor="end">DeepSeek-V4.1 · peak of c1–c256</text>
      <rect x="196" y="394" width="49" height="14" rx="3" fill="var(--ok)"/>
      <text class="val" x="253" y="405">2,184</text>
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
cell reaches **14,210 tok/s** — 3.4× its own low-latency sibling, and nine times
what that same model manages single-stream.

## Where the recipes cross — and why you cannot reuse the answer

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

## Engines, compared properly

My first pass at this compared SGLang's tuned high-throughput cell against
vLLM's default, concluded vLLM faded under load, and was wrong. vLLM publishes
deployment strategies with an explicit `orientation` — `single_node_tp` is
**latency**, `single_node_tep` **balanced**, `single_node_dep` **throughput** —
and I had only run the latency one. A latency strategy fading at high
concurrency is what it is designed to do.

At matched strategies the two engines are close, and which one leads depends on
the workload:

| GLM-5.3-Flash | CHAT-S peak | BATCH-D peak |
|---|--:|--:|
| SGLang high-throughput | 4,368 | **14,210** |
| vLLM throughput | **4,905** | 11,805 |

vLLM is ahead on chat-shaped traffic, SGLang on batch. On Qwen FP8, vLLM's
balanced cell leads CHAT-S 5,257 to 4,370. These are ordinary engineering
differences of tens of percent.

## The exception: DeepSeek-V4.1-Flash

| DeepSeek-V4.1-Flash | 1 stream | BATCH-D c64 | BATCH-D peak |
|---|--:|--:|--:|
| SGLang low-latency | 68 | 2,184 | 2,184 |
| SGLang high-throughput | 68 | 2,184 | 2,184 |
| vLLM latency | **294** | 5,343 | 5,343 |
| vLLM throughput | 167 | **5,700** | **10,490** |

vLLM is 4.3× faster single-stream and 2.6× faster at 64 concurrent requests on
the same model, same node, same workload. Part of that is speculative decoding —
vLLM runs DSpark, and the SGLang H200 cell is the one row in that cookbook with
no speculative decoding at all — but not a factor of four.

The other oddity: SGLang's two published DeepSeek cells return **identical
numbers at every point** (68, 546, 2,184; and 872 against 873 on CHAT-S). The
low-latency and high-throughput rows differ only in
`--enable-decoder-swa-bounded-replay` versus `--max-running-requests 256`, and on
this hardware neither flag changes anything. They are effectively one
configuration.

I had previously measured this same SGLang configuration in isolation and
attributed its ceiling to MXFP4 emulation on Hopper, which has no native FP4.
That explanation now looks wrong: the same checkpoint on the same GPUs reaches
five times the throughput under a different engine.

## Quantization bought nothing here

Qwen3.8-Flash-Next ships a bf16 checkpoint (336 GB) and an FP8 one (173 GB). On
this node they perform the same:

| Qwen3.8-Flash-Next | BATCH-D c64 | BATCH-D peak |
|---|--:|--:|
| bf16, SGLang high-throughput | 4,327 | 8,176 |
| FP8, SGLang high-throughput | 4,276 | **8,412** |

Half the weight memory, no measurable throughput change in either direction.
That is a good result if you need the VRAM back and a non-result if you were
hoping for speed.

## Three configurations do not fit at all

- **GLM-5.3's high-throughput cell rejects every 128k-token prompt.**
  `--dp 8 --enable-dp-attention` replicates the KV pool per data-parallel rank,
  leaving 111,552 tokens against a 131,084-token request. All 460 DOC-L requests
  were refused. The low-latency cell serves the same workload fine.
- **vLLM's throughput strategy fails outright for GLM-5.3** — 704 GB of weights
  replicated per DP rank leaves no room for cache blocks.
- **And for Qwen FP8** — a 95 GiB allocation on a 141 GiB card.

## NVFP4 needs different silicon

I set out to compare FP8 against NVFP4. These H200s are compute capability
**9.0**; NVFP4 needs a Blackwell FP4 tensor core at 10.0 or above. Both SGLang
cookbooks publish NVFP4 cells only for B200, B300, GB200 and GB300, and both
vLLM recipes say Blackwell-only. Half the intended matrix was never runnable
here — worth checking before you plan a quantization comparison around a node
you already have.

## Two ways this measurement nearly went wrong

**Throughput averaged over completed requests.** GuideLLM's
`output_tokens_per_second` averages across requests that finished. On BATCH-D at
256 concurrent streams, 2 of 257 requests finish inside the window — and that
metric reported **53 tok/s** while the engine was sustaining about **5,100**.
Every figure here is instead all output tokens produced over the measured
window, including by requests still streaming at the cutoff.

**A per-run timeout that deleted the hardest cells.** Runs are bounded by
duration, but the harness still has to wait for requests in flight when the
window closes. At 8,192 output tokens and a slow model that drain takes about an
hour; my ten-minute grace period killed those runs outright, leaving holes in
exactly the decode-heavy cells the campaign existed to measure — and leaving them
*silently*, because a killed run writes no output at all. Two of my interim
conclusions came from reading those holes as data.

## Method

One node, 8× NVIDIA H200 (SM90, 141 GiB each). TP8 throughout except Qwen's
SGLang cells, which are TP4. Flags copied verbatim from each model's published
`hw=h200` cell, with three forced deviations: `--max-model-len 262144` for
GLM-5.3 under vLLM (the published command will not boot otherwise),
`--max-num-seqs 512` for GLM-5.3-Flash under data parallelism, and a raised
engine-start timeout.

GuideLLM 0.7.1, seven workload shapes from 2,048 to 131,072 input tokens, runs
bounded by duration rather than prompt count. Every configuration was checked
for silent Triton `w8a8_block_fp8_matmul` fallbacks — all twenty came back
clean.

One repetition per cell, so treat differences under about 10% as unresolved. Two
DeepSeek BATCH-D runs at 256 concurrent streams were still draining when this
was written, so those two peaks are lower bounds.
