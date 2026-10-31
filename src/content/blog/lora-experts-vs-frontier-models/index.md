---
title: 'A 27B model with one LoRA vs the frontier on doctor-bench'
description: 'One LoRA adapter on all seven doctor-bench domains takes Qwen3.8-27B to 89.8, past DeepSeek-V4.1-Flash. Nemotron and Kolibri-1 for comparison.'
publishDate: 2026-10-31
tags:
  - ai
  - llm
  - benchmarking
  - fine-tuning
---

<style>
.lx {
  --acc: #2f6fd6;
  --mut: #b9b7ad;
  --base: #dedcd4;
  --grid: hsl(var(--border));
  --ink: hsl(var(--foreground));
  --dim: hsl(var(--muted-foreground));
  margin: 2rem 0;
}
.dark .lx { --acc: #6b9cf0; --mut: #6f6d66; --base: #4a4944; }
.lx svg { width: 100%; height: auto; display: block; overflow: visible; }
.lx figcaption { font-size: .875rem; line-height: 1.55; color: var(--dim); margin-top: .8rem; }
.lx .lab { font-size: 12px; fill: var(--ink); }
.lx .val { font-size: 12px; font-weight: 600; fill: var(--ink); }
.lx .ax  { font-size: 11px; fill: var(--dim); }
</style>

In the [first round of results](/blog/frontier-models-on-doctor-bench) the open frontier models
scored 80–92% on [doctor-bench](/blog/introducing-doctor-bench), the suite where a model fixes a
broken setup and only the real build and tests decide whether it worked. The question I actually
care about is a different one: how close can a model you can run on one or two GPUs get, if you
teach it the job?

So I trained one LoRA adapter on all seven domains at once and put it on three open models:
Qwen3.8-27B, NVIDIA's Nemotron-3.5 Super VL (120B total, 12B active) and Aleph Alpha's Kolibri-1
(78B total, 3.5B active). The short answer: Qwen3.8-27B with the adapter scores 89.8, ahead of
DeepSeek-V4.1-Flash and MiMo-V2.6-Pro and about a point behind GLM-5.3-Flash.

<figure class="lx">
  <svg viewBox="0 0 640 448" role="img" aria-label="Mean task success over 14 doctor-bench benchmarks: Kimi K3 ‡ 91.7; GLM-5.3 91.5; GLM-5.3-Flash 90.9; Qwen3.8-27B + LoRA, thinking 89.8; Qwen3.8-27B + LoRA 88.6; DeepSeek-V4.1-Flash 88.4; Nemotron Super VL + LoRA 88.4; MiMo-V2.6-Pro 88.3; Nemotron Super VL, thinking 84.2; MiMo-V2.6-Flash 83.9; Qwen3.8-27B, thinking 83.6; Qwen3.8-27B 82.0; Muse-Spark-Open 80.4; Kolibri-1 + LoRA 78.9; Nemotron Super VL 78.7; Kolibri-1 60.5; Qwen3-4B 51.3.">
    <line x1="230" x2="230" y1="8" y2="416" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="230" y="434" text-anchor="middle">0%</text>
    <line x1="322" x2="322" y1="8" y2="416" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="322" y="434" text-anchor="middle">25%</text>
    <line x1="415" x2="415" y1="8" y2="416" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="415" y="434" text-anchor="middle">50%</text>
    <line x1="508" x2="508" y1="8" y2="416" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="508" y="434" text-anchor="middle">75%</text>
    <line x1="600" x2="600" y1="8" y2="416" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="600" y="434" text-anchor="middle">100%</text>
    <text class="lab" x="220" y="26" text-anchor="end">Kimi K3 ‡</text>
    <rect x="230" y="14" width="339.3" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="575.3" y="26">91.7</text>
    <text class="lab" x="220" y="50" text-anchor="end">GLM-5.3</text>
    <rect x="230" y="38" width="338.5" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="574.5" y="50">91.5</text>
    <text class="lab" x="220" y="74" text-anchor="end">GLM-5.3-Flash</text>
    <rect x="230" y="62" width="336.3" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="572.3" y="74">90.9</text>
    <text class="lab" x="220" y="98" text-anchor="end">Qwen3.8-27B + LoRA, thinking</text>
    <rect x="230" y="86" width="332.3" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="568.3" y="98">89.8</text>
    <text class="lab" x="220" y="122" text-anchor="end">Qwen3.8-27B + LoRA</text>
    <rect x="230" y="110" width="327.8" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="563.8" y="122">88.6</text>
    <text class="lab" x="220" y="146" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <rect x="230" y="134" width="327.1" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="563.1" y="146">88.4</text>
    <text class="lab" x="220" y="170" text-anchor="end">Nemotron Super VL + LoRA</text>
    <rect x="230" y="158" width="327.1" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="563.1" y="170">88.4</text>
    <text class="lab" x="220" y="194" text-anchor="end">MiMo-V2.6-Pro</text>
    <rect x="230" y="182" width="326.7" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="562.7" y="194">88.3</text>
    <text class="lab" x="220" y="218" text-anchor="end">Nemotron Super VL, thinking</text>
    <rect x="230" y="206" width="311.5" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="547.5" y="218">84.2</text>
    <text class="lab" x="220" y="242" text-anchor="end">MiMo-V2.6-Flash</text>
    <rect x="230" y="230" width="310.4" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="546.4" y="242">83.9</text>
    <text class="lab" x="220" y="266" text-anchor="end">Qwen3.8-27B, thinking</text>
    <rect x="230" y="254" width="309.3" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="545.3" y="266">83.6</text>
    <text class="lab" x="220" y="290" text-anchor="end">Qwen3.8-27B</text>
    <rect x="230" y="278" width="303.4" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="539.4" y="290">82.0</text>
    <text class="lab" x="220" y="314" text-anchor="end">Muse-Spark-Open</text>
    <rect x="230" y="302" width="297.5" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="533.5" y="314">80.4</text>
    <text class="lab" x="220" y="338" text-anchor="end">Kolibri-1 + LoRA</text>
    <rect x="230" y="326" width="291.9" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="527.9" y="338">78.9</text>
    <text class="lab" x="220" y="362" text-anchor="end">Nemotron Super VL</text>
    <rect x="230" y="350" width="291.2" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="527.2" y="362">78.7</text>
    <text class="lab" x="220" y="386" text-anchor="end">Kolibri-1</text>
    <rect x="230" y="374" width="223.9" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="459.9" y="386">60.5</text>
    <text class="lab" x="220" y="410" text-anchor="end">Qwen3-4B</text>
    <rect x="230" y="398" width="189.8" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="425.8" y="410">51.3</text>
  </svg>
  <figcaption>Mean task success over the 14 doctor-bench benchmarks. Grey: frontier models (thinking off). Light: base models. Blue: the same base models with the global LoRA adapter. Thinking off unless the label says otherwise. ‡ Kimi K3 is also the teacher that wrote the training data.</figcaption>
</figure>

## What was trained

One adapter per model, on exactly the same data: 20,269 problems, the current training set of each
domain (C/C++ builds, Linux services, HPC jobs, ML deployments, GROMACS, Slurm administration,
Kubernetes). Every example is a broken setup plus a fix that a large teacher model wrote and the
sandbox confirmed. No benchmark problem is in it; the transfer benchmarks use whole projects the
adapter never saw.

One epoch, answers only (no reasoning traces), one random seed. LoRA rank 8 for Qwen3.8 and
Kolibri-1, rank 32 from NVIDIA's own recipe for Nemotron. Training took 14 hours for Qwen3.8 on four
B300s, under 5 hours for Nemotron on eight, and about four and a half for Kolibri-1 on two.

## Every base model improves

<figure class="lx">
  <svg viewBox="0 0 640 148" role="img" aria-label="Thinking off, base to LoRA global expert: Qwen3.8-27B 82.0 to 88.6; Nemotron Super VL 78.7 to 88.4; Kolibri-1 60.5 to 78.9.">
    <line x1="170" x2="170" y1="10" y2="110" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="170" y="128" text-anchor="middle">50%</text>
    <line x1="250" x2="250" y1="10" y2="110" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="250" y="128" text-anchor="middle">60%</text>
    <line x1="330" x2="330" y1="10" y2="110" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="330" y="128" text-anchor="middle">70%</text>
    <line x1="410" x2="410" y1="10" y2="110" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="410" y="128" text-anchor="middle">80%</text>
    <line x1="490" x2="490" y1="10" y2="110" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="490" y="128" text-anchor="middle">90%</text>
    <line x1="570" x2="570" y1="10" y2="110" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="570" y="128" text-anchor="middle">100%</text>
    <text class="lab" x="158" y="26" text-anchor="end">Qwen3.8-27B</text>
    <line x1="426.0" x2="478.8" y1="22" y2="22" stroke="var(--acc)" stroke-width="2"/>
    <circle cx="426.0" cy="22" r="5" fill="var(--mut)"/>
    <circle cx="478.8" cy="22" r="5" fill="var(--acc)"/>
    <text class="ax" x="418.0" y="26" text-anchor="end">82.0</text>
    <text class="val" x="487.8" y="26">88.6 (+6.6)</text>
    <text class="lab" x="158" y="60" text-anchor="end">Nemotron Super VL</text>
    <line x1="399.6" x2="477.2" y1="56" y2="56" stroke="var(--acc)" stroke-width="2"/>
    <circle cx="399.6" cy="56" r="5" fill="var(--mut)"/>
    <circle cx="477.2" cy="56" r="5" fill="var(--acc)"/>
    <text class="ax" x="391.6" y="60" text-anchor="end">78.7</text>
    <text class="val" x="486.2" y="60">88.4 (+9.7)</text>
    <text class="lab" x="158" y="94" text-anchor="end">Kolibri-1</text>
    <line x1="254.0" x2="401.2" y1="90" y2="90" stroke="var(--acc)" stroke-width="2"/>
    <circle cx="254.0" cy="90" r="5" fill="var(--mut)"/>
    <circle cx="401.2" cy="90" r="5" fill="var(--acc)"/>
    <text class="ax" x="246.0" y="94" text-anchor="end">60.5</text>
    <text class="val" x="410.2" y="94">78.9 (+18.4)</text>
  </svg>
  <figcaption>Thinking off, 14-benchmark mean: base model (grey) and with the global adapter (blue).</figcaption>
</figure>

The weaker the base, the bigger the jump. Kolibri-1 gains 18 points and goes up on all fourteen
benchmarks. Part of that is formatting: the base model kept writing until it ran out of tokens on
303 answers, and after training only 70 did. Nemotron gains almost 10 and ends level with
DeepSeek-V4.1-Flash. Qwen3.8 starts highest and still gains 6.6.

By category (thinking off, except the rows that say otherwise):

| model | build | linux | hpc | ml | gromacs | slurm | k8s |
|---|---:|---:|---:|---:|---:|---:|---:|
| best frontier model | 87.6 | 91.3 | 95.9 | 94.5 | 92.1 | 91.6 | 95.1 |
| Qwen3.8 + LoRA, thinking | 83.9 | 94.9 | 96.6 | 88.7 | 82.9 | 90.3 | 93.3 |
| Qwen3.8 + LoRA | 80.1 | 97.4 | 97.2 | 88.4 | 80.6 | 88.4 | 92.7 |
| Nemotron + LoRA | 84.8 | 92.9 | 95.6 | 87.3 | 81.9 | 87.4 | 91.9 |
| Kolibri-1 + LoRA | 76.5 | 91.3 | 87.8 | 73.8 | 68.2 | 76.7 | 84.9 |

On Linux services and HPC the Qwen3.8 adapter beats every frontier model I tested, and
Nemotron's comes close. GROMACS is where the gap stays widest: 9 to 11 points for Qwen3.8 and
Nemotron. It is also the domain where the 16k-token limit dropped the most training examples for
those two models (131 and 143), which hardly helps.

## Thinking mode: sometimes kept, sometimes broken

All three adapters were trained on plain answers. What happens when you then switch thinking on
differs a lot between models.

Qwen3.8 keeps it. With thinking on it improves from 83.6 to 89.8, its best score.

Nemotron breaks. With thinking on the adapted model drops from 84.2 to 73.5, and on the ML
benchmarks it collapses to around 35. If you use this adapter, use it with thinking off, where it
is excellent.

Kolibri-1 I couldn't measure properly with thinking on. Even the base model ran into the
32k-token limit on almost half the problems, so that number says more about the budget than about
the model.

## Generalist vs specialist

A single global adapter doesn't beat a specialist on its own turf. On the held-out Slurm benchmark
the Qwen3.8 Slurm-only adapters score 90.3–91.0; the global one 88.3. But the specialist only
helps with Slurm, and the global adapter lifts everything at once for the same price of one
training run. If I had to deploy one thing, it would be the global adapter.

## Caveats

- One seed per global adapter. My earlier per-domain experts varied by up to four points between
  seeds, so differences of a point or two between rows here are not meaningful.
- The ML columns only include problems that can be validated without a GPU (269 of 530).
- Kimi K3 wrote part of the training data and part of the benchmark selection, so its row is not
  an independent comparison. I measured the effect in the
  [frontier post](/blog/frontier-models-on-doctor-bench) and it looked small.
- All runs use greedy decoding, one attempt, 4,096 tokens with thinking off and 32,768 with it on,
  on vLLM.

## How it was run

Qwen3.8-27B was trained with Hugging Face PEFT/TRL. The trap there was serving: vLLM accepted the
adapter and silently ignored it until the weight names were rewritten to the multimodal layout it
loads Qwen3.8 with. Nemotron went through NVIDIA's own Megatron-Bridge LoRA cookbook, adapted to a
public container and to our data. Kolibri-1 isn't supported by Megatron-Bridge yet, so it went
through transformers 5.20, which added the architecture in mid-October, with the grouped-matmul
expert kernel; only attention and the shared expert carry LoRA weights there, not the 384 routed
experts, which leaves room for a better Kolibri adapter.
