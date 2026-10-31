---
title: 'Seven open frontier models on doctor-bench'
description: 'Kimi K3, GLM-5.3, DeepSeek, MiMo and Muse on 14 debugging benchmarks where a fix only counts if the build and tests pass. Plus a 4B model for scale.'
publishDate: 2026-10-16
tags:
  - ai
  - llm
  - benchmarking
  - gpu
---

<style>
.db {
  --acc: #2f6fd6;
  --mut: #b9b7ad;
  --base: #dedcd4;
  --grid: hsl(var(--border));
  --ink: hsl(var(--foreground));
  --dim: hsl(var(--muted-foreground));
  margin: 2rem 0;
}
.dark .db { --acc: #6b9cf0; --mut: #6f6d66; --base: #4a4944; }
.db svg { width: 100%; height: auto; display: block; overflow: visible; }
.db figcaption { font-size: .875rem; line-height: 1.55; color: var(--dim); margin-top: .8rem; }
.db .lab { font-size: 12px; fill: var(--ink); }
.db .val { font-size: 12px; font-weight: 600; fill: var(--ink); }
.db .ax  { font-size: 11px; fill: var(--dim); }
</style>

I pointed seven open-weight frontier models at
[doctor-bench](/blog/introducing-doctor-bench), the suite of broken setups I use
to evaluate small troubleshooting models. Each answer is a set of edits. They
get applied, the setup is rebuilt and tested, and the answer counts only if the
tests pass.

The best models fix about 92% of the problems. A 4B model fixes half.

<figure class="db">
  <svg viewBox="0 0 640 296" role="img" aria-label="Mean task success over 14 doctor-bench benchmarks: Kimi K3 91.7 percent; GLM-5.3 91.5 percent; GLM-5.3-Flash 90.9 percent; DeepSeek-V4.1-Flash 88.4 percent; MiMo-V2.6-Pro 88.3 percent; MiMo-V2.6-Flash 83.9 percent; Muse-Spark-Open 80.4 percent; Qwen3-4B (base) 51.3 percent.">
    <line x1="170" x2="170" y1="22" y2="264" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="170" y="282" text-anchor="middle">0%</text>
    <line x1="270" x2="270" y1="22" y2="264" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="270" y="282" text-anchor="middle">25%</text>
    <line x1="370" x2="370" y1="22" y2="264" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="370" y="282" text-anchor="middle">50%</text>
    <line x1="470" x2="470" y1="22" y2="264" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="470" y="282" text-anchor="middle">75%</text>
    <line x1="570" x2="570" y1="22" y2="264" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="570" y="282" text-anchor="middle">100%</text>
    <text class="lab" x="160" y="43" text-anchor="end">Kimi K3 ‡</text>
    <rect x="170" y="30" width="366.7" height="18" rx="2" fill="var(--acc)"/>
    <text class="val" x="542.7" y="43">91.7</text>
    <text class="lab" x="160" y="73" text-anchor="end">GLM-5.3</text>
    <rect x="170" y="60" width="366.1" height="18" rx="2" fill="var(--acc)"/>
    <text class="val" x="542.1" y="73">91.5</text>
    <text class="lab" x="160" y="103" text-anchor="end">GLM-5.3-Flash</text>
    <rect x="170" y="90" width="363.7" height="18" rx="2" fill="var(--acc)"/>
    <text class="val" x="539.7" y="103">90.9</text>
    <text class="lab" x="160" y="133" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <rect x="170" y="120" width="353.8" height="18" rx="2" fill="var(--mut)"/>
    <text class="val" x="529.8" y="133">88.4</text>
    <text class="lab" x="160" y="163" text-anchor="end">MiMo-V2.6-Pro</text>
    <rect x="170" y="150" width="353.4" height="18" rx="2" fill="var(--mut)"/>
    <text class="val" x="529.4" y="163">88.3</text>
    <text class="lab" x="160" y="193" text-anchor="end">MiMo-V2.6-Flash</text>
    <rect x="170" y="180" width="335.4" height="18" rx="2" fill="var(--mut)"/>
    <text class="val" x="511.4" y="193">83.9</text>
    <text class="lab" x="160" y="223" text-anchor="end">Muse-Spark-Open</text>
    <rect x="170" y="210" width="321.5" height="18" rx="2" fill="var(--mut)"/>
    <text class="val" x="497.5" y="223">80.4</text>
    <text class="lab" x="160" y="253" text-anchor="end">Qwen3-4B (base)</text>
    <rect x="170" y="240" width="205.1" height="18" rx="2" fill="var(--base)"/>
    <text class="val" x="381.1" y="253">51.3</text>
  </svg>
  <figcaption>Mean task success over the 14 canonical benchmarks. One greedy attempt per problem, 4,096 output tokens. ‡ Kimi K3 is also the teacher behind part of the benchmark (see below).</figcaption>
</figure>

Kimi K3, GLM-5.3 and GLM-5.3-Flash are within a point of each other. The one
that surprised me is GLM-5.3-Flash, which gives up only 0.6 points to its
bigger sibling. DeepSeek-V4.1-Flash and MiMo-V2.6-Pro tie a little further
back. The newly released Muse-Spark-Open comes last of the seven, at 80.4.

## Where they differ

The means hide most of the interesting variation. Per category (the mean of
that category's benchmarks):

| model | build | linux | hpc | ml-systems | gromacs | slurm-admin | kubernetes |
|---|---:|---:|---:|---:|---:|---:|---:|
| Kimi K3 | 87.6 | 91.3 | 94.5 | 92.4 | 92.1 | 89.5 | 95.1 |
| GLM-5.3 | 84.8 | 90.3 | 95.9 | 94.5 | 92.1 | 90.9 | 91.8 |
| GLM-5.3-Flash | 85.2 | 90.7 | 93.1 | 92.2 | 90.5 | 91.6 | 92.8 |
| DeepSeek-V4.1-Flash | 84.8 | 86.2 | 93.7 | 83.8 | 87.1 | 89.0 | 93.1 |
| MiMo-V2.6-Pro | 81.7 | 86.8 | 92.0 | 90.0 | 86.7 | 90.1 | 89.5 |
| MiMo-V2.6-Flash | 72.3 | 82.0 | 88.2 | 86.4 | 81.6 | 84.8 | 90.2 |
| Muse-Spark-Open | 76.8 | 91.0 | 89.1 | 71.0 | 87.6 | 71.6 | 85.3 |
| Qwen3-4B (base) | 31.5 | 56.6 | 61.5 | 42.0 | 44.5 | 57.4 | 65.2 |

The ml-systems column only covers the 269 of 530 problems that can be
validated without a GPU, since the model being scored was using all eight.

C/C++ builds are the hardest category for everyone. Nobody reaches 90 there.
Kubernetes is the easiest.

Muse is the odd one out. On the Linux services it is as good as anyone (91.0),
and it holds up on GROMACS. Then it falls to about 71 on ML systems and Slurm
administration, and to 64.7 on the held-out Slurm set, where Kimi K3 gets 94.7.
My first guess was truncation. Wrong: Muse ran out of tokens on 2 problems out
of 4,722. Its patches apply, and then the build or the tests fail. It's just
weaker in those domains.

## Getting "thinking off" right was half the work

Every model ran the same way: greedy decoding, thinking off, 4,096 tokens for
the answer. That's plenty for a JSON object with a few edits in it, and it's
what I give the small models. Two of the seven fought me on it.

GLM-5.3 can't turn thinking off. Its chat template ignores `enable_thinking`,
always opens a `<think>` block, and sets `reasoning_effort` to max unless told
otherwise. On my first run it reasoned through the entire budget on 45% of the
problems and averaged 53.3, about the same as the 4B model. Yet 99% of the
answers it did finish were correct. A test prompt on GLM-5.3-Flash used 822
reasoning tokens at the default and about 30 at `reasoning_effort: low`. Rerun
at low effort, GLM-5.3 truncated 1.2% of its answers and scored 91.5.

Same model, 38 points apart, and all of it was the token budget.

So both GLM rows use low effort, the nearest thing GLM has to the mode the
others ran in. I kept the max-effort run in the results as a reminder to
myself: a low score full of truncated answers measures the budget, not the
model.

Muse had the opposite problem. Its recipe serves it with
`--reasoning-parser muse`. On the recommended vLLM image that parser failed to
set up its reasoning tokens, and after that the server quietly stopped
enforcing the JSON schema. All 1,200 answers in that run came back in a shape
the harness couldn't use. Without the parser the schema applies from the first
token, so Muse answers with no thinking at all, like the rest. That's how it
was scored.

For everyone else I sent one test request before the full run and looked at
the reported reasoning tokens. MiMo, DeepSeek and Kimi all came back with 0.

## The teacher caveat

Kimi K3 isn't an independent row, and I'd rather say so up front. It's the
teacher that wrote the reference solutions my expert models train on. A
problem also only gets into a benchmark if some teacher solved it, which is
how I know it can be solved from the information given.

For all of the build benchmarks and the held-out Slurm set, that teacher was
Kimi K3. Same for roughly half of the GROMACS, Slurm and Kubernetes v2
problems. None of the linux, HPC or ML problems came from Kimi.

The mixed benchmarks let me test for a home advantage. Here is task success
on the problems Kimi solved when the benchmark was frozen, against the rest:

| benchmark | Kimi K3 | GLM-5.3 | GLM-5.3-Flash | DeepSeek-V4.1-Flash |
|---|---|---|---|---|
| gromacs-v2 | 89.3 / 94.1 | 87.0 / 94.1 | 89.3 / 89.3 | 80.9 / 87.6 |
| gromacs-indist-v2 | 92.7 / 91.5 | 92.7 / 93.9 | 92.7 / 90.2 | 90.8 / 87.8 |
| slurm-v2 | 92.9 / 86.4 | 91.8 / 86.4 | 94.0 / 86.4 | 91.8 / 83.9 |
| slurm-indist-v2 | 84.1 / 83.0 | 94.2 / 83.0 | 94.2 / 84.9 | 88.4 / 81.1 |
| k8s-v2 | 98.6 / 96.8 | 92.3 / 93.6 | 95.1 / 94.3 | 93.7 / 94.9 |
| k8s-indist-v2 | 93.9 / 91.2 | 88.5 / 92.9 | 90.1 / 92.0 | 93.1 / 90.3 |

I don't see a pattern. On slurm-v2 the Kimi-solved half is easier, but it's
easier for every model. Kimi's lead over GLM-5.3 on its own problems swings
between −10.1 and +6.3 points. On the benchmarks it never touched it scores
88.9 to 96.0, right next to GLM-5.3.

What I can't rule out is a small edge on the three benchmarks that are
entirely Kimi-solved, because there's nothing to compare against there. Read
its row with that in mind.

## And the small models?

This suite exists to measure 4B experts, so here's where one of them stands.
The Slurm administration category has a held-out benchmark, eval3, built from
four policy-heavy clusters that no expert has trained on: a hospital, a chip
design shop, a HEP group sharing GPUs and a teaching cluster.

Base Qwen3-4B scores 46.0. My previous Slurm expert scored around 40, which is
*worse* than base. The new one was trained on a bank of harder problems and
scores between 49.3 and 53.3 across three training runs, 51.8 on average.

That's real progress. It's also 13 points behind the weakest frontier model
and 43 behind the best.

## How it was run

The frontier models ran on one 8×B300 node with vLLM, tensor parallelism 8,
each model's published recipe and a pinned image, one model at a time. Kimi K3
was scored on its long-running endpoint with the same request settings. Qwen3-4B
and its experts ran on an 8×H200 node with the same harness.

Each answer got one greedy attempt and 4,096 tokens, with the output forced to
the answer schema. The edits were applied and the setup's build and
`tests/run.sh` ran in a Docker container with no network.

After the broken Muse run I added a smoke test: 5 problems, and at least 3
answers have to reach the validator before the full run is allowed to start.
That run had recorded nothing but harness failures.

| model | checkpoint | reasoning | truncated |
|---|---|---|---:|
| Kimi K3 | moonshotai/Kimi-K3 (MXFP4) | off | 0 / 4,722 |
| GLM-5.3 | zai-org/GLM-5.3 | `reasoning_effort: low` | 58 / 4,722 |
| GLM-5.3-Flash | zai-org/GLM-5.3-Flash | `reasoning_effort: low` | 46 / 4,722 |
| DeepSeek-V4.1-Flash | deepseek-ai/DeepSeek-V4.1-Flash | off | 11 / 4,722 |
| MiMo-V2.6-Pro | XiaomiMiMo/MiMo-V2.6-Pro-RL | off | 7 / 4,722 |
| MiMo-V2.6-Flash | XiaomiMiMo/MiMo-V2.6-Flash-RL | off | 4 / 4,722 |
| Muse-Spark-Open | meta-models/Muse-Spark-Open-816B-A42B-NVFP4 | off, no reasoning parser | 2 / 4,722 |
