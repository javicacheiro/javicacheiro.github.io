---
title: 'How many users can one 8×B300 server serve?'
description: 'Eight open-weight models on one 8×B300 node: from 14 to 200 fast chats at once, and several times more people once you count the time they spend reading.'
publishDate: 2026-11-16
draft: true
tags:
  - ai
  - llm
  - gpu
  - benchmarking
  - inference
---

<style>
.uu {
  --s1: #2a78d6; --s2: #eb6834; --s3: #1baf7a; --s4: #eda100; --s5: #e87ba4; --s6: #008300; --s7: #6250d6;
  --grid: hsl(var(--border));
  --ink: hsl(var(--foreground));
  --dim: hsl(var(--muted-foreground));
  margin: 2rem 0;
}
.dark .uu { --s1: #3987e5; --s2: #d95926; --s3: #199e70; --s4: #c98500; --s5: #d55181; --s6: #008300; --s7: #9085e9; }
.uu svg { width: 100%; height: auto; display: block; overflow: visible; }
.uu figcaption { font-size: .875rem; line-height: 1.55; color: var(--dim); margin-top: .8rem; }
.uu .lab { font-size: 11.5px; fill: var(--ink); }
.uu .sub { font-size: 10.5px; fill: var(--dim); }
.uu .ax  { font-size: 11px; fill: var(--dim); }
.uu .gl  { stroke: var(--grid); stroke-width: 1; }
.uu .ref { stroke: var(--dim); stroke-width: 1; stroke-dasharray: 4 4; opacity: .7; }
.uu .s1 { fill: none; stroke: var(--s1); stroke-width: 2; stroke-linejoin: round; }
.uu .d1 { fill: var(--s1); stroke: hsl(var(--background)); stroke-width: 2; }
.uu .s2 { fill: none; stroke: var(--s2); stroke-width: 2; stroke-linejoin: round; }
.uu .d2 { fill: var(--s2); stroke: hsl(var(--background)); stroke-width: 2; }
.uu .s3 { fill: none; stroke: var(--s3); stroke-width: 2; stroke-linejoin: round; }
.uu .d3 { fill: var(--s3); stroke: hsl(var(--background)); stroke-width: 2; }
.uu .s4 { fill: none; stroke: var(--s4); stroke-width: 2; stroke-linejoin: round; }
.uu .d4 { fill: var(--s4); stroke: hsl(var(--background)); stroke-width: 2; }
.uu .s5 { fill: none; stroke: var(--s5); stroke-width: 2; stroke-linejoin: round; }
.uu .d5 { fill: var(--s5); stroke: hsl(var(--background)); stroke-width: 2; }
.uu .s6 { fill: none; stroke: var(--s6); stroke-width: 2; stroke-linejoin: round; }
.uu .d6 { fill: var(--s6); stroke: hsl(var(--background)); stroke-width: 2; }
.uu .s7 { fill: none; stroke: var(--s7); stroke-width: 2; stroke-linejoin: round; }
.uu .d7 { fill: var(--s7); stroke: hsl(var(--background)); stroke-width: 2; }
.uu .lg { display: flex; flex-wrap: wrap; gap: .3rem 1rem; font-size: .8rem; color: var(--dim); margin-bottom: .6rem; }
.uu .lg span { display: inline-flex; align-items: center; gap: .35rem; }
.uu .lg i { display: inline-block; width: 14px; height: 3px; border-radius: 2px; }
.uu .k1 { background: var(--s1); }
.uu .k2 { background: var(--s2); }
.uu .k3 { background: var(--s3); }
.uu .k4 { background: var(--s4); }
.uu .k5 { background: var(--s5); }
.uu .k6 { background: var(--s6); }
.uu .k7 { background: var(--s7); }
</style>

After measuring how fast the doctor-bench models run on one 8×B300 server, the question I
get asked most is a different one: how many people can use it at the same time?

The answer depends on what you call a user. So I measured it three ways. First, how many
requests a model can serve at once while each one still gets a given speed. Then the same
with requests arriving at random, the way real users send them. And finally people, who
spend most of their time reading and typing rather than waiting for tokens.

## Setup

The node is the same 8×B300 server as in my earlier posts (268 GiB per GPU). I ran one model
at a time, each with the vLLM settings it was scored with on
[doctor-bench](/blog/frontier-models-on-doctor-bench), including its speculative decoding:
MTP for GLM, DSpark for DeepSeek and DFlash for MiMo. Kimi K3.1 Pro ran in its standard
configuration and also with DSpark. Mistral Large 4 ran on half the node (TP4), as it was
served for doctor-bench, so a full node holds two copies and roughly doubles its numbers.

There are two workloads, both with synthetic text of fixed length:

- **Chat:** a 2k-token prompt and a 512-token answer.
- **Long prompts:** a 16k-token prompt and a 2k-token answer, closer to a coding assistant
  or a question over a long document.

GuideLLM generated the load from another server. I measured on the server side, from vLLM's
metrics, after a warm-up and over a window of about two request durations. *Speed per user*
is the total output rate divided by the number of requests running. *First token* is the
median time to first token. vLLM reports it as a histogram, so the values are bucket upper
bounds (≤ 0.25, 0.5, 0.75, 1, 2.5, 5 s and so on). Chat was tested at 1 to 512 concurrent
requests, long prompts at 1 to 256. Between tested points I interpolate on a log scale.

## What speed does a user need?

People read about 4 words per second, roughly 5–6 tokens per second. So 20 tokens per
second already outruns anyone reading the answer as it streams. 50 tokens per second feels
instant, and it matters for two cases: answers you skim rather than read, like code, and
thinking models, which generate reasoning tokens before the answer. At 20 tok/s, 1,000
reasoning tokens are 50 seconds of waiting.

So I use two levels. *Fast* means at least 50 tok/s per user, with the first token within 1 s.
*Usable* means at least 20 tok/s, with the first token within 2.5 s. For long prompts the
first token takes longer by nature, so I allow 2.5 s and 10 s.

## Concurrent requests

<figure class="uu">
<div class="lg"><span><i class="k1"></i>MiMo-Flash</span><span><i class="k2"></i>DeepSeek-Flash</span><span><i class="k3"></i>GLM-Flash</span><span><i class="k4"></i>GLM-5.3</span><span><i class="k5"></i>Mistral (½ node)</span><span><i class="k7"></i>Kimi + DSpark</span><span><i class="k6"></i>Kimi K3.1</span></div>
<svg viewBox="0 0 640 360" role="img" aria-labelledby="uu-t">
<title id="uu-t">Chat output speed per user against concurrent requests on one 8×B300 server, log scales. Full numbers in the table below.</title>
<line class="gl" x1="56" x2="500" y1="302.5" y2="302.5"/><text class="ax" x="50" y="306.5" text-anchor="end">10</text>
<line class="ref" x1="56" x2="500" y1="254.3" y2="254.3"/><text class="ax" x="50" y="258.3" text-anchor="end">20</text>
<line class="ref" x1="56" x2="500" y1="190.7" y2="190.7"/><text class="ax" x="50" y="194.7" text-anchor="end">50</text>
<line class="gl" x1="56" x2="500" y1="142.5" y2="142.5"/><text class="ax" x="50" y="146.5" text-anchor="end">100</text>
<line class="gl" x1="56" x2="500" y1="94.3" y2="94.3"/><text class="ax" x="50" y="98.3" text-anchor="end">200</text>
<line class="gl" x1="56" x2="500" y1="30.7" y2="30.7"/><text class="ax" x="50" y="34.7" text-anchor="end">500</text>
<text class="ax" x="56.0" y="334" text-anchor="middle">1</text>
<text class="ax" x="105.3" y="334" text-anchor="middle">2</text>
<text class="ax" x="154.7" y="334" text-anchor="middle">4</text>
<text class="ax" x="204.0" y="334" text-anchor="middle">8</text>
<text class="ax" x="253.3" y="334" text-anchor="middle">16</text>
<text class="ax" x="302.7" y="334" text-anchor="middle">32</text>
<text class="ax" x="352.0" y="334" text-anchor="middle">64</text>
<text class="ax" x="401.3" y="334" text-anchor="middle">128</text>
<text class="ax" x="450.7" y="334" text-anchor="middle">256</text>
<text class="ax" x="500.0" y="334" text-anchor="middle">512</text>
<text class="ax" x="278" y="352" text-anchor="middle">concurrent chat requests</text>
<text class="ax" x="50" y="12" text-anchor="end">tok/s</text>
<path class="s1" d="M56.0,36.3 L154.7,37.6 L204.0,64.9 L253.3,61.2 L302.7,106.0 L352.0,126.7 L401.3,157.5 L450.7,209.5 L500.0,227.1"/>
<circle class="d1" cx="56.0" cy="36.3" r="3.5"><title>MiMo-V2.6-Flash, 1 requests: 461 tok/s per user, first token ≤ 0.08 s</title></circle>
<circle class="d1" cx="154.7" cy="37.6" r="3.5"><title>MiMo-V2.6-Flash, 4 requests: 452 tok/s per user, first token ≤ 0.08 s</title></circle>
<circle class="d1" cx="204.0" cy="64.9" r="3.5"><title>MiMo-V2.6-Flash, 8 requests: 306 tok/s per user, first token ≤ 0.08 s</title></circle>
<circle class="d1" cx="253.3" cy="61.2" r="3.5"><title>MiMo-V2.6-Flash, 16 requests: 322 tok/s per user, first token ≤ 0.08 s</title></circle>
<circle class="d1" cx="302.7" cy="106.0" r="3.5"><title>MiMo-V2.6-Flash, 32 requests: 169 tok/s per user, first token ≤ 0.1 s</title></circle>
<circle class="d1" cx="352.0" cy="126.7" r="3.5"><title>MiMo-V2.6-Flash, 64 requests: 126 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d1" cx="401.3" cy="157.5" r="3.5"><title>MiMo-V2.6-Flash, 128 requests: 81 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d1" cx="450.7" cy="209.5" r="3.5"><title>MiMo-V2.6-Flash, 256 requests: 38 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d1" cx="500.0" cy="227.1" r="3.5"><title>MiMo-V2.6-Flash, 512 requests: 30 tok/s per user, first token ≤ 0.25 s</title></circle>
<path class="s2" d="M56.0,60.8 L154.7,61.2 L204.0,99.1 L253.3,123.8 L302.7,153.9 L352.0,186.2 L401.3,213.5 L450.7,240.5 L500.0,251.6"/>
<circle class="d2" cx="56.0" cy="60.8" r="3.5"><title>DeepSeek-V4.1-Flash-1017, 1 requests: 324 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d2" cx="154.7" cy="61.2" r="3.5"><title>DeepSeek-V4.1-Flash-1017, 4 requests: 322 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d2" cx="204.0" cy="99.1" r="3.5"><title>DeepSeek-V4.1-Flash-1017, 8 requests: 187 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d2" cx="253.3" cy="123.8" r="3.5"><title>DeepSeek-V4.1-Flash-1017, 16 requests: 131 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d2" cx="302.7" cy="153.9" r="3.5"><title>DeepSeek-V4.1-Flash-1017, 32 requests: 85 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d2" cx="352.0" cy="186.2" r="3.5"><title>DeepSeek-V4.1-Flash-1017, 64 requests: 53 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d2" cx="401.3" cy="213.5" r="3.5"><title>DeepSeek-V4.1-Flash-1017, 128 requests: 36 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d2" cx="450.7" cy="240.5" r="3.5"><title>DeepSeek-V4.1-Flash-1017, 256 requests: 24 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d2" cx="500.0" cy="251.6" r="3.5"><title>DeepSeek-V4.1-Flash-1017, 512 requests: 21 tok/s per user, first token ≤ 0.5 s</title></circle>
<path class="s3" d="M56.0,75.0 L154.7,96.7 L204.0,117.7 L253.3,138.6 L302.7,164.5 L352.0,187.1 L401.3,221.5 L450.7,261.3 L500.0,280.1"/>
<circle class="d3" cx="56.0" cy="75.0" r="3.5"><title>GLM-5.3-Flash, 1 requests: 264 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d3" cx="154.7" cy="96.7" r="3.5"><title>GLM-5.3-Flash, 4 requests: 193 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d3" cx="204.0" cy="117.7" r="3.5"><title>GLM-5.3-Flash, 8 requests: 143 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d3" cx="253.3" cy="138.6" r="3.5"><title>GLM-5.3-Flash, 16 requests: 106 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d3" cx="302.7" cy="164.5" r="3.5"><title>GLM-5.3-Flash, 32 requests: 73 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d3" cx="352.0" cy="187.1" r="3.5"><title>GLM-5.3-Flash, 64 requests: 53 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d3" cx="401.3" cy="221.5" r="3.5"><title>GLM-5.3-Flash, 128 requests: 32 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d3" cx="450.7" cy="261.3" r="3.5"><title>GLM-5.3-Flash, 256 requests: 18 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d3" cx="500.0" cy="280.1" r="3.5"><title>GLM-5.3-Flash, 512 requests: 14 tok/s per user, first token ≤ 0.75 s</title></circle>
<path class="s4" d="M56.0,93.0 L154.7,99.8 L204.0,139.6 L253.3,157.7 L302.7,189.2 L352.0,212.3 L401.3,236.1 L450.7,275.3 L500.0,312.2"/>
<circle class="d4" cx="56.0" cy="93.0" r="3.5"><title>GLM-5.3, 1 requests: 204 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d4" cx="154.7" cy="99.8" r="3.5"><title>GLM-5.3, 4 requests: 185 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d4" cx="204.0" cy="139.6" r="3.5"><title>GLM-5.3, 8 requests: 104 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d4" cx="253.3" cy="157.7" r="3.5"><title>GLM-5.3, 16 requests: 80 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d4" cx="302.7" cy="189.2" r="3.5"><title>GLM-5.3, 32 requests: 51 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d4" cx="352.0" cy="212.3" r="3.5"><title>GLM-5.3, 64 requests: 37 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d4" cx="401.3" cy="236.1" r="3.5"><title>GLM-5.3, 128 requests: 26 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d4" cx="450.7" cy="275.3" r="3.5"><title>GLM-5.3, 256 requests: 15 tok/s per user, first token ≤ 0.75 s</title></circle>
<circle class="d4" cx="500.0" cy="312.2" r="3.5"><title>GLM-5.3, 512 requests: 9 tok/s per user, first token ≤ 2.5 s</title></circle>
<path class="s5" d="M56.0,138.5 L154.7,141.3 L204.0,156.4 L253.3,173.8 L302.7,197.2 L352.0,220.2 L401.3,254.0"/>
<circle class="d5" cx="56.0" cy="138.5" r="3.5"><title>Mistral Large 4 (half node), 1 requests: 106 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d5" cx="154.7" cy="141.3" r="3.5"><title>Mistral Large 4 (half node), 4 requests: 102 tok/s per user, first token ≤ 0.75 s</title></circle>
<circle class="d5" cx="204.0" cy="156.4" r="3.5"><title>Mistral Large 4 (half node), 8 requests: 82 tok/s per user, first token ≤ 0.75 s</title></circle>
<circle class="d5" cx="253.3" cy="173.8" r="3.5"><title>Mistral Large 4 (half node), 16 requests: 64 tok/s per user, first token ≤ 2.5 s</title></circle>
<circle class="d5" cx="302.7" cy="197.2" r="3.5"><title>Mistral Large 4 (half node), 32 requests: 46 tok/s per user, first token ≤ 2.5 s</title></circle>
<circle class="d5" cx="352.0" cy="220.2" r="3.5"><title>Mistral Large 4 (half node), 64 requests: 33 tok/s per user, first token ≤ 5 s</title></circle>
<circle class="d5" cx="401.3" cy="254.0" r="3.5"><title>Mistral Large 4 (half node), 128 requests: 20 tok/s per user, first token ≤ 5 s</title></circle>
<path class="s7" d="M56.0,109.8 L154.7,140.9 L204.0,155.8 L253.3,183.4 L302.7,206.7 L352.0,230.2"/>
<circle class="d7" cx="56.0" cy="109.8" r="3.5"><title>Kimi K3.1 Pro + DSpark, 1 requests: 160 tok/s per user, first token ≤ 0.08 s</title></circle>
<circle class="d7" cx="154.7" cy="140.9" r="3.5"><title>Kimi K3.1 Pro + DSpark, 4 requests: 102 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d7" cx="204.0" cy="155.8" r="3.5"><title>Kimi K3.1 Pro + DSpark, 8 requests: 83 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d7" cx="253.3" cy="183.4" r="3.5"><title>Kimi K3.1 Pro + DSpark, 16 requests: 56 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d7" cx="302.7" cy="206.7" r="3.5"><title>Kimi K3.1 Pro + DSpark, 32 requests: 40 tok/s per user, first token ≤ 0.5 s</title></circle>
<circle class="d7" cx="352.0" cy="230.2" r="3.5"><title>Kimi K3.1 Pro + DSpark, 64 requests: 28 tok/s per user, first token ≤ 2.5 s</title></circle>
<path class="s6" d="M56.0,164.4 L154.7,152.7 L204.0,177.4 L253.3,194.2 L302.7,208.6 L352.0,232.0 L401.3,259.4"/>
<circle class="d6" cx="56.0" cy="164.4" r="3.5"><title>Kimi K3.1 Pro, 1 requests: 73 tok/s per user, first token ≤ 0.25 s</title></circle>
<circle class="d6" cx="154.7" cy="152.7" r="3.5"><title>Kimi K3.1 Pro, 4 requests: 86 tok/s per user, first token ≤ 0.75 s</title></circle>
<circle class="d6" cx="204.0" cy="177.4" r="3.5"><title>Kimi K3.1 Pro, 8 requests: 60 tok/s per user, first token ≤ 1 s</title></circle>
<circle class="d6" cx="253.3" cy="194.2" r="3.5"><title>Kimi K3.1 Pro, 16 requests: 48 tok/s per user, first token ≤ 1 s</title></circle>
<circle class="d6" cx="302.7" cy="208.6" r="3.5"><title>Kimi K3.1 Pro, 32 requests: 39 tok/s per user, first token ≤ 2.5 s</title></circle>
<circle class="d6" cx="352.0" cy="232.0" r="3.5"><title>Kimi K3.1 Pro, 64 requests: 28 tok/s per user, first token ≤ 5 s</title></circle>
<circle class="d6" cx="401.3" cy="259.4" r="3.5"><title>Kimi K3.1 Pro, 128 requests: 19 tok/s per user, first token ≤ 5 s</title></circle>
<text class="lab" x="508.0" y="231.1">MiMo-Flash</text>
<text class="lab" x="508.0" y="255.6">DeepSeek-Flash</text>
<text class="lab" x="508.0" y="284.1">GLM-Flash</text>
<text class="lab" x="508.0" y="316.2">GLM-5.3</text>
<text class="sub" x="60" y="186.7">50 tok/s</text>
<text class="sub" x="60" y="250.3">20 tok/s</text>
</svg>
<figcaption>Chat (2k → 512 tokens): output speed each user sees against the number of
concurrent requests, both on log scales. The dashed lines are the 50 and 20 tok/s levels. A
line ends where requests start queueing (median first token above 5 s). DeepSeek runs with
its recipe up to 128 requests and with the tuned configuration above that (see below). Kimi
with DSpark is limited to 64 sequences. Hover over a point for its value. All values are in
the tables further down.</figcaption>
</figure>

How many concurrent requests each model serves at each level, interpolated between tested
points. A "+" means it still passed at the highest concurrency I tested:

| model | chat ≥ 50 tok/s, first token ≤ 1 s | chat ≥ 20 tok/s, ≤ 2.5 s | long prompts ≥ 20 tok/s, ≤ 2.5 s | long prompts ≥ 20 tok/s, ≤ 10 s |
|---|---:|---:|---:|---:|
| MiMo-V2.6-Flash | 200 | 512+ | 256+ | 256+ |
| DeepSeek-V4.1-Flash-1017 | 72 | 512+ | 256+ | 256+ |
| MiMo-V2.6-Pro | 130 | 410 | 230 | 230 |
| GLM-5.3-Flash | 69 | 230 | 180 | 180 |
| Muse-Spark-Open | 32 | 260 | 4 | 190 |
| GLM-5.3 | 33 | 180 | 110 | 110 |
| Mistral Large 4 (half node) | 8 | 32 | 4 | 64 |
| Kimi K3.1 Pro + DSpark | 20 | 64 (max seqs) | 32 | 32 |
| Kimi K3.1 Pro | 14 | 32 | 4 | 16 |

The small-active-parameter models with speculative decoding are in a different league.
MiMo-V2.6-Flash keeps 200 chats at 50 tok/s and still gives 30 tok/s to each of 512.
DeepSeek-V4.1-Flash-1017 is limited by its recipe, which allows 128 sequences: beyond that
requests just queue. With 512 sequences **and** CUDA graphs captured for the larger
speculative batches, it still gives 21 tok/s to 512 chats. Raising only the sequence limit
makes it slower, a trap I described in the
[Kimi K3.1 Pro post](/blog/kimi-k3.1-pro-on-doctor-bench).

At the other end, Kimi K3.1 Pro serves about 14 fast chats and 32 usable ones. It's a 2.8T
parameter model, and a single user already gets only 73 tok/s, so there's little room before
it drops below 50. With DSpark speculative decoding, single-user speed doubles and it serves
about 20 fast chats. But it's limited to 64 sequences, and it isn't stable in the current
version of vLLM, so I don't run it in production. Mistral Large 4 on half a node serves
8 fast chats, so about 16 on a full node.

Long prompts are where the large models struggle. A 16k-token prompt takes a while to
process, and with many requests the prefills queue behind each other. With the first token
within 2.5 s, Kimi K3.1 Pro and Mistral Large 4 serve only about 4 requests at once. Muse
also shows 4 in that column, but its first-token times jump around between tests, so I
wouldn't read much into it. Allowing 10 s helps: Kimi gets to 16 and Mistral to 64. Beyond
that, Kimi's KV cache fills at about 75 requests of this size and requests queue for minutes.

## Random arrivals

In a fixed-concurrency test, a new request starts the moment one finishes. With fixed
lengths, requests tend to move in step: they prefill together and then decode together,
which makes the decode steps fast. Real users don't coordinate. So for each model I also
ran chat with Poisson arrivals at two rates, and compared the speed per user with the
fixed-concurrency curve at the same number of running requests:

| model | arrivals per s | running at once | speed per user | first token p50 / p95 | fixed concurrency, same load | people at one message per minute |
|---|---:|---:|---:|---:|---:|---:|
| MiMo-V2.6-Flash | 14.9 | 136 | 51 tok/s | 0.25 / 0.5 s | -28% | ~1,030 |
| MiMo-V2.6-Flash | 18.7 | 250 | 36 tok/s | 0.25 / 0.5 s | -1% | ~1,370 |
| DeepSeek-V4.1-Flash-1017 | 3.85 | 9 | 213 tok/s | 0.06 / 0.08 s | +26% | ~240 |
| DeepSeek-V4.1-Flash-1017 | 6.2 | 18 | 166 tok/s | 0.06 / 0.08 s | +47% | ~390 |
| MiMo-V2.6-Pro | 9.7 | 97 | 48 tok/s | 0.25 / 0.75 s | -15% | ~680 |
| MiMo-V2.6-Pro | 13.6 | 191 | 33 tok/s | 0.25 / 0.75 s | -3% | ~1,010 |
| GLM-5.3-Flash | 6.3 | 74 | 43 tok/s | 0.5 / 0.75 s | -2% | ~450 |
| GLM-5.3-Flash | 8.9 | 254 | 18 tok/s | 0.5 / 0.75 s | +2% | ~790 |
| Muse-Spark-Open | 6.7 | 108 | 32 tok/s | 0.5 / 0.75 s | -28% | ~510 |
| Muse-Spark-Open | 13.5 | 257 | 21 tok/s | 0.25 / 1 s | -14% | ~1,070 |
| GLM-5.3 | 2.4 | 21 | 58 tok/s | 0.25 / 0.75 s | -9% | ~170 |
| GLM-5.3 | 4.7 | 45 | 50 tok/s | 0.25 / 1 s | +17% | ~330 |
| Mistral Large 4 (half node) | 1.52 | 17 | 45 tok/s | 0.25 / 2.5 s | -22% | ~110 |
| Mistral Large 4 (half node) | 3.1 | 56 | 28 tok/s | 0.25 / 2.5 s | -14% | ~240 |
| Kimi K3.1 Pro + DSpark | 1.63 | 18 | 47 tok/s | 0.5 / 2.5 s | -8% | ~120 |
| Kimi K3.1 Pro | 1.37 | 26 | 30 tok/s | 0.5 / 1 s | -26% | ~110 |
| Kimi K3.1 Pro | 2.75 | 91 | 16 tok/s | 0.5 / 2.5 s | -23% | ~260 |

Most models lose speed per user under random arrivals, by up to 28%: MiMo-V2.6-Flash, Muse,
Mistral and Kimi lose 22–28% at the lower rate. The probable cause is that with a steady
trickle of arrivals, most decode steps also carry a prefill. The loss usually shrinks near
saturation (MiMo-Flash −1%, MiMo-Pro −3%), but not for Kimi, which loses 23% with its KV
cache full. GLM-5.3-Flash doesn't change, GLM-5.3 goes both ways (−9% and +17%), and
DeepSeek is 26–47% *faster* than its fixed-concurrency tests suggest, which I can't explain
yet. The median first token is as good or better with random arrivals, because prefills no
longer come in bursts, although the 95th percentile is sometimes worse. In practice, I would
discount fixed-concurrency numbers by up to a quarter.

## From requests to people

A person in a chat doesn't send requests back to back. They send a message, read the answer
and type the next one. Little's law gives the conversion: the number of requests running
equals the arrival rate times how long each request takes. If each person sends a message
about once a minute on top of the time spent waiting for the answer, then

*people ≈ requests running + arrival rate × 60 s*

That is the last column of the table above. Kimi K3.1 Pro at 1.37 requests per second ran
26 requests at once at 30 tok/s, with the first token in half a second. That's about 110
people actively chatting. At 2.75 requests per second it still gave 16 tok/s to each, and
that's about 260 people. MiMo-V2.6-Flash handled about 1,000 people at 51 tok/s and 1,400
at 36 tok/s. Near useful load, that's 3 to 8 times more people than concurrent requests,
and more at light load, where each request is short.

Two warnings. These are people chatting at that moment, not registered users. An
organisation with a few thousand users rarely has more than a fraction of them active at
once. And agents have no thinking time: a coding agent sends its next request the moment
the last one finishes. For agents, users and concurrent requests are the same thing, and
they usually carry long prompts, so the long-prompt column is the one to look at.

## What I take from this

- On one 8×B300 server, the fast mixture-of-experts models with speculative decoding
  (MiMo-V2.6-Flash, MiMo-V2.6-Pro, DeepSeek-V4.1-Flash) serve 70–200 fast chats at once and
  400–500 or more usable ones. Kimi K3.1 Pro serves 14 and 32.
- Counting people instead of requests multiplies those numbers by 3 to 8 for chat. It doesn't
  for agents.
- Test with random arrivals. Fixed concurrency flatters most models by up to a quarter.
- If you serve a model with speculative decoding at high concurrency, check that vLLM
  captures CUDA graphs for concurrency × (draft tokens + 1) tokens. Otherwise the server gets
  slower as you add requests.
- For long prompts, time to first token is the limit for the largest models, not speed per
  user.

**Caveats.** The text is synthetic, with fixed lengths and the end-of-sequence token ignored.
Speculative decoding depends on the text, so real traffic can accept more or fewer draft
tokens. Each point is a single run, and repeats of the same test differed by up to about
10%. First-token times are histogram bucket upper bounds. At 1–4 requests, speed per user is
slightly overstated, because there are short gaps between requests. DeepSeek above 128
requests uses a tuned configuration (512 sequences, CUDA graphs up to 3,072 tokens); every
other model uses its doctor-bench settings. Kimi K3.1 Pro with DSpark is not stable in the
current version of vLLM.

## All measurements

Chat, 2k → 512 tokens: speed per user (tok/s) · median first token, by number of concurrent
requests.

| model | 1 | 4 | 8 | 16 | 32 | 64 | 128 | 256 | 512 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| MiMo-V2.6-Flash | 461 · 0.08 s | 452 · 0.08 s | 306 · 0.08 s | 322 · 0.08 s | 169 · 0.1 s | 126 · 0.25 s | 81 · 0.25 s | 38 · 0.25 s | 30 · 0.25 s |
| DeepSeek-V4.1-Flash-1017 | 324 · 0.25 s | 322 · 0.25 s | 187 · 0.25 s | 131 · 0.25 s | 85 · 0.25 s | 53 · 0.5 s | 36 · 0.5 s | 24 · 0.5 s | 21 · 0.5 s |
| MiMo-V2.6-Pro | 333 · 0.1 s | 292 · 0.1 s | 198 · 0.1 s | 164 · 0.25 s | 107 · 0.25 s | 67 · 0.25 s | 51 · 0.25 s | 28 · 0.5 s | 17 · 0.5 s |
| GLM-5.3-Flash | 264 · 0.25 s | 193 · 0.25 s | 143 · 0.25 s | 106 · 0.25 s | 73 · 0.25 s | 53 · 0.5 s | 32 · 0.5 s | 18 · 0.5 s | 14 · 0.75 s |
| Muse-Spark-Open | 179 · 0.25 s | 148 · 0.25 s | 122 · 0.5 s | 120 · 0.5 s | 78 · 1 s | 72 · 2.5 s | 44 · 2.5 s | 28 · 0.75 s | 19 · 10 s |
| GLM-5.3 | 204 · 0.25 s | 185 · 0.25 s | 104 · 0.25 s | 80 · 0.25 s | 51 · 0.25 s | 37 · 0.5 s | 26 · 0.5 s | 15 · 0.75 s | 9 · 2.5 s |
| Mistral Large 4 (half node) | 106 · 0.25 s | 102 · 0.75 s | 82 · 0.75 s | 64 · 2.5 s | 46 · 2.5 s | 33 · 5 s | 20 · 5 s | 20 · 20 s | 19 · 80 s |
| Kimi K3.1 Pro + DSpark | 160 · 0.08 s | 102 · 0.5 s | 83 · 0.5 s | 56 · 0.5 s | 40 · 0.5 s | 28 · 2.5 s |  |  |  |
| Kimi K3.1 Pro | 73 · 0.25 s | 86 · 0.75 s | 60 · 1 s | 48 · 1 s | 39 · 2.5 s | 28 · 5 s | 19 · 5 s | 13 · 7.5 s | 13 · 80 s |

Long prompts, 16k → 2k tokens:

| model | 1 | 4 | 16 | 32 | 64 | 128 | 256 |
|---|---:|---:|---:|---:|---:|---:|---:|
| MiMo-V2.6-Flash | 541 · 0.25 s | 416 · 0.25 s | 276 · 0.25 s | 199 · 0.25 s | 136 · 0.25 s | 78 · 0.5 s | 38 · 0.25 s |
| DeepSeek-V4.1-Flash-1017 | 403 · 0.25 s | 324 · 0.25 s | 192 · 0.25 s | 127 · 0.25 s | 72 · 0.5 s | 44 · 0.75 s | 29 · 0.75 s |
| MiMo-V2.6-Pro | 382 · 0.5 s | 252 · 0.5 s | 145 · 0.5 s | 100 · 0.5 s | 63 · 0.5 s | 37 · 0.75 s | 18 · 1 s |
| GLM-5.3-Flash | 273 · 0.5 s | 202 · 0.5 s | 113 · 0.5 s | 76 · 0.5 s | 47 · 0.75 s | 30 · 1 s | 13 · 2.5 s |
| Muse-Spark-Open | 181 · 0.5 s | 146 · 2.5 s | 97 · 5 s | 66 · 2.5 s | 42 · 2.5 s | 27 · 2.5 s | 16 · 7.5 s |
| GLM-5.3 | 213 · 0.75 s | 135 · 0.75 s | 71 · 0.75 s | 44 · 0.75 s | 30 · 2.5 s | 18 · 2.5 s | 9 · 160 s |
| Mistral Large 4 (half node) | 100 · 2.5 s | 79 · 2.5 s | 50 · 10 s | 34 · 10 s | 20 · 10 s | 11 · 20 s | 10 · 640 s |
| Kimi K3.1 Pro + DSpark | 167 · 0.75 s | 106 · 1 s | 53 · 1 s | 35 · 1 s | 32 · 80 s |  |  |
| Kimi K3.1 Pro | 79 · 0.75 s | 63 · 2.5 s | 44 · 7.5 s | 30 · 20 s | 24 · 10 s | 16 · 160 s | 15 · 640 s |
