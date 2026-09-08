---
title: 'Measuring how much quantization affects model quality'
description: 'I benchmark different variants of Qwen3.8-27B to measure the effect of quantization'
publishDate: 2026-09-06
tags:
  - ai
  - llm
  - gpu
  - quantization
  - benchmarking
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
.qz .sub { font-size: 11px; fill: var(--dim); }
.qz .val { font-size: 12px; font-weight: 600; fill: var(--ink); }
.qz .ax  { font-size: 11px; fill: var(--dim); }
.qz .key { display: flex; gap: 1.25rem; flex-wrap: wrap; font-size: .8rem; margin-bottom: .7rem; color: var(--dim); }
.qz .key i { display: inline-block; width: 11px; height: 11px; border-radius: 2px; margin-right: .35rem; vertical-align: -1px; }
</style>

It is really fun to run models locally, testing different variants and experimenting with different configurations.

For a given model you generally have, besides the main BF16 checkpoint, several quantized checkpoints whose aim is to reduce its size and increase performance.

When running on large servers with 8xB300 GPUs and 2.1TB of HBM memory, I usually do not spend much time selecting the quantization. In these cases what I mainly look at is the tuning for throughput or latency, using tensor parallelism. When the BF16 checkpoint does not fit in memory, I just choose the FP8 or NVFP4 variants, because they have hardware support in Blackwell and so they perform really well.

But there are cases where I want to run on a more modest server, or even on a 4090 GPU. This is when having a quantized checkpoint helps a lot, because it can fit in the VRAM that you have.

It is also when things start to get a little messy, because there are a lot of ways to quantize a model and, for the popular ones, we have many readily available checkpoints at our disposal.

With so many options, sometimes it is not very clear how much a given quantization affects the quality of the model. In some cases the model card includes information about top-1% accuracy; in others there are no clues about how much the quantization degraded the checkpoint. We have all seen cases where aggressive quantizations drop the quality of the model dramatically.

So the best way to know for sure is to run some benchmarks.

In this post I will show the results for **Qwen3.8-27B**, one of my favourite models right now at this size. I use it both for running inference and for creating fine-tuned versions.

As you will see, quantization does a pretty good job, but the model card alone is not enough to know which checkpoint is better.

If you want, you can also go straight to the [results](#results).

## Configuration

**Qwen3.8-27B** served with **vLLM** on an RTX PRO 6000 (the same options in all cases: greedy decoding and reasoning mode enabled).

These are the checkpoints that were evaluated:

| variant | checkpoint | weights | activations | weights on GPU |
|---|---|---|---|--:|
| BF16 | [Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) | 16-bit | 16-bit | 51.1 GiB |
| FP8 | [Qwen/Qwen3.8-27B-FP8](https://huggingface.co/Qwen/Qwen3.8-27B-FP8) | 8-bit float | 8-bit float | 28.5 |
| NVFP4 | [Inferact/Qwen3.8-27B-NVFP4](https://huggingface.co/Inferact/Qwen3.8-27B-NVFP4) | 4-bit float, gs 16 | 4-bit float | 24.2 |
| NVFP4 | [unsloth/Qwen3.8-27B-NVFP4](https://huggingface.co/unsloth/Qwen3.8-27B-NVFP4) | 4-bit on MLP, 8-bit elsewhere | 8-bit | 21.3 |
| NVFP4 | [RadixArk/Qwen3.8-27B-NVFP4](https://huggingface.co/RadixArk/Qwen3.8-27B-NVFP4) | mixed 4/8-bit | mixed | 20.0 |
| INT4 | [cyankiwi/Qwen3.8-27B-AWQ-INT4](https://huggingface.co/cyankiwi/Qwen3.8-27B-AWQ-INT4) | 4-bit int, gs 32 | 16-bit | 19.2 |
| INT4 | [RedHatAI/Qwen3.8-27B-INT4](https://huggingface.co/RedHatAI/Qwen3.8-27B-INT4) | 4-bit int, gs 128 | 16-bit | 17.7 |

And this is the vLLM command used to run all the checkpoints:

```bash
vllm serve "${MODEL}" \
    --served-model-name "${SERVED_NAME}" \
    --tensor-parallel-size 1 \
    --max-num-batched-tokens 8192 \
    --gpu-memory-utilization 0.95 \
    --max-num-seqs 256 \
    --max-model-len 262144 \
    --kv-cache-dtype fp8 \
    --no-enable-prefix-caching \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder \
    --reasoning-parser qwen3 \
    --mm-encoder-tp-mode data
```

For the one-shot benchmarks I use [lm-eval](https://github.com/EleutherAI/lm-evaluation-harness).

## Benchmarks

- **Agentic performance**:
  - **SWE-bench Verified**: the model drives a terminal to fix real GitHub
    issues in real repositories until the project's own tests pass.

- **General accuracy in one-shot tasks**:
  - **MMLU-Pro**: multiple-choice general knowledge and reasoning across 14
    academic and professional domains.
  - **BBH** (BIG-Bench Hard): mostly multi-step symbolic, logical and algorithmic reasoning.
  - **Minerva-MATH**: competition mathematics problems requiring worked
    derivations, graded on the final boxed answer.
  - **MBPP+**: short Python programming problems.
  - **IFEval**: instruction following under verifiable constraints ("answer in
    exactly three bullets", "no commas").
  - **GPQA Diamond**: 198 graduate-level biology, chemistry and physics
    questions written to be hard even for a non-expert with web access.

## Results

### Agentic performance

<figure class="qz">
  <div class="key">
    <span><i style="background:var(--ok)"></i>no measurable loss</span>
    <span><i style="background:var(--ref)"></i>BF16 reference</span>
    <span><i style="background:var(--bad)"></i>significant loss</span>
  </div>
  <svg viewBox="0 0 640 250" role="img" aria-label="SWE-bench Verified resolve rate. FP8 71.8 percent, BF16 71.6, INT4-RedHat 71.6, NVFP4-unsloth 71.4, AWQ-INT4 71.0, NVFP4-Inferact 63.6 percent.">
    <line x1="170" y1="14" x2="170" y2="196" stroke="var(--grid)"/>
    <line x1="288" y1="14" x2="288" y2="196" stroke="var(--grid)"/>
    <line x1="406" y1="14" x2="406" y2="196" stroke="var(--grid)"/>
    <line x1="524" y1="14" x2="524" y2="196" stroke="var(--grid)"/>
    <text class="ax" x="170" y="212" text-anchor="middle">60%</text>
    <text class="ax" x="288" y="212" text-anchor="middle">64%</text>
    <text class="ax" x="406" y="212" text-anchor="middle">68%</text>
    <text class="ax" x="524" y="212" text-anchor="middle">72%</text>
    <text class="ax" x="170" y="234" text-anchor="start">resolve rate — 500 SWE-bench Verified instances</text>
    <text class="lab" x="162" y="30" text-anchor="end">FP8</text>
    <rect x="170" y="19" width="349" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="527" y="31">71.8%</text>
    <text class="lab" x="162" y="59" text-anchor="end">BF16</text>
    <rect x="170" y="48" width="343" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="521" y="60">71.6%</text>
    <text class="lab" x="162" y="88" text-anchor="end">INT4-RedHat</text>
    <rect x="170" y="77" width="343" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="521" y="89">71.6%</text>
    <text class="lab" x="162" y="117" text-anchor="end">NVFP4-unsloth</text>
    <rect x="170" y="106" width="337" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="515" y="118">71.4%</text>
    <text class="lab" x="162" y="146" text-anchor="end">AWQ-INT4</text>
    <rect x="170" y="135" width="325" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="503" y="147">71.0%</text>
    <text class="lab" x="162" y="175" text-anchor="end">NVFP4-Inferact</text>
    <rect x="170" y="164" width="106" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="284" y="176" fill="var(--bad)">63.6%</text>
    <text class="sub" x="284" y="191" fill="var(--bad)">−8.0 points · p = 3.5×10⁻⁵</text>
  </svg>
  <figcaption>
    Paired McNemar test against BF16, counting only instances where both
    variants reached a verdict.
  </figcaption>
</figure>

> Four of the five quantizations are statistically indistinguishable from BF16 on the agentic coding benchmark; only **NVFP4-Inferact** degrades considerably.

Understanding why NVFP4-Inferact performs much worse than the other NVFP4 checkpoints took some research. The model cards do not make the difference clear, and the reason only shows up in the model repo: Inferact quantizes **activations alongside the weights**.

<figure class="qz">
  <svg viewBox="0 0 640 226" role="img" aria-label="Quantization scheme by checkpoint. FP8 uses 8-bit weights and activations. AWQ-INT4 and INT4-RedHat use 4-bit weights with 16-bit activations. NVFP4-unsloth uses 4-bit on MLP only with the rest at 8-bit. NVFP4-Inferact uses 4-bit weights and activations everywhere.">
    <text class="ax" x="288" y="14" text-anchor="middle">WEIGHTS</text>
    <text class="ax" x="452" y="14" text-anchor="middle">ACTIVATIONS</text>
    <text class="lab" x="212" y="42" text-anchor="end">FP8</text>
    <rect x="224" y="30" width="128" height="17" rx="3" fill="var(--ok)" opacity=".8"/>
    <text class="sub" x="288" y="43" text-anchor="middle" fill="#fff">8-bit</text>
    <rect x="388" y="30" width="128" height="17" rx="3" fill="var(--ok)" opacity=".8"/>
    <text class="sub" x="452" y="43" text-anchor="middle" fill="#fff">8-bit</text>
    <text class="lab" x="212" y="76" text-anchor="end">AWQ-INT4</text>
    <rect x="224" y="64" width="128" height="17" rx="3" fill="var(--ok)"/>
    <text class="sub" x="288" y="77" text-anchor="middle" fill="#fff">4-bit int</text>
    <rect x="388" y="64" width="128" height="17" rx="3" fill="none" stroke="var(--ref)" stroke-width="1.5" stroke-dasharray="3 3"/>
    <text class="sub" x="452" y="77" text-anchor="middle">16-bit</text>
    <text class="lab" x="212" y="110" text-anchor="end">INT4-RedHat</text>
    <rect x="224" y="98" width="128" height="17" rx="3" fill="var(--ok)"/>
    <text class="sub" x="288" y="111" text-anchor="middle" fill="#fff">4-bit int</text>
    <rect x="388" y="98" width="128" height="17" rx="3" fill="none" stroke="var(--ref)" stroke-width="1.5" stroke-dasharray="3 3"/>
    <text class="sub" x="452" y="111" text-anchor="middle">16-bit</text>
    <text class="lab" x="212" y="144" text-anchor="end">NVFP4-unsloth</text>
    <rect x="224" y="132" width="60" height="17" rx="3" fill="var(--ok)"/>
    <text class="sub" x="254" y="145" text-anchor="middle" fill="#fff">4-bit</text>
    <rect x="288" y="132" width="64" height="17" rx="3" fill="var(--ok)" opacity=".5"/>
    <text class="sub" x="320" y="145" text-anchor="middle" fill="#fff">8-bit</text>
    <rect x="388" y="132" width="128" height="17" rx="3" fill="var(--ok)" opacity=".5"/>
    <text class="sub" x="452" y="145" text-anchor="middle" fill="#fff">8-bit</text>
    <text class="lab" x="212" y="178" text-anchor="end">NVFP4-Inferact</text>
    <rect x="224" y="166" width="128" height="17" rx="3" fill="var(--bad)"/>
    <text class="sub" x="288" y="179" text-anchor="middle" fill="#fff">4-bit</text>
    <rect x="388" y="166" width="128" height="17" rx="3" fill="var(--bad)"/>
    <text class="sub" x="452" y="179" text-anchor="middle" fill="#fff">4-bit</text>
    <text class="sub" x="224" y="208">weights left block: MLP only; activations dashed: full precision</text>
  </svg>
  <figcaption>
    What each checkpoint actually quantizes, read from
    <code>quantization_config</code> in the model repo.
  </figcaption>
</figure>


> Quantization does a pretty good job.
> The only thing to avoid is quantizing the activations to 4 bits, because that degrades model performance in agentic coding tasks.
> Unfortunately, you have to manually open `config_groups` in the
> checkpoint config and read it to see which checkpoints do that.


### General accuracy in one-shot tasks

Two BF16 runs are plotted on every chart, so the difference between them represents the run-to-run noise in that benchmark.

Each bar in the figures is split into correct answers, incorrect answers and no-answer. I prefer to separate failed answers into wrong and no-answer, because answering wrongly is not the same as consuming all the output tokens before answering.

To have a better view of how the checkpoints compare, apart from the raw benchmark score, I also compare each one against the base BF16 variant using a paired McNemar test. The test ignores the cases where the two models agree and focuses on the disagreements, i.e. the questions that they answer differently.

A bar is marked in red when a paired McNemar test against BF16 returned
`p < 0.05` on that benchmark, which means that there is an important difference from the BF16 checkpoint; a ▲ marks the one case where a variant is
significantly *better*.

<figure class="qz">
  <svg viewBox="0 0 640 276" role="img" aria-label="MMLU-Pro: share of all items correct, answered but wrong, and not answered, for eight precision variants.">
    <defs><pattern id="pw" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(45)"><rect width="5" height="5" fill="var(--grid)"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.7" opacity=".55"/></pattern><pattern id="pn" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(135)"><rect width="5" height="5" fill="none"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.2" opacity=".38"/></pattern></defs>
    <line x1="150" y1="20" x2="150" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="150" y="250" text-anchor="middle">0%</text>
    <line x1="226" y1="20" x2="226" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="226" y="250" text-anchor="middle">25%</text>
    <line x1="301" y1="20" x2="301" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="301" y="250" text-anchor="middle">50%</text>
    <line x1="376" y1="20" x2="376" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="376" y="250" text-anchor="middle">75%</text>
    <line x1="452" y1="20" x2="452" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="452" y="250" text-anchor="middle">100%</text>
    <text class="ax" x="150" y="270" text-anchor="start">MMLU-Pro</text>
    <text class="lab" x="142" y="46" text-anchor="end">BF16 (a)</text>
    <rect x="150" y="34" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="34" width="290.5" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="34" width="251.1" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="46">83.1%</text>
    <text class="lab" x="142" y="72" text-anchor="end">BF16 (b)</text>
    <rect x="150" y="60" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="60" width="291.0" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="60" width="250.2" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="72">82.8%</text>
    <text class="lab" x="142" y="98" text-anchor="end">FP8</text>
    <rect x="150" y="86" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="86" width="289.4" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="86" width="250.1" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="98">82.8%</text>
    <text class="lab" x="142" y="124" text-anchor="end">NVFP4-unsloth</text>
    <rect x="150" y="112" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="112" width="291.8" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="112" width="250.4" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="124">82.9%</text>
    <text class="lab" x="142" y="150" text-anchor="end">NVFP4-RadixArk</text>
    <rect x="150" y="138" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="138" width="292.8" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="138" width="251.0" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="150">83.1%</text>
    <text class="lab" x="142" y="176" text-anchor="end">AWQ-INT4</text>
    <rect x="150" y="164" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="164" width="287.8" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="164" width="247.4" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="464" y="176" fill="var(--bad)">81.9%</text>
    <text class="sub" x="510" y="176" fill="var(--bad)">p = 0.0111</text>
    <text class="lab" x="142" y="202" text-anchor="end">INT4-RedHat</text>
    <rect x="150" y="190" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="190" width="276.1" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="190" width="238.1" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="202">78.8%</text>
    <text class="lab" x="142" y="228" text-anchor="end">NVFP4-Inferact</text>
    <rect x="150" y="216" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="216" width="282.6" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="216" width="244.3" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="464" y="228" fill="var(--bad)">80.9%</text>
    <text class="sub" x="510" y="228" fill="var(--bad)">p = 0.00349</text>
  </svg>
  <figcaption>
    INT4-RedHat has the lowest score, but this is mainly due to a large percentage of no-answer results. See below.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 276" role="img" aria-label="BBH: share of all items correct, answered but wrong, and not answered, for eight precision variants.">
    <defs><pattern id="pw" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(45)"><rect width="5" height="5" fill="var(--grid)"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.7" opacity=".55"/></pattern><pattern id="pn" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(135)"><rect width="5" height="5" fill="none"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.2" opacity=".38"/></pattern></defs>
    <line x1="150" y1="20" x2="150" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="150" y="250" text-anchor="middle">0%</text>
    <line x1="226" y1="20" x2="226" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="226" y="250" text-anchor="middle">25%</text>
    <line x1="301" y1="20" x2="301" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="301" y="250" text-anchor="middle">50%</text>
    <line x1="376" y1="20" x2="376" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="376" y="250" text-anchor="middle">75%</text>
    <line x1="452" y1="20" x2="452" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="452" y="250" text-anchor="middle">100%</text>
    <text class="ax" x="150" y="270" text-anchor="start">BBH</text>
    <text class="lab" x="142" y="46" text-anchor="end">BF16 (a)</text>
    <rect x="150" y="34" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="34" width="299.4" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="34" width="274.4" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="46">90.8%</text>
    <text class="lab" x="142" y="72" text-anchor="end">BF16 (b)</text>
    <rect x="150" y="60" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="60" width="299.1" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="60" width="274.0" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="72">90.7%</text>
    <text class="lab" x="142" y="98" text-anchor="end">FP8</text>
    <rect x="150" y="86" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="86" width="299.8" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="86" width="275.4" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="98">91.2%</text>
    <text class="lab" x="142" y="124" text-anchor="end">NVFP4-unsloth</text>
    <rect x="150" y="112" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="112" width="299.9" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="112" width="269.8" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="464" y="124" fill="var(--bad)">89.3%</text>
    <text class="sub" x="510" y="124" fill="var(--bad)">p = 3.0e-07</text>
    <text class="lab" x="142" y="150" text-anchor="end">NVFP4-RadixArk</text>
    <rect x="150" y="138" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="138" width="300.0" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="138" width="268.2" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="464" y="150" fill="var(--bad)">88.8%</text>
    <text class="sub" x="510" y="150" fill="var(--bad)">p = 8.4e-12</text>
    <text class="lab" x="142" y="176" text-anchor="end">AWQ-INT4</text>
    <rect x="150" y="164" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="164" width="299.1" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="164" width="270.9" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="464" y="176" fill="var(--bad)">89.7%</text>
    <text class="sub" x="510" y="176" fill="var(--bad)">p = 7.5e-05</text>
    <text class="lab" x="142" y="202" text-anchor="end">INT4-RedHat</text>
    <rect x="150" y="190" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="190" width="298.6" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="190" width="273.6" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="202">90.6%</text>
    <text class="lab" x="142" y="228" text-anchor="end">NVFP4-Inferact</text>
    <rect x="150" y="216" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="216" width="297.3" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="216" width="256.1" height="15" rx="3" fill="var(--bad)"/>
    <text class="val" x="464" y="228" fill="var(--bad)">84.8%</text>
    <text class="sub" x="510" y="228" fill="var(--bad)">p < 1e-50</text>
  </svg>
  <figcaption>
    Four variants are highlighted because they separate from BF16 in the paired McNemar test. The NVFP4-Inferact checkpoint is the most affected, so this benchmark also seems to be highly impacted by 4-bit activation quantization, as was the case in SWE-bench Verified.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 276" role="img" aria-label="Minerva-MATH: share of all items correct, answered but wrong, and not answered, for eight precision variants.">
    <defs><pattern id="pw" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(45)"><rect width="5" height="5" fill="var(--grid)"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.7" opacity=".55"/></pattern><pattern id="pn" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(135)"><rect width="5" height="5" fill="none"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.2" opacity=".38"/></pattern></defs>
    <line x1="150" y1="20" x2="150" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="150" y="250" text-anchor="middle">0%</text>
    <line x1="226" y1="20" x2="226" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="226" y="250" text-anchor="middle">25%</text>
    <line x1="301" y1="20" x2="301" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="301" y="250" text-anchor="middle">50%</text>
    <line x1="376" y1="20" x2="376" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="376" y="250" text-anchor="middle">75%</text>
    <line x1="452" y1="20" x2="452" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="452" y="250" text-anchor="middle">100%</text>
    <text class="ax" x="150" y="270" text-anchor="start">Minerva-MATH</text>
    <text class="lab" x="142" y="46" text-anchor="end">BF16 (a)</text>
    <rect x="150" y="34" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="34" width="298.7" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="34" width="293.3" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="46">97.1%</text>
    <text class="lab" x="142" y="72" text-anchor="end">BF16 (b)</text>
    <rect x="150" y="60" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="60" width="298.4" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="60" width="292.8" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="72">97.0%</text>
    <text class="lab" x="142" y="98" text-anchor="end">FP8</text>
    <rect x="150" y="86" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="86" width="298.3" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="86" width="293.8" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="98">97.3%</text>
    <text class="lab" x="142" y="124" text-anchor="end">NVFP4-unsloth</text>
    <rect x="150" y="112" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="112" width="298.4" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="112" width="292.3" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="124">96.8%</text>
    <text class="lab" x="142" y="150" text-anchor="end">NVFP4-RadixArk</text>
    <rect x="150" y="138" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="138" width="298.4" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="138" width="292.5" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="150">96.9%</text>
    <text class="lab" x="142" y="176" text-anchor="end">AWQ-INT4</text>
    <rect x="150" y="164" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="164" width="298.5" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="164" width="292.8" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="176">96.9%</text>
    <text class="lab" x="142" y="202" text-anchor="end">INT4-RedHat</text>
    <rect x="150" y="190" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="190" width="297.6" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="190" width="292.3" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="202">96.8%</text>
    <text class="lab" x="142" y="228" text-anchor="end">NVFP4-Inferact</text>
    <rect x="150" y="216" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="216" width="296.3" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="216" width="290.9" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="228">96.3%</text>
  </svg>
  <figcaption>
    All perform well in maths.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 276" role="img" aria-label="MBPP+: share of all items correct, answered but wrong, and not answered, for eight precision variants.">
    <defs><pattern id="pw" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(45)"><rect width="5" height="5" fill="var(--grid)"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.7" opacity=".55"/></pattern><pattern id="pn" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(135)"><rect width="5" height="5" fill="none"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.2" opacity=".38"/></pattern></defs>
    <line x1="150" y1="20" x2="150" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="150" y="250" text-anchor="middle">0%</text>
    <line x1="226" y1="20" x2="226" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="226" y="250" text-anchor="middle">25%</text>
    <line x1="301" y1="20" x2="301" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="301" y="250" text-anchor="middle">50%</text>
    <line x1="376" y1="20" x2="376" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="376" y="250" text-anchor="middle">75%</text>
    <line x1="452" y1="20" x2="452" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="452" y="250" text-anchor="middle">100%</text>
    <text class="ax" x="150" y="270" text-anchor="start">MBPP+</text>
    <text class="lab" x="142" y="46" text-anchor="end">BF16 (a)</text>
    <rect x="150" y="34" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="34" width="293.2" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="34" width="290.8" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="46">96.3%</text>
    <text class="lab" x="142" y="72" text-anchor="end">BF16 (b)</text>
    <rect x="150" y="60" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="60" width="294.0" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="60" width="291.6" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="72">96.6%</text>
    <text class="lab" x="142" y="98" text-anchor="end">FP8</text>
    <rect x="150" y="86" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="86" width="291.6" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="86" width="289.2" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="98">95.8%</text>
    <text class="lab" x="142" y="124" text-anchor="end">NVFP4-unsloth</text>
    <rect x="150" y="112" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="112" width="292.4" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="112" width="287.6" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="124">95.2%</text>
    <text class="lab" x="142" y="150" text-anchor="end">NVFP4-RadixArk</text>
    <rect x="150" y="138" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="138" width="296.4" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="138" width="291.6" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="150">96.6%</text>
    <text class="lab" x="142" y="176" text-anchor="end">AWQ-INT4</text>
    <rect x="150" y="164" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="164" width="289.2" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="164" width="285.2" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="176">94.4%</text>
    <text class="lab" x="142" y="202" text-anchor="end">INT4-RedHat</text>
    <rect x="150" y="190" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="190" width="290.0" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="190" width="287.6" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="202">95.2%</text>
    <text class="lab" x="142" y="228" text-anchor="end">NVFP4-Inferact</text>
    <rect x="150" y="216" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="216" width="285.2" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="216" width="281.2" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="228">93.1%</text>
  </svg>
  <figcaption>
    All perform well at creating short Python functions.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 276" role="img" aria-label="IFEval: share of all items correct, answered but wrong, and not answered, for eight precision variants.">
    <defs><pattern id="pw" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(45)"><rect width="5" height="5" fill="var(--grid)"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.7" opacity=".55"/></pattern><pattern id="pn" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(135)"><rect width="5" height="5" fill="none"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.2" opacity=".38"/></pattern></defs>
    <line x1="150" y1="20" x2="150" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="150" y="250" text-anchor="middle">0%</text>
    <line x1="226" y1="20" x2="226" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="226" y="250" text-anchor="middle">25%</text>
    <line x1="301" y1="20" x2="301" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="301" y="250" text-anchor="middle">50%</text>
    <line x1="376" y1="20" x2="376" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="376" y="250" text-anchor="middle">75%</text>
    <line x1="452" y1="20" x2="452" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="452" y="250" text-anchor="middle">100%</text>
    <text class="ax" x="150" y="270" text-anchor="start">IFEval</text>
    <text class="lab" x="142" y="46" text-anchor="end">BF16 (a)</text>
    <rect x="150" y="34" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="34" width="288.6" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="34" width="263.5" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="46">87.2%</text>
    <text class="lab" x="142" y="72" text-anchor="end">BF16 (b)</text>
    <rect x="150" y="60" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="60" width="288.6" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="60" width="264.0" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="72">87.4%</text>
    <text class="lab" x="142" y="98" text-anchor="end">FP8</text>
    <rect x="150" y="86" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="86" width="290.3" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="86" width="268.5" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="98">88.9%</text>
    <text class="lab" x="142" y="124" text-anchor="end">NVFP4-unsloth</text>
    <rect x="150" y="112" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="112" width="290.8" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="112" width="269.6" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="124">89.3%</text>
    <text class="lab" x="142" y="150" text-anchor="end">NVFP4-RadixArk</text>
    <rect x="150" y="138" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="138" width="291.4" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="138" width="271.9" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="150">90.0%</text>
    <text class="sub" x="510" y="150" fill="var(--ok)">▲ p = 0.0127</text>
    <text class="lab" x="142" y="176" text-anchor="end">AWQ-INT4</text>
    <rect x="150" y="164" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="164" width="286.9" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="164" width="265.2" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="176">87.8%</text>
    <text class="lab" x="142" y="202" text-anchor="end">INT4-RedHat</text>
    <rect x="150" y="190" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="190" width="283.0" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="190" width="262.4" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="202">86.9%</text>
    <text class="lab" x="142" y="228" text-anchor="end">NVFP4-Inferact</text>
    <rect x="150" y="216" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="216" width="279.1" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="216" width="256.8" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="228">85.0%</text>
  </svg>
  <figcaption>
    NVFP4-RadixArk at p = 0.0127 (▲) shows better performance than BF16 in the McNemar test, but this seems to be just due to chance across 42 comparisons.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 276" role="img" aria-label="GPQA Diamond: share of all items correct, answered but wrong, and not answered, for eight precision variants.">
    <defs><pattern id="pw" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(45)"><rect width="5" height="5" fill="var(--grid)"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.7" opacity=".55"/></pattern><pattern id="pn" width="5" height="5" patternUnits="userSpaceOnUse" patternTransform="rotate(135)"><rect width="5" height="5" fill="none"/><line x1="0" y1="0" x2="0" y2="5" stroke="var(--dim)" stroke-width="1.2" opacity=".38"/></pattern></defs>
    <line x1="150" y1="20" x2="150" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="150" y="250" text-anchor="middle">0%</text>
    <line x1="226" y1="20" x2="226" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="226" y="250" text-anchor="middle">25%</text>
    <line x1="301" y1="20" x2="301" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="301" y="250" text-anchor="middle">50%</text>
    <line x1="376" y1="20" x2="376" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="376" y="250" text-anchor="middle">75%</text>
    <line x1="452" y1="20" x2="452" y2="234" stroke="var(--grid)"/>
    <text class="ax" x="452" y="250" text-anchor="middle">100%</text>
    <text class="ax" x="150" y="270" text-anchor="start">GPQA Diamond</text>
    <text class="lab" x="142" y="46" text-anchor="end">BF16 (a)</text>
    <rect x="150" y="34" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="34" width="207.4" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="34" width="190.7" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="46">63.1%</text>
    <text class="lab" x="142" y="72" text-anchor="end">BF16 (b)</text>
    <rect x="150" y="60" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="60" width="210.5" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="60" width="199.8" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="464" y="72">66.2%</text>
    <text class="lab" x="142" y="98" text-anchor="end">FP8</text>
    <rect x="150" y="86" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="86" width="213.5" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="86" width="195.2" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="98">64.6%</text>
    <text class="lab" x="142" y="124" text-anchor="end">NVFP4-unsloth</text>
    <rect x="150" y="112" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="112" width="210.5" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="112" width="193.7" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="124">64.1%</text>
    <text class="lab" x="142" y="150" text-anchor="end">NVFP4-RadixArk</text>
    <rect x="150" y="138" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="138" width="216.6" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="138" width="205.9" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="150">68.2%</text>
    <text class="lab" x="142" y="176" text-anchor="end">AWQ-INT4</text>
    <rect x="150" y="164" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="164" width="210.5" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="164" width="192.2" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="176">63.6%</text>
    <text class="lab" x="142" y="202" text-anchor="end">INT4-RedHat</text>
    <rect x="150" y="190" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="190" width="190.7" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="190" width="169.3" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="202">56.1%</text>
    <text class="lab" x="142" y="228" text-anchor="end">NVFP4-Inferact</text>
    <rect x="150" y="216" width="302" height="15" rx="3" fill="url(#pn)" stroke="var(--grid)" stroke-width="1"/>
    <rect x="150" y="216" width="186.1" height="15" rx="3" fill="url(#pw)"/>
    <rect x="150" y="216" width="170.8" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="464" y="228">56.6%</text>
  </svg>
  <figcaption>
    A large percentage of no-answers in all variants, roughly 30%, and just a small set of 198 questions, so the final reported numbers are not statistically significant.
  </figcaption>
</figure>


**INT4-RedHat has the lowest raw score** on this family of benchmarks, last on MMLU-Pro and last on GPQA, but on the paired test it is the *only* 4-bit
checkpoint not significantly worse than BF16. This is why having both the raw scores and the McNemar test results is useful.

<figure class="qz">
  <svg viewBox="0 0 640 296" role="img" aria-label="Two panels. Top: MMLU-Pro raw accuracy, INT4-RedHat lowest at 78.8 percent. Bottom: non-termination rate, INT4-RedHat highest at 8.6 percent.">
    <text class="ax" x="0" y="12">MMLU-PRO RAW ACCURACY</text>
    <line x1="130" y1="22" x2="130" y2="132" stroke="var(--grid)"/>
    <text class="lab" x="122" y="38" text-anchor="end">BF16</text>
    <rect x="130" y="27" width="332" height="14" rx="3" fill="var(--ref)"/>
    <text class="val" x="470" y="39">83.1%</text>
    <text class="lab" x="122" y="64" text-anchor="end">AWQ-INT4</text>
    <rect x="130" y="53" width="308" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="446" y="65">81.9%</text>
    <text class="lab" x="122" y="90" text-anchor="end">NVFP4-Inferact</text>
    <rect x="130" y="79" width="288" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="426" y="91">80.9%</text>
    <text class="lab" x="122" y="116" text-anchor="end">INT4-RedHat</text>
    <rect x="130" y="105" width="246" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="384" y="117">78.8%</text>
    <text class="sub" x="440" y="117">← lowest of any variant</text>
    <line x1="0" y1="160" x2="640" y2="160" stroke="var(--grid)"/>
    <text class="ax" x="0" y="186">NON-TERMINATION — MODEL PRODUCED NO ANSWER AT ALL</text>
    <line x1="130" y1="196" x2="130" y2="282" stroke="var(--grid)"/>
    <text class="lab" x="122" y="212" text-anchor="end">BF16</text>
    <rect x="130" y="201" width="106" height="14" rx="3" fill="var(--ref)"/>
    <text class="val" x="244" y="213">3.8%</text>
    <text class="lab" x="122" y="238" text-anchor="end">AWQ-INT4</text>
    <rect x="130" y="227" width="131" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="269" y="239">4.7%</text>
    <text class="lab" x="122" y="264" text-anchor="end">INT4-RedHat</text>
    <rect x="130" y="253" width="240" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="378" y="265">8.6%</text>
    <text class="sub" x="434" y="265">← highest of any variant</text>
  </svg>
  <figcaption>
    Benchmark evaluations score an empty response identically
    to a wrong one, so a model that fails to <em>finish</em> looks like a model
    that answers <em>badly</em>. INT4-RedHat is penalized by this because it has the highest no-answer rate.
  </figcaption>
</figure>

Looking at the details, **INT4-RedHat's low score comes from its high percentage of no-answer results**.
This happens when the model reasons past its token budget and so returns no answer at all.
**On the questions it does answer, it is the least degraded 4-bit checkpoint.**

Looking at some of the specific questions that it does not answer, the model is not looping.
The traces show reasoning that is making progress, cut off mid-stream by the budget. Increasing the budget improves the results.
For some reason this checkpoint needs more reasoning than the others.

So when ranking on the raw benchmark numbers INT4-RedHat is the worst checkpoint, but when ranking on the paired
test it is the best 4-bit checkpoint in the set.

Surprisingly, on SWE-bench the same checkpoint had **zero** non-termination and performed on par with BF16.

## Memory usage

This is the memory usage for each checkpoint, as reported by vLLM on one GPU:

<figure class="qz">
  <svg viewBox="0 0 640 240" role="img" aria-label="Weight footprint by variant. BF16 51.1 gibibytes, FP8 28.51 gibibytes, NVFP4-Inferact 24.18 gibibytes, NVFP4-unsloth 21.34 gibibytes, NVFP4-RadixArk 19.95 gibibytes, AWQ-INT4 19.24 gibibytes, INT4-RedHat 17.71 gibibytes.">
    <line x1="221" y1="14" x2="221" y2="200" stroke="var(--grid)"/>
    <text class="ax" x="221" y="214" text-anchor="middle">10</text>
    <line x1="292" y1="14" x2="292" y2="200" stroke="var(--grid)"/>
    <text class="ax" x="292" y="214" text-anchor="middle">20</text>
    <line x1="363" y1="14" x2="363" y2="200" stroke="var(--grid)"/>
    <text class="ax" x="363" y="214" text-anchor="middle">30</text>
    <line x1="434" y1="14" x2="434" y2="200" stroke="var(--grid)"/>
    <text class="ax" x="434" y="214" text-anchor="middle">40</text>
    <line x1="505" y1="14" x2="505" y2="200" stroke="var(--grid)"/>
    <text class="ax" x="505" y="214" text-anchor="middle">50</text>
    <text class="ax" x="150" y="234" text-anchor="start">GiB of weights on one GPU</text>
    <text class="lab" x="142" y="38" text-anchor="end">BF16</text>
    <rect x="150" y="26" width="362" height="15" rx="3" fill="var(--ref)"/>
    <text class="val" x="520" y="38">51.1 GiB</text>
    <text class="lab" x="142" y="64" text-anchor="end">FP8</text>
    <rect x="150" y="52" width="202" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="360" y="64">28.5 GiB</text>
    <text class="sub" x="418" y="64">−44%</text>
    <text class="lab" x="142" y="90" text-anchor="end">NVFP4-Inferact</text>
    <rect x="150" y="78" width="171" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="329" y="90">24.2 GiB</text>
    <text class="sub" x="387" y="90">−53%</text>
    <text class="lab" x="142" y="116" text-anchor="end">NVFP4-unsloth</text>
    <rect x="150" y="104" width="151" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="309" y="116">21.3 GiB</text>
    <text class="sub" x="367" y="116">−58%</text>
    <text class="lab" x="142" y="142" text-anchor="end">NVFP4-RadixArk</text>
    <rect x="150" y="130" width="141" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="299" y="142">19.9 GiB</text>
    <text class="sub" x="357" y="142">−61%</text>
    <text class="lab" x="142" y="168" text-anchor="end">AWQ-INT4</text>
    <rect x="150" y="156" width="136" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="294" y="168">19.2 GiB</text>
    <text class="sub" x="352" y="168">−62%</text>
    <text class="lab" x="142" y="194" text-anchor="end">INT4-RedHat</text>
    <rect x="150" y="182" width="126" height="15" rx="3" fill="var(--ok)"/>
    <text class="val" x="284" y="194">17.7 GiB</text>
    <text class="sub" x="342" y="194">−65%</text>
  </svg>
  <figcaption>
    Measured at load, TP=1 on one RTX PRO 6000. Percentages are against BF16.
  </figcaption>
</figure>

The GPU memory budget is fixed, and I like to use a higher utilization setting (`--gpu-memory-utilization 0.95`), so whatever the weights do not use is assigned to the KV cache:

| variant | on disk | weights<br/>(+overhead) | peak<br/>activations | CUDA<br/>graphs | KV cache | KV tokens | vs BF16 |
|---|--:|--:|--:|--:|--:|--:|--:|
| BF16 | 51.8 | 51.9 | 3.27 | 1.82 | 35.0 | 1,120,627 | 1.00× |
| FP8 | 28.8 | 29.4 | 3.34 | 1.86 | 57.5 | 1,842,673 | 1.64× |
| NVFP4-Inferact | 24.6 | 25.1 | 3.27 | 1.87 | 61.8 | 1,979,110 | 1.77× |
| NVFP4-unsloth | 21.8 | 22.3 | 3.27 | 1.86 | 64.6 | 2,069,557 | 1.85× |
| NVFP4-RadixArk | — | 21.1 | 3.34 | 1.86 | 65.8 | 2,107,883 | 1.88× |
| AWQ-INT4 | 19.6 | 20.1 | 3.34 | 1.86 | 66.8 | 2,138,543 | 1.91× |
| **INT4-RedHat** | **18.1** | **18.3** | 3.34 | 1.86 | **68.6** | **2,196,797** | **1.96×** |


## Conclusions

Quantization performs really well in most cases, close enough to the original model
that you cannot tell them apart, but you have to be careful with the checkpoint that you choose.

In the case of **Qwen3.8-27B**:

- FP8 is indistinguishable from BF16.
- 4-bit variants are also good, but you should choose carefully based on the use case.
- NVFP4-Inferact is the one that performs worst, due to 4-bit activations.
- INT4-RedHat performs very well being the smallest in size, but it has a very high no-answer rate in one-shot benchmarks.

This is the final summary:

| variant | weights | saved | measured cost |
|---|--:|--:|---|
| BF16 | 51.1 GiB | — | reference |
| FP8 | 28.5 GiB | 44% | none |
| NVFP4-Inferact | 24.2 GiB | 53% | **−8.0 pts SWE-bench**, significant on 2 of 6 benchmarks |
| NVFP4-unsloth | 21.3 GiB | 58% | significant on BBH only |
| NVFP4-RadixArk | 20.0 GiB | 61% | significant on BBH only |
| AWQ-INT4 | 19.2 GiB | 62% | significant on BBH and MMLU-Pro |
| INT4-RedHat | 17.7 GiB | 65% | small, but highest no-answer rate |
