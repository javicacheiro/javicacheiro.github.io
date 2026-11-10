---
title: 'Kimi K3.1 Pro on doctor-bench'
description: 'Kimi K3.1 Pro ties its predecessor Kimi K3 under the standard rules, at 91.4. With thinking at maximum effort it scores 93.4, the best I have measured.'
publishDate: 2026-11-08
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

Kimi K3.1 Pro is the successor to Kimi K3, the model that solved part of
[doctor-bench](/blog/introducing-doctor-bench) and wrote the training data for my
[LoRA experts](/blog/lora-experts-vs-frontier-models). So this one is a bit personal. I ran it
the same way as the [frontier models](/blog/frontier-models-on-doctor-bench) and
[Mistral Large 4](/blog/mistral-large-4-on-doctor-bench): one greedy attempt per problem, and a
fix only counts if the build and tests pass afterwards.

Under the standard rules (thinking off, 4,096 output tokens) it scores 91.4, level with Kimi K3
and GLM-5.3. With thinking at maximum effort it scores 93.4, the highest mean I've measured on
doctor-bench. At the end there are also some quick throughput numbers on 8×B300 and its scores
on GPQA, AIME and IFBench.

<figure class="ml">
  <svg viewBox="0 0 640 594" role="img" aria-label="Mean task success over 14 doctor-bench benchmarks: Kimi K3.1 Pro, thinking max † 93.4; Mistral Large 4, high effort † 92.6; Kimi K3 ‡ 91.7; GLM-5.3 91.6; Kimi K3.1 Pro 91.4; GLM-5.3-Flash 90.9; Qwen3.8-27B + LoRA, thinking * 89.8; DeepSeek-V4.1-Flash-1017 89.3; Qwen3.8-27B + LoRA 88.6; Mistral Large 4 88.6; DeepSeek-V4.1-Flash 88.6; MiMo-V2.6-Pro 88.4; Nemotron Super VL + LoRA 88.4; Nemotron Super VL, thinking * 84.2; MiMo-V2.6-Flash 84.2; Qwen3.8-27B, thinking * 83.6; Qwen3.8-27B 82.0; Muse-Spark-Open 80.4; Kolibri-1 + LoRA 78.9; Nemotron Super VL 78.7; Qwen3-4B + 7 domain LoRAs 68.6; Kolibri-1 60.5; Qwen3-4B 51.3.">
    <line x1="230" x2="230" y1="8" y2="560" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="230" y="578" text-anchor="middle">0%</text>
    <line x1="322.5" x2="322.5" y1="8" y2="560" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="322.5" y="578" text-anchor="middle">25%</text>
    <line x1="415" x2="415" y1="8" y2="560" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="415" y="578" text-anchor="middle">50%</text>
    <line x1="507.5" x2="507.5" y1="8" y2="560" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="507.5" y="578" text-anchor="middle">75%</text>
    <line x1="600" x2="600" y1="8" y2="560" stroke="var(--grid)" stroke-width="1"/>
    <text class="ax" x="600" y="578" text-anchor="middle">100%</text>
    <text class="lab" x="220" y="26" text-anchor="end" style="font-weight:600">Kimi K3.1 Pro, thinking max †</text>
    <rect x="230" y="14" width="345.6" height="15" rx="2" fill="var(--hl)" fill-opacity=".45" stroke="var(--hl)" stroke-width="1.2"/>
    <text class="val" x="581.6" y="26">93.4</text>
    <text class="lab" x="220" y="50" text-anchor="end">Mistral Large 4, high effort †</text>
    <rect x="230" y="38" width="342.7" height="15" rx="2" fill="var(--mut)" fill-opacity=".45" stroke="var(--mut)" stroke-width="1.2"/>
    <text class="val" x="578.7" y="50">92.6</text>
    <text class="lab" x="220" y="74" text-anchor="end">Kimi K3 ‡</text>
    <rect x="230" y="62" width="339.2" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="575.2" y="74">91.7</text>
    <text class="lab" x="220" y="98" text-anchor="end">GLM-5.3</text>
    <rect x="230" y="86" width="338.9" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="574.9" y="98">91.6</text>
    <text class="lab" x="220" y="122" text-anchor="end" style="font-weight:600">Kimi K3.1 Pro</text>
    <rect x="230" y="110" width="338.3" height="15" rx="2" fill="var(--hl)"/>
    <text class="val" x="574.3" y="122">91.4</text>
    <text class="lab" x="220" y="146" text-anchor="end">GLM-5.3-Flash</text>
    <rect x="230" y="134" width="336.4" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="572.4" y="146">90.9</text>
    <text class="lab" x="220" y="170" text-anchor="end">Qwen3.8-27B + LoRA, thinking *</text>
    <rect x="230" y="158" width="332.4" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="568.4" y="170">89.8</text>
    <text class="lab" x="220" y="194" text-anchor="end">DeepSeek-V4.1-Flash-1017</text>
    <rect x="230" y="182" width="330.6" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="566.6" y="194">89.3</text>
    <text class="lab" x="220" y="218" text-anchor="end">Qwen3.8-27B + LoRA</text>
    <rect x="230" y="206" width="327.9" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="563.9" y="218">88.6</text>
    <text class="lab" x="220" y="242" text-anchor="end">Mistral Large 4</text>
    <rect x="230" y="230" width="327.8" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="563.8" y="242">88.6</text>
    <text class="lab" x="220" y="266" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <rect x="230" y="254" width="327.6" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="563.6" y="266">88.6</text>
    <text class="lab" x="220" y="290" text-anchor="end">MiMo-V2.6-Pro</text>
    <rect x="230" y="278" width="327.2" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="563.2" y="290">88.4</text>
    <text class="lab" x="220" y="314" text-anchor="end">Nemotron Super VL + LoRA</text>
    <rect x="230" y="302" width="327.2" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="563.2" y="314">88.4</text>
    <text class="lab" x="220" y="338" text-anchor="end">Nemotron Super VL, thinking *</text>
    <rect x="230" y="326" width="311.5" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="547.5" y="338">84.2</text>
    <text class="lab" x="220" y="362" text-anchor="end">MiMo-V2.6-Flash</text>
    <rect x="230" y="350" width="311.4" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="547.4" y="362">84.2</text>
    <text class="lab" x="220" y="386" text-anchor="end">Qwen3.8-27B, thinking *</text>
    <rect x="230" y="374" width="309.3" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="545.3" y="386">83.6</text>
    <text class="lab" x="220" y="410" text-anchor="end">Qwen3.8-27B</text>
    <rect x="230" y="398" width="303.4" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="539.4" y="410">82.0</text>
    <text class="lab" x="220" y="434" text-anchor="end">Muse-Spark-Open</text>
    <rect x="230" y="422" width="297.5" height="15" rx="2" fill="var(--mut)"/>
    <text class="val" x="533.5" y="434">80.4</text>
    <text class="lab" x="220" y="458" text-anchor="end">Kolibri-1 + LoRA</text>
    <rect x="230" y="446" width="291.8" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="527.8" y="458">78.9</text>
    <text class="lab" x="220" y="482" text-anchor="end">Nemotron Super VL</text>
    <rect x="230" y="470" width="291.2" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="527.2" y="482">78.7</text>
    <text class="lab" x="220" y="506" text-anchor="end">Qwen3-4B + 7 domain LoRAs</text>
    <rect x="230" y="494" width="253.8" height="15" rx="2" fill="var(--acc)"/>
    <text class="val" x="489.8" y="506">68.6</text>
    <text class="lab" x="220" y="530" text-anchor="end">Kolibri-1</text>
    <rect x="230" y="518" width="224.0" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="460.0" y="530">60.5</text>
    <text class="lab" x="220" y="554" text-anchor="end">Qwen3-4B</text>
    <rect x="230" y="542" width="189.7" height="15" rx="2" fill="var(--base)"/>
    <text class="val" x="425.7" y="554">51.3</text>
  </svg>
  <figcaption>Mean task success over the 14 canonical benchmarks, one greedy attempt per problem. Grey: open frontier models. Orange: Kimi K3.1 Pro. Blue: base models with my LoRA adapters. Light: the same base models without them. Every row has 4,096 output tokens except † side runs with reasoning on at 32,768 (not leaderboard rows) and * thinking mode with 32,768. ‡ Kimi K3 is also the teacher behind part of the benchmark.</figcaption>
</figure>

## Same score, different problems

Kimi K3.1 Pro and Kimi K3 land 0.2 apart on the mean, but they don't solve the same problems.
Of the 4,722 problems, 280 pass with one and fail with the other: 138 go K3.1's way and 142 go
K3's.

How much of that is just noise? I ran K3.1 a second time with the same settings. The mean came
out at 92.1 instead of 91.4, and 156 problems flipped, 93 up and 63 down. That's a bigger shift
than Mistral's repeat run (0.15), and probably not all of it is noise. Between the two runs the
endpoint was restarted several times after hangs, and the instance that served the repeat used
a different all-reduce path (plain NCCL instead of the recipe's custom kernels) and had one
kernel fusion (all-reduce + RMSNorm) turned off. The weights are the same, but the
arithmetic isn't bit-for-bit identical. So the honest summary is a
range: 91.4 and 92.1 on two runs, with Kimi K3's 91.7 in between. The two Kimis are a tie.

Category by category, the differences that hold up in both runs are small. K3.1 is ahead on
Linux services (+1.3 and +3.2) and ML systems (+2.5 and +1.1), and about a point ahead on
GROMACS. K3 stays ahead on C/C++ builds, by 2.0 and 0.8 points. The first run's Kubernetes gap
(−3.1) mostly vanished in the repeat (−0.3), so I wouldn't read anything into it.

Neither run truncated anything to speak of: 0 and 3 answers ran out of room at 4,096 tokens.
That's unlike Mistral, whose low-effort reasoning still ran out of budget on 5% of the problems.

## Thinking at maximum effort

Kimi K3.1 Pro has a real off switch for reasoning, and a `reasoning_effort` setting when it's on.
I ran the whole suite once more with thinking on, `reasoning_effort: max` and 32,768 output
tokens. It scored 93.4, against 92.1 for the thinking-off repeat on the same server.

<figure class="ml">
  <svg viewBox="0 0 640 282" role="img" aria-label="Kimi K3.1 Pro per category, thinking off then thinking at max effort, with Kimi K3 for reference: build: 86.8 to 87.4, Kimi K3 87.6; linux: 94.5 to 95.2, Kimi K3 91.3; hpc: 94.5 to 97.5, Kimi K3 94.5; ml-systems: 93.5 to 96.5, Kimi K3 92.4; gromacs: 93.0 to 96.3, Kimi K3 92.1; slurm-admin: 89.9 to 92.4, Kimi K3 89.5; kubernetes: 94.8 to 89.8, Kimi K3 95.1.">
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
    <line x1="351.8" x2="363.1" y1="27" y2="27" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="351.8" cy="27" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="363.1" cy="27" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="65" text-anchor="end">linux</text>
    <line x1="436.4" x2="436.4" y1="51" y2="71" stroke="var(--ink)" stroke-width="2"/>
    <line x1="496.6" x2="509.8" y1="61" y2="61" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="496.6" cy="61" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="509.8" cy="61" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="99" text-anchor="end">hpc</text>
    <line x1="496.6" x2="496.6" y1="85" y2="105" stroke="var(--ink)" stroke-width="2"/>
    <line x1="496.6" x2="553.0" y1="95" y2="95" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="496.6" cy="95" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="553.0" cy="95" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="133" text-anchor="end">ml-systems</text>
    <line x1="457.1" x2="457.1" y1="119" y2="139" stroke="var(--ink)" stroke-width="2"/>
    <line x1="477.8" x2="534.2" y1="129" y2="129" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="477.8" cy="129" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="534.2" cy="129" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="167" text-anchor="end">gromacs</text>
    <line x1="451.5" x2="451.5" y1="153" y2="173" stroke="var(--ink)" stroke-width="2"/>
    <line x1="468.4" x2="530.4" y1="163" y2="163" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="468.4" cy="163" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="530.4" cy="163" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="201" text-anchor="end">slurm-admin</text>
    <line x1="402.6" x2="402.6" y1="187" y2="207" stroke="var(--ink)" stroke-width="2"/>
    <line x1="410.1" x2="457.1" y1="197" y2="197" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="410.1" cy="197" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="457.1" cy="197" r="5" fill="var(--hl)"/>
    <text class="lab" x="118" y="235" text-anchor="end">kubernetes</text>
    <line x1="507.9" x2="507.9" y1="221" y2="241" stroke="var(--ink)" stroke-width="2"/>
    <line x1="502.2" x2="408.2" y1="231" y2="231" stroke="var(--hl)" stroke-width="3" stroke-opacity=".5"/>
    <circle cx="502.2" cy="231" r="5" fill="var(--bg)" stroke="var(--hl)" stroke-width="2"/>
    <circle cx="408.2" cy="231" r="5" fill="var(--hl)"/>
  </svg>
  <figcaption>Kimi K3.1 Pro per category: hollow dot with thinking off at 4,096 output tokens (the repeat run, on the same server instance as the thinking run), filled dot with thinking at <code>reasoning_effort: max</code> and 32,768 tokens. The black tick is Kimi K3, thinking off.</figcaption>
</figure>

The gains are where the problems need the most digging. HPC goes up 3.0 points, GROMACS 3.3,
ML systems 3.0 and Slurm administration 2.5. On the held-out Slurm set, 295 of 300 now pass
(98.3). Of the problems that failed in both thinking-off runs, 129 pass with thinking. Only 25
answers ran out of room even at 32,768 tokens, 14 of them on the C/C++ build benchmark.

Kubernetes is the exception: it drops five points, from 94.8 to 89.8. Most of that comes from one
kind of rejection. The scorer checks that the answer names the fault that was actually
injected, and with thinking on, K3.1 got that wrong 50 times on the two Kubernetes benchmarks,
against 18 times without thinking (and 10 for Kimi K3). I haven't dug into the answers yet. My
guess is that it reasons its way to a deeper or different root cause than the one the problem
was built around. That's a fair failure under the benchmark's rules, but it's a different kind
of failure from a fix that doesn't build.

Thinking runs belong next to other thinking runs, not next to the 4,096-token leaderboard. The
comparison that matters here is Mistral Large 4 at high effort (92.6, also 32,768 tokens) and my
Qwen3.8-27B expert with thinking (89.8). K3.1 at max effort beats both. It also wastes less of
its budget: 25 truncations against Mistral's 78.

## Category by category

Mean task success per category (the mean of that category's benchmarks):

| model | build | linux | hpc | ml-systems | gromacs | slurm-admin | kubernetes |
|---|---:|---:|---:|---:|---:|---:|---:|
| **Kimi K3.1 Pro, thinking max †** | 87.4 | 95.2 | 97.5 | 96.5 | 96.3 | 92.4 | 89.8 |
| Mistral Large 4, high effort † | 91.2 | 89.1 | 97.0 | 94.0 | 91.9 | 89.3 | 95.9 |
| **Kimi K3.1 Pro, repeat** | 86.8 | 94.5 | 94.5 | 93.5 | 93.0 | 89.9 | 94.8 |
| **Kimi K3.1 Pro** | 85.6 | 92.6 | 94.2 | 94.9 | 93.4 | 89.1 | 92.0 |
| Kimi K3 ‡ | 87.6 | 91.3 | 94.5 | 92.4 | 92.1 | 89.5 | 95.1 |
| GLM-5.3 | 84.8 | 91.3 | 95.9 | 94.5 | 92.1 | 90.9 | 91.8 |
| GLM-5.3-Flash | 85.2 | 90.7 | 93.1 | 92.2 | 90.5 | 91.6 | 92.8 |
| DeepSeek-V4.1-Flash-1017 | 87.3 | 84.9 | 93.1 | 89.8 | 87.5 | 86.8 | 94.9 |
| Mistral Large 4 | 80.8 | 91.0 | 95.7 | 88.6 | 88.1 | 86.2 | 92.2 |
| Qwen3.8-27B + LoRA, thinking * | 83.9 | 94.9 | 96.6 | 88.7 | 82.9 | 90.3 | 93.3 |
| Qwen3.8-27B + LoRA | 80.1 | 97.4 | 97.2 | 88.4 | 80.6 | 88.4 | 92.7 |

The ml-systems column covers only the 269 of 530 problems that can be validated without a GPU.

The two models with reasoning on are good at different things. Mistral at high effort is far
ahead on C/C++ builds (91.2 against 87.4) and Kubernetes. Kimi at max effort is far ahead on
Linux services, GROMACS and Slurm. My 27B expert still holds Linux services at 97.4, with
thinking off and at a fraction of the size, and it's still well behind on GROMACS and builds.

## The family problem

Kimi K3 is not a neutral party on this benchmark. It picked and solved part of the problems,
and every answer my experts learned from was written by it. Kimi K3.1 Pro is not the same
model, but it's the same family, probably trained on similar data and with a similar way of
writing a fix. If doctor-bench has a blind spot that suits Kimi's style, K3.1 benefits from it
too.

The held-out Slurm set is where that matters most. Every problem in it was kept because Kimi K3
solved it while the benchmark was being built. Re-run under the benchmark's rules, with one
greedy attempt each, K3 scores 94.7. K3.1 at max effort passes 98.3%. Without thinking, it's at
94.3 and 94.7, the same as K3, and Mistral at 4,096 tokens is at 92.3. So treat the Kimi rows
with some caution. I think the numbers are real, but they're the ones I'd discount first.

## Throughput and standard benchmarks

I also took some quick numbers outside doctor-bench, with the whole node to myself: one pass,
no repeats, only the main workloads and benchmarks. The server was the relaunched one (plain
NCCL, fusion off).

**Throughput.** These are the synthetic workloads from my
[throughput post](/blog/throughput-benchmarking-on-b300), sent with GuideLLM exactly as for
Kimi K3. The client was on another node, so time to first token includes about 0.1 s of network.

| workload (in → out) | Kimi K3.1 Pro, peak tok/s | at concurrency | Kimi K3 | change |
|---|---:|---:|---:|---:|
| API-S (2k → 256) | 1,914 | 256 | 1,651 | +16% |
| CHAT-S (2k → 512) | 2,617 | 512 | 2,880 | −9% |
| CODE-I (16k → 2k) | 1,246 | 64 | 1,390 | −10% |
| CODE-A (64k → 4k) | 586 | 64 | 691 | −15% |
| BATCH-D (4k → 8k) | 2,252 | 256 | 4,422 | −49% |
| single stream (CHAT-S) | 85 | 1 | 102 | −17% |

Short requests come close to Kimi K3. The coding workloads stop gaining at about 64 concurrent
requests, and chat and API at 256–512. Batch generation reaches only half of K3, and that's
mostly the server rather than the model. After the weights there's room for 1.95M tokens of KV
cache, which holds only about 160 BATCH-D requests at once, and the model's state cache caps the
server at 512 sequences, where K3 peaked at 1,024. The NCCL-only all-reduce probably costs some
decode speed too, but I didn't measure that separately. One more caveat: K3's peaks are vLLM's
server-side rate and K3.1's come from the client. Where both exist for K3, they agree within
about 8%.

**Standard benchmarks.** These use the harness, prompts and settings from my
[Kolibri-1 post](/blog/kolibri-1-benchmarks), with `reasoning_effort: max`, temperature 1.0 and
top-p 0.95, which are also the model card's settings.

| model | GPQA Diamond | AIME 2025 | AIME 2026 | IFBench loose / strict | GPQA tokens per answer |
|---|---:|---:|---:|---:|---:|
| **Kimi K3.1 Pro** | **93.4** | **99.8** | **100.0** | 77.2 / 69.3 | 5,192 |
| Kimi K3 | 92.6 | 93.8 | 95.2 | 76.5 / 72.1 | 4,948 |
| DeepSeek-V4.1-Flash | 89.1 | 98.8 | 98.8 | 79.7 / 75.0 | 10,088 |
| Qwen3.8-27B | 89.3 | 97.7 | 95.6 | **81.7** / 74.4 | 11,925 |
| Mistral Large 4, high | 86.6 | 99.0 | 99.0 | 76.7 / 73.1 | 13,874 |
| Kolibri-1 | 82.6 | 95.6 | 95.2 | 78.8 / 70.3 | 15,954 |

GPQA reproduces the model card's 94.2 (our 95% interval is 90.0–96.2), and it's the best score
I've measured with this harness. But it's only 0.8 above K3, so it's within noise. What stands
out is how few tokens both Kimis use: about 5k per answer, against 10–16k for the rest. AIME is
essentially perfect, about five points above K3. AIME 2026 was published before K3.1 came out,
though, so I'd treat those scores as a ceiling rather than something that separates the models.
IFBench shows no gain: loose is level with K3, strict is 2.8 points lower, and both trail
Qwen3.8-27B and DeepSeek-V4.1-Flash.

## How it was run

Kimi K3.1 Pro ran from the official checkpoint on a long-running vLLM endpoint across a whole
8×B300 node: tensor parallelism 8, the `vllm/vllm-openai:kimi-k31` image and the `kimi_k3`
reasoning parser. The endpoint was shared with another benchmark the whole time. The first run
used the vLLM recipe as published. The next day the endpoint hung five times under load: one
GPU stalled while the other seven spun at 100%. So it was relaunched with every custom
all-reduce path turned off (plain NCCL only) and the all-reduce + RMSNorm fusion off. It then
ran for about 28 hours under shared load without a hang. The repeat and the thinking run both
used that relaunched server.

Thinking is switched on and off through the chat template (`thinking: true` or `false`), and
the effort goes in `reasoning_effort`. A probe with thinking off produced 0 reasoning tokens. On
the same short prompt with thinking on, low effort used 88 reasoning tokens, high 264 and max
287. Everything else was the same as for every other model: greedy decoding, the answer forced
to the JSON schema, and the edits applied and checked by the real build and tests in a Docker
container with no network.

| run | budget | reasoning | truncated | mean |
|---|---:|---|---:|---:|
| Kimi K3.1 Pro | 4,096 | off | 0 / 4,722 | 91.4 |
| Kimi K3.1 Pro, repeat | 4,096 | off | 3 / 4,722 | 92.1 |
| Kimi K3.1 Pro † | 32,768 | `reasoning_effort: max` | 25 / 4,722 | 93.4 |
