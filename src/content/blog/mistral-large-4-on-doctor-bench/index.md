---
title: 'Mistral Large 4 on doctor-bench'
description: 'Mistral Large 4 scores 88.6 on doctor-bench under the standard 4,096-token budget, and 92.1 with 16,384. Where it lands among the open models I have tested.'
publishDate: 2026-10-31
tags:
  - ai
  - llm
  - benchmarking
---

<style>
.ml {
  --acc: #2f6fd6;
  --hl: #d9822b;
  --mut: #b9b7ad;
  --base: #dedcd4;
  --grid: hsl(var(--border));
  --ink: hsl(var(--foreground));
  --dim: hsl(var(--muted-foreground));
  --bg: hsl(var(--background));
  margin: 2rem 0;
}
.dark .ml { --acc: #6b9cf0; --hl: #f0a35e; --mut: #6f6d66; --base: #4a4944; }
.ml svg { width: 100%; height: auto; display: block; overflow: visible; }
.ml figcaption { font-size: .875rem; line-height: 1.55; color: var(--dim); margin-top: .8rem; }
.ml .lab { font-size: 12px; fill: var(--ink); }
.ml .val { font-size: 12px; font-weight: 600; fill: var(--ink); }
.ml .ax  { font-size: 11px; fill: var(--dim); }
</style>

Mistral Large 4 came out a month ago: a trillion parameters, 49 billion active, open weights.
I ran it through [doctor-bench](/blog/introducing-doctor-bench) the same way as the
[seven frontier models](/blog/frontier-models-on-doctor-bench) and the
[LoRA experts](/blog/lora-experts-vs-frontier-models) before it. A fix only counts if the build
and tests pass afterwards.

Under the standard rules it scores 88.6, tied with DeepSeek-V4.1-Flash and my Qwen3.8-27B
expert. That number undersells it, though, and most of this post is about why.

<figure class="ml">
  <svg viewBox="0 0 640 498" role="img" aria-label="Mean task success over 14 doctor-bench benchmarks: Mistral Large 4, 16k tokens † 92.1; Kimi K3 ‡ 91.7; GLM-5.3 91.5; GLM-5.3-Flash 90.9; Qwen3.8-27B + LoRA, thinking * 89.8; Qwen3.8-27B + LoRA 88.6; Mistral Large 4 88.6; DeepSeek-V4.1-Flash 88.4; Nemotron Super VL + LoRA 88.4; MiMo-V2.6-Pro 88.3; Nemotron Super VL, thinking * 84.2; MiMo-V2.6-Flash 83.8; Qwen3.8-27B, thinking * 83.6; Qwen3.8-27B 82.0; Muse-Spark-Open 80.4; Kolibri-1 + LoRA 78.9; Nemotron Super VL 78.7; Kolibri-1 60.5; Qwen3-4B 51.3.">
    <line x1="230" x2="230" y1="8" y2="464" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="230" y="482" text-anchor="middle">0%</text>
    <line x1="322" x2="322" y1="8" y2="464" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="322" y="482" text-anchor="middle">25%</text>
    <line x1="415" x2="415" y1="8" y2="464" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="415" y="482" text-anchor="middle">50%</text>
    <line x1="508" x2="508" y1="8" y2="464" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="508" y="482" text-anchor="middle">75%</text>
    <line x1="600" x2="600" y1="8" y2="464" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="600" y="482" text-anchor="middle">100%</text>
    <text class="lab" x="220" y="26" text-anchor="end" style="font-weight:600">Mistral Large 4, 16k tokens †</text>
    <rect x="230" y="14" width="340.8" height="15" rx="2" fill="var(--hl)" fill-opacity=".45" stroke="var(--hl)" stroke-width="1.2"/>
    <text class="val" x="576.8" y="26">92.1</text>
    <text class="lab" x="220" y="50" text-anchor="end">Kimi K3 ‡</text>
    <rect x="230" y="38" width="339.2" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="575.2" y="50">91.7</text>
    <text class="lab" x="220" y="74" text-anchor="end">GLM-5.3</text>
    <rect x="230" y="62" width="338.6" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="574.6" y="74">91.5</text>
    <text class="lab" x="220" y="98" text-anchor="end">GLM-5.3-Flash</text>
    <rect x="230" y="86" width="336.4" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="572.4" y="98">90.9</text>
    <text class="lab" x="220" y="122" text-anchor="end">Qwen3.8-27B + LoRA, thinking *</text>
    <rect x="230" y="110" width="332.1" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="568.1" y="122">89.8</text>
    <text class="lab" x="220" y="146" text-anchor="end">Qwen3.8-27B + LoRA</text>
    <rect x="230" y="134" width="327.9" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="563.9" y="146">88.6</text>
    <text class="lab" x="220" y="170" text-anchor="end" style="font-weight:600">Mistral Large 4</text>
    <rect x="230" y="158" width="327.8" height="15" rx="2" fill="var(--hl)"/>
    <text class="val" x="563.8" y="170">88.6</text>
    <text class="lab" x="220" y="194" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <rect x="230" y="182" width="327.2" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="563.2" y="194">88.4</text>
    <text class="lab" x="220" y="218" text-anchor="end">Nemotron Super VL + LoRA</text>
    <rect x="230" y="206" width="327.2" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="563.2" y="218">88.4</text>
    <text class="lab" x="220" y="242" text-anchor="end">MiMo-V2.6-Pro</text>
    <rect x="230" y="230" width="326.9" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="562.9" y="242">88.3</text>
    <text class="lab" x="220" y="266" text-anchor="end">Nemotron Super VL, thinking *</text>
    <rect x="230" y="254" width="311.7" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="547.7" y="266">84.2</text>
    <text class="lab" x="220" y="290" text-anchor="end">MiMo-V2.6-Flash</text>
    <rect x="230" y="278" width="310.2" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="546.2" y="290">83.8</text>
    <text class="lab" x="220" y="314" text-anchor="end">Qwen3.8-27B, thinking *</text>
    <rect x="230" y="302" width="309.4" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="545.4" y="314">83.6</text>
    <text class="lab" x="220" y="338" text-anchor="end">Qwen3.8-27B</text>
    <rect x="230" y="326" width="303.4" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="539.4" y="338">82.0</text>
    <text class="lab" x="220" y="362" text-anchor="end">Muse-Spark-Open</text>
    <rect x="230" y="350" width="297.4" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="533.4" y="362">80.4</text>
    <text class="lab" x="220" y="386" text-anchor="end">Kolibri-1 + LoRA</text>
    <rect x="230" y="374" width="291.8" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="527.8" y="386">78.9</text>
    <text class="lab" x="220" y="410" text-anchor="end">Nemotron Super VL</text>
    <rect x="230" y="398" width="291.2" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="527.2" y="410">78.7</text>
    <text class="lab" x="220" y="434" text-anchor="end">Kolibri-1</text>
    <rect x="230" y="422" width="223.9" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="459.9" y="434">60.5</text>
    <text class="lab" x="220" y="458" text-anchor="end">Qwen3-4B</text>
    <rect x="230" y="446" width="189.7" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="425.7" y="458">51.3</text>
  </svg>
  <figcaption>Mean task success over the 14 canonical benchmarks, one greedy attempt per problem. Grey: open frontier models. Orange: Mistral Large 4. Blue: base models with my LoRA adapter. Light: the same base models without it. Every row has 4,096 output tokens except † Mistral Large 4 with 16,384 (a side run, not a leaderboard row) and * thinking mode with 32,768. ‡ Kimi K3 is also the teacher behind part of the benchmark.</figcaption>
</figure>

## It ran out of room

Mistral Large 4 can't switch reasoning off. The lowest setting is `reasoning_effort: low`, the
same situation as GLM-5.3, so it ran at low effort with the 4,096-token budget every other model
gets. On most problems that's fine: a test prompt used about 200 reasoning tokens before the
answer.

On 248 of the 4,722 problems it wasn't fine. The model was still reasoning when the budget ran
out and never produced an answer. That's 5.3%, against 1.2% and 1.0% for the two GLMs and about
0.2% or less for everyone else. And when it did answer, the answer passed 93.1% of the time, the
best rate of any frontier model I've tested (GLM-5.3 is next at 92.0).

So I ran the whole suite again with 16,384 tokens and nothing else changed. Truncations fell to
56. Of the 248 problems that had run out of room, 155 now pass. The problems it had already
answered barely moved (4,165 passes before, 4,175 after). The mean went from 88.6 to 92.1, which
is above Kimi K3.

<figure class="ml">
  <svg viewBox="0 0 640 282" role="img" aria-label="Mistral Large 4 per category at 4,096 and 16,384 output tokens, with the best frontier model at 4,096: build: 80.8 to 86.2, best frontier 87.6; linux: 91.0 to 91.6, best frontier 91.3; hpc: 95.7 to 96.9, best frontier 95.9; ml-systems: 88.6 to 94.1, best frontier 94.5; gromacs: 88.1 to 91.2, best frontier 92.1; slurm-admin: 86.2 to 90.5, best frontier 91.6; kubernetes: 92.2 to 94.8, best frontier 95.1.">
    <line x1="130.0" x2="130.0" y1="10" y2="248" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="130.0" y="266" text-anchor="middle">75%</text>
    <line x1="224.0" x2="224.0" y1="10" y2="248" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="224.0" y="266" text-anchor="middle">80%</text>
    <line x1="318.0" x2="318.0" y1="10" y2="248" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="318.0" y="266" text-anchor="middle">85%</text>
    <line x1="412.0" x2="412.0" y1="10" y2="248" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="412.0" y="266" text-anchor="middle">90%</text>
    <line x1="506.0" x2="506.0" y1="10" y2="248" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="506.0" y="266" text-anchor="middle">95%</text>
    <line x1="600.0" x2="600.0" y1="10" y2="248" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="600.0" y="266" text-anchor="middle">100%</text>
    <text class="lab" x="118" y="31" text-anchor="end">build</text>
    <line x1="366.9" x2="366.9" y1="17" y2="37" stroke="var(--ink)" stroke-width="2"/>
    <line x1="239.0" x2="340.6" y1="27" y2="27" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="239.0" cy="27" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="340.6" cy="27" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="65" text-anchor="end">linux</text>
    <line x1="436.4" x2="436.4" y1="51" y2="71" stroke="var(--ink)" stroke-width="2"/>
    <line x1="430.8" x2="442.1" y1="61" y2="61" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="430.8" cy="61" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="442.1" cy="61" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="99" text-anchor="end">hpc</text>
    <line x1="522.9" x2="522.9" y1="85" y2="105" stroke="var(--ink)" stroke-width="2"/>
    <line x1="519.2" x2="541.7" y1="95" y2="95" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="519.2" cy="95" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="541.7" cy="95" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="133" text-anchor="end">ml-systems</text>
    <line x1="496.6" x2="496.6" y1="119" y2="139" stroke="var(--ink)" stroke-width="2"/>
    <line x1="385.7" x2="489.1" y1="129" y2="129" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="385.7" cy="129" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="489.1" cy="129" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="167" text-anchor="end">gromacs</text>
    <line x1="451.5" x2="451.5" y1="153" y2="173" stroke="var(--ink)" stroke-width="2"/>
    <line x1="376.3" x2="434.6" y1="163" y2="163" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="376.3" cy="163" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="434.6" cy="163" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="201" text-anchor="end">slurm-admin</text>
    <line x1="442.1" x2="442.1" y1="187" y2="207" stroke="var(--ink)" stroke-width="2"/>
    <line x1="340.6" x2="421.4" y1="197" y2="197" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="340.6" cy="197" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="421.4" cy="197" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="235" text-anchor="end">kubernetes</text>
    <line x1="507.9" x2="507.9" y1="221" y2="241" stroke="var(--ink)" stroke-width="2"/>
    <line x1="453.4" x2="502.2" y1="231" y2="231" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="453.4" cy="231" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="502.2" cy="231" r="5" fill="var(--hl)"/>
  </svg>
  <figcaption>Mistral Large 4 per category: hollow dot at 4,096 output tokens, filled dot at 16,384. The black tick is the best frontier model in that category, at 4,096.</figcaption>
</figure>

The extra room helps most where the problems are long and tangled: C/C++ builds gain 5.4
points, ML systems 5.5, Slurm administration 4.3. Linux services and HPC hardly move.

Is 92.1 a fair comparison? Mostly. Kimi K3 never truncated, so a bigger budget wouldn't change
its score. GLM-5.3 and GLM-5.3-Flash truncated 58 and 46 answers; if every one of those became a
pass they'd gain 1.3 and 1.0 points, and in practice less. My best guess is that with a generous budget the top four are
within about a point of each other, and Mistral is one of them. The leaderboard row stays at 88.6,
because changing the rules for one model is how leaderboards stop meaning anything.

I've been here before. GLM-5.3 at its default maximum effort scored 53.3 on its first run,
because it reasoned through the whole budget on 45% of the problems. The lesson is the same: a
score full of truncated answers is measuring the budget, not the model. Mistral's case is milder
and it doesn't go away by picking a lower effort, because low is already the floor.

## Category by category

Mean task success per category (the mean of that category's benchmarks):

| model | build | linux | hpc | ml-systems | gromacs | slurm-admin | kubernetes |
|---|---:|---:|---:|---:|---:|---:|---:|
| Kimi K3 ‡ | 87.6 | 91.3 | 94.5 | 92.4 | 92.1 | 89.5 | 95.1 |
| GLM-5.3 | 84.8 | 90.3 | 95.9 | 94.5 | 92.1 | 90.9 | 91.8 |
| GLM-5.3-Flash | 85.2 | 90.7 | 93.1 | 92.2 | 90.5 | 91.6 | 92.8 |
| **Mistral Large 4, 16k †** | 86.2 | 91.6 | 96.9 | 94.1 | 91.2 | 90.5 | 94.8 |
| **Mistral Large 4** | 80.8 | 91.0 | 95.7 | 88.6 | 88.1 | 86.2 | 92.2 |
| DeepSeek-V4.1-Flash | 84.8 | 86.2 | 93.7 | 83.8 | 87.1 | 89.0 | 93.1 |
| MiMo-V2.6-Pro | 81.7 | 86.8 | 92.0 | 90.0 | 86.7 | 90.1 | 89.5 |
| MiMo-V2.6-Flash | 72.3 | 82.0 | 88.2 | 86.4 | 81.6 | 84.8 | 90.2 |
| Muse-Spark-Open | 76.8 | 91.0 | 89.1 | 71.0 | 87.6 | 71.6 | 85.3 |
| Qwen3.8-27B + LoRA, thinking * | 83.9 | 94.9 | 96.6 | 88.7 | 82.9 | 90.3 | 93.3 |
| Qwen3.8-27B + LoRA | 80.1 | 97.4 | 97.2 | 88.4 | 80.6 | 88.4 | 92.7 |
| Nemotron Super VL + LoRA | 84.8 | 92.9 | 95.6 | 87.3 | 81.9 | 87.4 | 91.9 |

The ml-systems column covers only the 269 of 530 problems that can be validated without a GPU.

Even at 4,096 tokens Mistral is second among the frontier models on HPC, a fifth of a point behind
GLM-5.3, and ties Muse for second on Linux services. The 27B expert still beats it on both. That's the point of
training one, so I'm glad it holds. Where the expert can't keep up is GROMACS and C/C++
builds, and that gap doesn't change with Mistral in the picture.

One number I like: on the held-out Slurm set, the four policy-heavy clusters that no expert was
trained on, Mistral scores 92.3 at 4,096 tokens and 95.7 at 16,384. The best before was Kimi K3
at 94.7, on a benchmark where Kimi had solved every problem when it was frozen.

That's the other thing worth saying about Mistral's row. It played no part in building
doctor-bench. Kimi K3 picked and solved part of the problems and wrote the training data for my
experts, so its row carries an asterisk. Mistral's doesn't.

## Update, 1 November: more effort, and a repeat run

I left Mistral running overnight with three more configurations. All of them are side runs; the
leaderboard row is still 88.6.

| run | budget | truncated | mean | build | ml-systems | linux |
|---|---:|---:|---:|---:|---:|---:|
| `reasoning_effort: low` | 4,096 | 248 | 88.6 | 80.8 | 88.6 | 91.0 |
| `reasoning_effort: low`, repeat | 4,096 | 274 | 88.8 | 81.3 | 89.5 | 89.7 |
| `reasoning_effort: low` | 16,384 | 56 | 92.1 | 86.2 | 94.1 | 91.6 |
| `reasoning_effort: medium` | 32,768 | 73 | 92.0 | 87.0 | 95.2 | 89.7 |
| `reasoning_effort: high` | 32,768 | 78 | 92.6 | 91.2 | 94.0 | 89.1 |

High effort, which is Mistral's default, scores 92.6. That's the best mean I've measured on
doctor-bench. Nearly all the gain over low effort with a big budget is in C/C++ builds, which go
from 86.2 to 91.2. Medium effort buys nothing over low with room to finish. Linux services get
slightly worse with more reasoning, not better. Keep in mind this is thinking mode at 32,768
tokens, so it belongs next to the other thinking rows (the best of those is my Qwen3.8 expert at
89.8), not next to Kimi K3's 91.7 with reasoning off.

The repeat run was the most useful one. Same endpoint, same settings, greedy decoding, and the
mean moved by only 0.15. But 329 of the 4,722 problems flipped between pass and fail, 168 one way
and 161 the other, and single benchmarks moved by up to 3.3 points. My guess is the shared
endpoint: other users' requests change how the server batches mine, and it also runs speculative
decoding. "Greedy" doesn't mean "repeatable" there. So read every single-run number on this blog,
including the frontier rows, as good to roughly ±0.2 on the mean and ±3 on any one benchmark.
Gaps of a few tenths between models are noise.

## How it was run

Mistral Large 4 ran from the
[NVFP4 checkpoint](https://huggingface.co/mistralai/Mistral-Large-4-1T-A52B-NVFP4) on a
long-running vLLM endpoint on half of an 8×B300 node: tensor parallelism 4, the
`vllm/vllm-openai:mistral-large-4` image, Mistral's reasoning parser, and its built-in DSpark
draft head for speculative decoding. Mistral says the NVFP4 checkpoint performs like the full
one. I haven't checked that myself, and Muse-Spark-Open also ran in NVFP4.

The settings were the same as for everyone else: greedy decoding, one attempt, the answer forced
to the JSON schema, and the edits applied and checked by the real build and tests in a Docker
container with no network.

One snag: in Mistral's tokenizer mode, vLLM rejects any request that carries
`chat_template_kwargs`, with an HTTP 400. My harness always sent a thinking switch that way, so
it needed a flag to leave it out. The JSON schema itself works fine alongside the reasoning
parser, which wasn't true for Muse.

| run | budget | reasoning | truncated | mean |
|---|---:|---|---:|---:|
| Mistral Large 4 | 4,096 | `reasoning_effort: low` | 248 / 4,722 | 88.6 |
| Mistral Large 4 † | 16,384 | `reasoning_effort: low` | 56 / 4,722 | 92.1 |
| Mistral Large 4 † | 32,768 | `reasoning_effort: medium` | 73 / 4,722 | 92.0 |
| Mistral Large 4 † | 32,768 | `reasoning_effort: high` | 78 / 4,722 | 92.6 |
