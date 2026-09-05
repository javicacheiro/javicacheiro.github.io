---
title: 'How quantization affects model quality'
description: 'Unfortunately quality is not always preserved'
publishDate: 2026-09-03
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

It is always nice to have the option to run a quantized version of a model that can fit in memory, but I wanted to measure how much quantization affected model quality.

So I have taken eight variants of the latest **Qwen3.8-27B** model and run six common benchmarks to compare them objectively.

The conclusion is that quantization in general does a pretty good job, but sometimes just looking at the model card is not enought to understand which version is better.

For example, two checkpoints of the same model, both labelled NVFP4, both nominally 4-bit,
differ by eight points on real software-engineering tasks.

## The setup

**Qwen3.8-27B** in eight precision variants, served with
**vLLM** with greedy decoding and reasoning mode pinned. Everything except
the checkpoint held constant.

Two independent benchmark families:

- **One-shot answer quality**:
  - **MMLU-Pro** — multiple-choice general knowledge and reasoning across 14
    academic and professional domains; the harder, ten-option successor to MMLU.
  - **BBH** (BIG-Bench Hard) — the 23 BIG-Bench tasks that language models used
    to fail, mostly multi-step symbolic, logical and algorithmic reasoning.
  - **Minerva-MATH** — competition mathematics problems requiring worked
    derivations, graded on the final boxed answer.
  - **MBPP+** — short Python programming problems checked by running an expanded
    set of unit tests, so a plausible-looking function that doesn't work fails.
  - **IFEval** — instruction following under verifiable constraints ("answer in
    exactly three bullets", "no commas"), scored programmatically rather than by
    content.
  - **GPQA Diamond** — 198 graduate-level biology, chemistry and physics
    questions written to be hard even for a non-expert with web access.
- **Agentic task benchmarks**:
  - **SWE-bench Verified**: the model driving a
  terminal to fix real GitHub issues in real repositories until the project's
  own tests pass. 3,000 instance-runs across six variants.

The first family measures one-shot answer quality; the second measures whether
the model can hold a multi-turn task together. They turned out to disagree.

I run **bf16 twice** because vLLM is not bit-deterministic at
temperature 0 — kernel selection varies with batch composition — so without
measuring the same-checkpoint spread there is no way to know whether a
two-point difference means anything. That control came back at `p = 0.377`.
It is what makes every other number below interpretable.

## Results

<figure class="qz">
  <div class="key">
    <span><i style="background:var(--ok)"></i>no measurable loss</span>
    <span><i style="background:var(--ref)"></i>bf16 reference</span>
    <span><i style="background:var(--bad)"></i>significant loss</span>
  </div>
  <svg viewBox="0 0 640 250" role="img" aria-label="SWE-bench Verified resolve rate. FP8 71.8 percent, bf16 71.6, INT4-RedHat 71.6, NVFP4-unsloth 71.4, AWQ-INT4 71.0, NVFP4-Inferact 63.6 percent.">
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
    <text class="lab" x="162" y="59" text-anchor="end">bf16</text>
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
    Paired McNemar test against bf16, counting only instances where both
    variants reached a verdict. Five variants sit within four instances of the
    reference out of 500. The sixth does not.
  </figcaption>
</figure>

Five of six quantizations are statistically indistinguishable from full precision on a 500-instance agentic coding benchmark. The sixth loses eight points.

## Why the outlier is an outlier

The obvious hypotheses are wrong. It isn't bit-width: two of the free variants
are also 4-bit. It isn't integer-versus-float: the free set contains both.

It's **whether activations are quantized alongside the weights.**

<figure class="qz">
  <svg viewBox="0 0 640 226" role="img" aria-label="Quantization scheme by checkpoint. FP8 uses 8-bit weights and activations. AWQ-INT4 and INT4-RedHat use 4-bit weights with 16-bit activations. NVFP4-unsloth uses 4-bit on MLP only with the rest at 8-bit. NVFP4-Inferact uses 4-bit weights and activations everywhere.">
    <text class="ax" x="288" y="14" text-anchor="middle">WEIGHTS</text>
    <text class="ax" x="452" y="14" text-anchor="middle">ACTIVATIONS</text>
    <text class="ax" x="580" y="14" text-anchor="middle">COST</text>
    <text class="lab" x="212" y="42" text-anchor="end">FP8</text>
    <rect x="224" y="30" width="128" height="17" rx="3" fill="var(--ok)" opacity=".8"/>
    <text class="sub" x="288" y="43" text-anchor="middle" fill="#fff">8-bit</text>
    <rect x="388" y="30" width="128" height="17" rx="3" fill="var(--ok)" opacity=".8"/>
    <text class="sub" x="452" y="43" text-anchor="middle" fill="#fff">8-bit</text>
    <text class="val" x="580" y="43" text-anchor="middle" fill="var(--ok)">none</text>
    <text class="lab" x="212" y="76" text-anchor="end">AWQ-INT4</text>
    <rect x="224" y="64" width="128" height="17" rx="3" fill="var(--ok)"/>
    <text class="sub" x="288" y="77" text-anchor="middle" fill="#fff">4-bit int</text>
    <rect x="388" y="64" width="128" height="17" rx="3" fill="none" stroke="var(--ref)" stroke-width="1.5" stroke-dasharray="3 3"/>
    <text class="sub" x="452" y="77" text-anchor="middle">16-bit</text>
    <text class="val" x="580" y="77" text-anchor="middle" fill="var(--ok)">none</text>
    <text class="lab" x="212" y="110" text-anchor="end">INT4-RedHat</text>
    <rect x="224" y="98" width="128" height="17" rx="3" fill="var(--ok)"/>
    <text class="sub" x="288" y="111" text-anchor="middle" fill="#fff">4-bit int</text>
    <rect x="388" y="98" width="128" height="17" rx="3" fill="none" stroke="var(--ref)" stroke-width="1.5" stroke-dasharray="3 3"/>
    <text class="sub" x="452" y="111" text-anchor="middle">16-bit</text>
    <text class="val" x="580" y="111" text-anchor="middle" fill="var(--ok)">none</text>
    <text class="lab" x="212" y="144" text-anchor="end">NVFP4-unsloth</text>
    <rect x="224" y="132" width="60" height="17" rx="3" fill="var(--ok)"/>
    <text class="sub" x="254" y="145" text-anchor="middle" fill="#fff">4-bit</text>
    <rect x="288" y="132" width="64" height="17" rx="3" fill="var(--ok)" opacity=".5"/>
    <text class="sub" x="320" y="145" text-anchor="middle" fill="#fff">8-bit</text>
    <rect x="388" y="132" width="128" height="17" rx="3" fill="var(--ok)" opacity=".5"/>
    <text class="sub" x="452" y="145" text-anchor="middle" fill="#fff">8-bit</text>
    <text class="val" x="580" y="145" text-anchor="middle" fill="var(--ok)">none</text>
    <text class="lab" x="212" y="178" text-anchor="end">NVFP4-Inferact</text>
    <rect x="224" y="166" width="128" height="17" rx="3" fill="var(--bad)"/>
    <text class="sub" x="288" y="179" text-anchor="middle" fill="#fff">4-bit</text>
    <rect x="388" y="166" width="128" height="17" rx="3" fill="var(--bad)"/>
    <text class="sub" x="452" y="179" text-anchor="middle" fill="#fff">4-bit</text>
    <text class="val" x="580" y="179" text-anchor="middle" fill="var(--bad)">−8.0 pts</text>
    <text class="sub" x="224" y="208">left block = MLP only · dashed = left at full precision</text>
  </svg>
  <figcaption>
    What each checkpoint actually quantizes, read from
    <code>quantization_config</code> in the model repo. The three middle rows
    are all described as "4-bit". Only the bottom row takes activations down
    with the weights.
  </figcaption>
</figure>

W4A16 — four-bit weights, sixteen-bit activations — is free. So is quantizing
only the MLP projections and leaving attention at eight bits. Uniform W4A4
across every `Linear` layer is not.

> **The practical rule:** you can quantize this model to four bits essentially
> for free, provided you don't take activations down with the weights. And the
> model card will not tell you which you're getting — you have to open
> `config_groups` in the checkpoint config and read it.

I only found this because a *third* NVFP4 checkpoint failed to serve, and
diagnosing that sent me to compare configs. Three checkpoints, three genuinely
different schemes, one shared label.

## The number that lies

Here is the result that most changed how I read benchmarks.

On the accuracy suite, **INT4-RedHat has the lowest raw score of any variant** —
last on MMLU-Pro, last on GPQA. On the paired test it is the *only* 4-bit
checkpoint not significantly worse than bf16. Both are true, because they
measure different things.

<figure class="qz">
  <svg viewBox="0 0 640 296" role="img" aria-label="Two panels. Top: MMLU-Pro raw accuracy, INT4-RedHat lowest at 78.8 percent. Bottom: non-termination rate, INT4-RedHat highest at 8.6 percent.">
    <text class="ax" x="0" y="12">MMLU-PRO RAW ACCURACY</text>
    <line x1="130" y1="22" x2="130" y2="132" stroke="var(--grid)"/>
    <text class="lab" x="122" y="38" text-anchor="end">bf16</text>
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
    <text class="lab" x="122" y="212" text-anchor="end">bf16</text>
    <rect x="130" y="201" width="106" height="14" rx="3" fill="var(--ref)"/>
    <text class="val" x="244" y="213">3.8%</text>
    <text class="lab" x="122" y="238" text-anchor="end">AWQ-INT4</text>
    <rect x="130" y="227" width="131" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="269" y="239">4.7%</text>
    <text class="lab" x="122" y="264" text-anchor="end">INT4-RedHat</text>
    <rect x="130" y="253" width="240" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="378" y="265">8.6%</text>
    <text class="sub" x="434" y="265">← highest, and the whole explanation</text>
  </svg>
  <figcaption>
    Same variants, two measures. lm-eval scores an empty response identically
    to a wrong one, so a model that fails to <em>finish</em> looks like a model
    that answers <em>badly</em>. Both panels are percentages but they are
    separate charts, not two axes on one plot.
  </figcaption>
</figure>

INT4-RedHat's low score is not wrong answers. It is **answers that never
arrive** — the model reasoning past its token budget and returning nothing,
which the harness scores as incorrect. On the questions it actually answers, it
is the least degraded 4-bit checkpoint I tested.

Two distinct failure modes, *answers wrongly* and *fails to answer*, merged into
one number. Ranking on the raw figure would have put this checkpoint last. It is
arguably first.

The postscript sharpens it: on SWE-bench the same checkpoint had **zero**
non-termination and landed exactly on bf16. That behaviour belonged to
long-form generation against a token ceiling — not to the checkpoint in agentic
use, which is the mode anyone would actually deploy it in.

## Agentic benchmarks see things accuracy suites don't

The two families disagreed about magnitude, and the agentic one was far more
sensitive:

| | MMLU-Pro | SWE-bench |
|---|--:|--:|
| NVFP4-Inferact vs bf16 | −2.24 pts | **−8.0 pts** |

Agentic tasks chain many model calls, so a small per-step quality loss compounds
across turns. Eight points of resolved issues is a much clearer signal than two
points on multiple choice.

There's a related problem worth naming. Of six benchmarks, **Minerva-MATH
(0.97), MBPP+ (0.96) and IFEval (0.87) are at ceiling** for a 2026 reasoning
model and null in every pairing, and GPQA Diamond can't resolve anything at 198
items with ~30% non-termination. A six-benchmark suite produced two benchmarks'
worth of evidence. I included Minerva expecting maths to be the *most*
quantization-sensitive task; it turned out to be saturated.

## An 8-bit aside

I planned W8A8 INT8 as the integer comparator to FP8. It cannot run on Blackwell
at all — both SM100 (B200) and SM120 (RTX PRO 6000) refuse it:

```
Int8 not supported on SM100. Use FP8 quantization instead,
or run on older arch (SM < 100).
```

The generation dropped INT8 tensor cores. That reframes the FP8 result: it isn't
FP8 winning a comparison at eight bits, it's **the only eight-bit option that
exists** on this hardware.

## The part I'd want someone else to read

Before any of the above was true, I published a version of it that was wrong.

The first run reported three benchmarks at exactly `0.0000` for every variant.
A flat zero across all variants is not a model result, it's a harness result —
and investigating found that **five of six benchmarks were not measuring what
their names claimed**:

- **BBH** used stop strings written for a raw completion model (`"\n\n"`, `"Q"`),
  which terminate a chat response at position zero. Repaired: `0.0000 → 0.9094`.
- **MMLU-Pro** stopped on `"Question:"` — which a reasoning model emits while
  restating the problem to itself. `0.2857 → 0.8095`.
- **Minerva-MATH** had the same stop-string bug *plus* an `exact_match` with no
  extraction filter, comparing the whole response against the target. It scored
  `\boxed{2}` against target `2` as wrong. Reported 0.1926; true value 0.9712.
- **GPQA** and **MBPP+** each had answer extraction assuming output formats a
  reasoning chat model never produces.

Three of those were the same root cause wearing different clothes:
few-shot-transcript stop strings are wrong for a chat endpoint, in every task
that ships them.

The consequence wasn't just missing data. My published effect sizes were wrong
in a specific and instructive way:

| Claim | First reported | After repair |
|---|--:|--:|
| bf16 MMLU-Pro absolute | 0.4089 | **0.8313** |
| NVFP4-Inferact vs bf16 | −11.41 | **−2.24** |
| Gap between NVFP4 checkpoints | 6.75 (p=3.2×10⁻⁵¹) | **2.01 (n.s.)** |

The stop string cost items, and it cost them **at different rates per
checkpoint**. So a *termination* difference was being measured and published as
an *accuracy* difference, with a p-value of 10⁻⁵¹ attached.

Two things I'd take from that:

**A benchmark returning a plausible number is not evidence it works.** The three
zeros were obvious. Minerva was the dangerous one — it returned 0.1926, a number
nobody questions, while measuring output formatting rather than mathematics. I
now check that extracted answers are *right*, not that the aggregate is
non-zero.

**Track non-termination as a first-class metric.** Every eval harness I know
scores an empty response identically to a wrong one. That single conflation
produced the false effect sizes above and inverted an entire checkpoint's
ranking. It costs one counter to fix.

## Conclusions

In general quantization performs really well, and it is almost indistinguishable
from the original model, but you have to be careful because sometimes this is not
the case, and it is not easy to spot.

In the case of **Qwen3.8-27B on Blackwell**:

- **FP8 is free.** Indistinguishable from bf16 on a six-benchmark suite *and* on
  500 agentic coding tasks. There is no accuracy argument against it, and on
  this hardware nothing else at eight bits even runs.
- **4-bit is free too — if it's W4A16.** Both integer 4-bit checkpoints landed
  on the reference, as did partial NVFP4 (MLP-only).
- **Read `config_groups` before you trust a label.** Three checkpoints called
  "NVFP4" implemented three different schemes, spanning eight points of
  real-world capability.
- **Test on something agentic.** The accuracy suite showed 2.24 points where
  SWE-bench showed 8.0, and half the suite was too saturated to show anything at
  all.
