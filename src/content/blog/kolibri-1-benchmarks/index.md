---
title: 'Does Kolibri-1 live up to its benchmarks?'
description: "Re-running Aleph Alpha's Kolibri-1 claims on H200, with Qwen3.6, Qwen3.8, DeepSeek-V4.1-Flash and Kimi-K3 through the same harness."
publishDate: 2026-10-08
tags:
  - ai
  - llm
  - gpu
  - benchmarking
  - evaluation
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

On 3 October Aleph Alpha released [Kolibri-1](https://huggingface.co/Aleph-Alpha/Kolibri-1),
an open-weight, Apache-2.0 model built and trained in Germany: a 78.1B-parameter
mixture of experts with **3.46B active parameters per token**, bilingual English and
German, a 256k-token native window and a claimed 1M with extrapolation. The
[release post](https://aleph-alpha.com/en/blog/kolibri-has-landed-a-sovereign-open-weight-model/)
and a 189-page technical report claim it sits on the Pareto frontier of quality
versus serving cost, ahead of every other 3B-active model and level with models that
activate four times as many parameters.

Those are self-reported numbers, so I re-ran them.

**Most of them hold.** Of the fourteen headline benchmarks I could re-run, thirteen
landed within about two points of the published figure, in both directions. Code is
the weak spot, with LiveCodeBench 3.7 points low. One agentic number turned out not to
be comparable, and the 1M-token context could not be exercised at all on the public
serving stack.

## Method

### What to test

The report evaluates around forty benchmarks. I kept the ones that have public data,
can be scored by rule or by a published judge prompt, and carry a headline claim:

| group | benchmarks |
|---|---|
| knowledge | GPQA Diamond, MMLU-Pro (EN), MMLU-ProX (DE), AA-Omniscience |
| maths | AIME 2025 and 2026, in English and in German |
| code | LiveCodeBench v6, HumanEval+ |
| instruction following | IFBench |
| agentic | Tau2-Bench telecom, retail, airline |
| long context | AA-LCR, RULER |
| German | tokenizer compression, language of the reasoning trace |
| efficiency | throughput on 1 and 2 GPUs, "18 concurrent 256k-token queries on two H100s" |

Left out: the five in-house "industry RAG" proxies and the Merlin–Arthur grounding
score, whose data is not public; SWE-Bench Verified, TerminalBench and BrowseComp,
which need agent scaffolds and hundreds of per-task containers; Tau3 banking, whose
retrieval setup needs OpenAI embeddings; and German GPQA, which is gated.

### How

Aleph Alpha publishes its evaluation harness,
[eval-framework](https://github.com/Aleph-Alpha-Research/eval-framework), but its
OpenAI-compatible backend cannot pass `top_k` or the reasoning effort, both of which
the model card asks for. So I wrote a small asynchronous runner and copied the
prompts, the answer-option shuffles and the answer extractors from eval-framework
verbatim. The settings follow Table 27 of the report: avg@8 on GPQA, avg@16 on AIME,
avg@3 on LiveCodeBench and Tau2, avg@4 on HumanEval+, avg@5 on IFBench. That leaves
the serving stack and sampling noise as the main differences.

The model ran FP8 on vLLM 0.29.1 with Aleph Alpha's `aleph-alpha-inference` 1.0.0
plugin, data-parallel over four H200s, at the model card's sampling (temperature 1.0,
top-p 0.97, top-k 128) and reasoning effort *high*. The judge for the two Artificial
Analysis benchmarks and the simulated customer in Tau2 were DeepSeek-V4.1-Flash with
thinking switched off — Aleph Alpha used Gemini 2.5 Flash, GPT-5.6 and gpt-4.1.

## The claims, re-run

<figure class="qz">
  <svg viewBox="0 0 640 438" role="img" aria-label="Measured minus claimed score for 14 Kolibri-1 benchmarks. GPQA Diamond EN: claimed 84.3, measured 85.6; MMLU-Pro EN: claimed 80.0, measured 81.9; MMLU-ProX DE: claimed 75.5, measured 77.7; AIME 2025 EN: claimed 96.9, measured 95.6; AIME 2026 EN: claimed 96.0, measured 95.2; AIME 2025 DE: claimed 87.5, measured 88.1; AIME 2026 DE: claimed 90.0, measured 91.3; IFBench loose prompt: claimed 78.1, measured 78.8; Tau2 telecom avg@3: claimed 94.7, measured 95.0; Tau2 airline avg@3: claimed 76.7, measured 74.7; AA-Omniscience accuracy, public set: claimed 14.8, measured 14.5; AA-LCR avg@3: claimed 68.3, measured 66.7; HumanEval+ avg@4: claimed 92.7, measured 90.7; LiveCodeBench v6 2024-08 → 2025-04: claimed 85.9, measured 82.2.">
    <line x1="316" y1="28" x2="316" y2="394" stroke="var(--grid)"/>
    <text class="sub" x="316" y="24" text-anchor="middle">-4</text>
    <line x1="391" y1="28" x2="391" y2="394" stroke="var(--grid)"/>
    <text class="sub" x="391" y="24" text-anchor="middle">-2</text>
    <line x1="466" y1="28" x2="466" y2="394" stroke-width="1.5" stroke="var(--dim)"/>
    <text class="sub" x="466" y="24" text-anchor="middle">0</text>
    <line x1="541" y1="28" x2="541" y2="394" stroke="var(--grid)"/>
    <text class="sub" x="541" y="24" text-anchor="middle">+2</text>
    <line x1="616" y1="28" x2="616" y2="394" stroke="var(--grid)"/>
    <text class="sub" x="616" y="24" text-anchor="middle">+4</text>
    <text class="lab" x="188" y="43" text-anchor="end">GPQA Diamond</text>
    <text class="sub" x="188" y="54" text-anchor="end">EN · avg@8</text>
    <text class="sub" x="270" y="48" text-anchor="end">84.3 → 85.6</text>
    <rect x="466.0" y="39" width="48.8" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="520" y="48" text-anchor="start">+1.3</text>
    <text class="lab" x="188" y="69" text-anchor="end">MMLU-Pro</text>
    <text class="sub" x="188" y="80" text-anchor="end">EN · 12,032 items</text>
    <text class="sub" x="270" y="74" text-anchor="end">80.0 → 81.9</text>
    <rect x="466.0" y="65" width="71.2" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="542" y="74" text-anchor="start">+1.9</text>
    <text class="lab" x="188" y="95" text-anchor="end">MMLU-ProX</text>
    <text class="sub" x="188" y="106" text-anchor="end">DE · 11,759 items</text>
    <text class="sub" x="270" y="100" text-anchor="end">75.5 → 77.7</text>
    <rect x="466.0" y="91" width="82.5" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="554" y="100" text-anchor="start">+2.2</text>
    <text class="lab" x="188" y="121" text-anchor="end">AIME 2025</text>
    <text class="sub" x="188" y="132" text-anchor="end">EN · avg@16</text>
    <text class="sub" x="270" y="126" text-anchor="end">96.9 → 95.6</text>
    <rect x="417.2" y="117" width="48.8" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="412" y="126" text-anchor="end">-1.3</text>
    <text class="lab" x="188" y="147" text-anchor="end">AIME 2026</text>
    <text class="sub" x="188" y="158" text-anchor="end">EN · avg@16</text>
    <text class="sub" x="270" y="152" text-anchor="end">96.0 → 95.2</text>
    <rect x="436.0" y="143" width="30.0" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="431" y="152" text-anchor="end">-0.8</text>
    <text class="lab" x="188" y="173" text-anchor="end">AIME 2025</text>
    <text class="sub" x="188" y="184" text-anchor="end">DE · avg@16</text>
    <text class="sub" x="270" y="178" text-anchor="end">87.5 → 88.1</text>
    <rect x="466.0" y="169" width="22.5" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="494" y="178" text-anchor="start">+0.6</text>
    <text class="lab" x="188" y="199" text-anchor="end">AIME 2026</text>
    <text class="sub" x="188" y="210" text-anchor="end">DE · avg@16</text>
    <text class="sub" x="270" y="204" text-anchor="end">90.0 → 91.3</text>
    <rect x="466.0" y="195" width="48.8" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="520" y="204" text-anchor="start">+1.3</text>
    <text class="lab" x="188" y="225" text-anchor="end">IFBench</text>
    <text class="sub" x="188" y="236" text-anchor="end">loose prompt · avg@5</text>
    <text class="sub" x="270" y="230" text-anchor="end">78.1 → 78.8</text>
    <rect x="466.0" y="221" width="26.2" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="497" y="230" text-anchor="start">+0.7</text>
    <text class="lab" x="188" y="251" text-anchor="end">Tau2 telecom</text>
    <text class="sub" x="188" y="262" text-anchor="end">avg@3</text>
    <text class="sub" x="270" y="256" text-anchor="end">94.7 → 95.0</text>
    <rect x="466.0" y="247" width="11.2" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="482" y="256" text-anchor="start">+0.3</text>
    <text class="lab" x="188" y="277" text-anchor="end">Tau2 airline</text>
    <text class="sub" x="188" y="288" text-anchor="end">avg@3</text>
    <text class="sub" x="270" y="282" text-anchor="end">76.7 → 74.7</text>
    <rect x="391.0" y="273" width="75.0" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="386" y="282" text-anchor="end">-2.0</text>
    <text class="lab" x="188" y="303" text-anchor="end">AA-Omniscience</text>
    <text class="sub" x="188" y="314" text-anchor="end">accuracy, public set</text>
    <text class="sub" x="270" y="308" text-anchor="end">14.8 → 14.5</text>
    <rect x="454.8" y="299" width="11.2" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="450" y="308" text-anchor="end">-0.3</text>
    <text class="lab" x="188" y="329" text-anchor="end">AA-LCR</text>
    <text class="sub" x="188" y="340" text-anchor="end">avg@3</text>
    <text class="sub" x="270" y="334" text-anchor="end">68.3 → 66.7</text>
    <rect x="406.0" y="325" width="60.0" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="401" y="334" text-anchor="end">-1.6</text>
    <text class="lab" x="188" y="355" text-anchor="end">HumanEval+</text>
    <text class="sub" x="188" y="366" text-anchor="end">avg@4</text>
    <text class="sub" x="270" y="360" text-anchor="end">92.7 → 90.7</text>
    <rect x="391.0" y="351" width="75.0" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="386" y="360" text-anchor="end">-2.0</text>
    <text class="lab" x="188" y="381" text-anchor="end">LiveCodeBench v6</text>
    <text class="sub" x="188" y="392" text-anchor="end">2024-08 → 2025-04 · avg@3</text>
    <text class="sub" x="270" y="386" text-anchor="end">85.9 → 82.2</text>
    <rect x="327.2" y="377" width="138.8" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="322" y="386" text-anchor="end">-3.7</text>
    <text class="ax" x="316" y="430" text-anchor="start">measured − claimed, percentage points</text>
  </svg>

  <figcaption>
    Measured minus claimed, for every headline number re-run with the report's own datasets, prompts and avg@k. Green is above the claim, orange below. Where samples hit the 65,536-token output cap they were re-run at 196k; see the end of the post.
  </figcaption>
</figure>

Thirteen of the fourteen are inside ±2.2 points, in both directions, which is what
re-running a sampled benchmark looks like. The intervals are wide: the 95% bootstrap
interval on 30 AIME problems at avg@16 is ±4 to ±9 points, and on GPQA's 198
questions about ±4. Nothing in the knowledge, maths or instruction-following rows
suggests the published numbers were inflated.

| benchmark | claimed | measured | 95% interval |
|---|--:|--:|--:|
| GPQA Diamond (EN) | 84.3 | 85.6 | 81.0–89.6 |
| MMLU-Pro (EN) | 80.0 | 81.9 | 81.2–82.6 |
| MMLU-ProX (DE) | 75.5 | 77.7 | 76.9–78.4 |
| AIME 2025 (EN) | 96.9 | 95.6 | 91.0–99.2 |
| AIME 2026 (EN) | 96.0 | 95.2 | 88.3–99.4 |
| AIME 2025 (DE) | 87.5 | 88.1 | 78.5–95.6 |
| AIME 2026 (DE) | 90.0 | 91.3 | 82.1–97.7 |
| IFBench (loose prompt) | 78.1 | 78.8 | |
| AA-Omniscience accuracy / index / non-hallucination | 14.8 / −32.8 / 44.0 | 14.5 / −36.0 / 40.9 | |
| AA-LCR | 68.3 | 66.7 | |
| Tau2 telecom / airline | 94.7 / 76.7 | 95.0 / 74.7 | |
| LiveCodeBench v6 | 85.9 | 82.2 | |
| HumanEval+ | 92.7 | 90.7 | |
| Tau2 retail | 69.9 | 86.8 | not comparable |

Code is the one area where both benchmarks come in below the claim. HumanEval+ is
the easier one to explain: eval-framework sends it as a bare code-fence completion,
which is an odd thing to give a chat model, and I used evalplus's standard instruction
prompt instead. On LiveCodeBench the model solves 98.5% of the easy problems and 92.9%
of the medium ones; the gap is all in the hard third, at 65.9%.

IFBench deserves a mention because it is the most surprising claim in the report —
Kolibri is 12 points ahead of Qwen3.6-35B-A3B on it — and it reproduces exactly.

## Against other models, through the same harness

A reproduction says the numbers are real; it does not say where they put the model.
For that I ran four other models through the identical prompts, datasets and scoring,
each at its own model card's sampling and its maximum reasoning effort:

- **Qwen3.6-35B-A3B**, the 3B-active model the release post compares against;
- **Qwen3.8-27B**, a dense model and the strongest entry in Aleph Alpha's own table;
- **DeepSeek-V4.1-Flash** and **Kimi-K3**, two much larger frontier MoEs already being
  served on our nodes.

<figure class="qz">
  <svg viewBox="0 0 640 700" role="img" aria-label="Scores of five models on six benchmarks, same harness. GPQA Diamond EN: Kolibri-1 82.6, Qwen3.6-35B-A3B 83.2, Qwen3.8-27B 89.3, DeepSeek-V4.1-Flash 89.1, Kimi-K3 92.6; AIME 2025 EN: Kolibri-1 95.6, Qwen3.6-35B-A3B 87.5, Qwen3.8-27B 97.7, DeepSeek-V4.1-Flash 98.8, Kimi-K3 93.8; AIME 2026 EN: Kolibri-1 95.2, Qwen3.6-35B-A3B 88.3, Qwen3.8-27B 95.6, DeepSeek-V4.1-Flash 98.8, Kimi-K3 95.2; AIME 2025 DE: Kolibri-1 88.1, Qwen3.6-35B-A3B not run, Qwen3.8-27B 97.1, DeepSeek-V4.1-Flash 98.8, Kimi-K3 95.8; AIME 2026 DE: Kolibri-1 90.8, Qwen3.6-35B-A3B not run, Qwen3.8-27B 96.9, DeepSeek-V4.1-Flash 96.9, Kimi-K3 96.5; IFBench: Kolibri-1 78.8, Qwen3.6-35B-A3B 64.1, Qwen3.8-27B 81.7, DeepSeek-V4.1-Flash 79.7, Kimi-K3 76.5.">
    <line x1="190" y1="20" x2="190" y2="670" stroke="var(--grid)"/>
    <text class="sub" x="190" y="16" text-anchor="middle">60</text>
    <line x1="290" y1="20" x2="290" y2="670" stroke="var(--grid)"/>
    <text class="sub" x="290" y="16" text-anchor="middle">70</text>
    <line x1="390" y1="20" x2="390" y2="670" stroke="var(--grid)"/>
    <text class="sub" x="390" y="16" text-anchor="middle">80</text>
    <line x1="490" y1="20" x2="490" y2="670" stroke="var(--grid)"/>
    <text class="sub" x="490" y="16" text-anchor="middle">90</text>
    <line x1="590" y1="20" x2="590" y2="670" stroke="var(--grid)"/>
    <text class="sub" x="590" y="16" text-anchor="middle">100</text>
    <text class="lab" x="182" y="40" text-anchor="end" font-weight="600">GPQA Diamond EN</text>
    <text class="sub" x="182" y="55" text-anchor="end" font-weight="700" fill="var(--ok)">Kolibri-1</text>
    <rect x="190" y="46" width="226.0" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="421" y="55" text-anchor="start">82.6</text>
    <text class="sub" x="182" y="69" text-anchor="end">Qwen3.6-35B-A3B</text>
    <rect x="190" y="60" width="232.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="427" y="69" text-anchor="start">83.2</text>
    <text class="sub" x="182" y="83" text-anchor="end">Qwen3.8-27B</text>
    <rect x="190" y="74" width="293.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="488" y="83" text-anchor="start">89.3</text>
    <text class="sub" x="182" y="97" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <rect x="190" y="88" width="291.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="486" y="97" text-anchor="start">89.1</text>
    <text class="sub" x="182" y="111" text-anchor="end">Kimi-K3</text>
    <rect x="190" y="102" width="326.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="521" y="111" text-anchor="start">92.6</text>
    <text class="lab" x="182" y="148" text-anchor="end" font-weight="600">AIME 2025 EN</text>
    <text class="sub" x="182" y="163" text-anchor="end" font-weight="700" fill="var(--ok)">Kolibri-1</text>
    <rect x="190" y="154" width="356.0" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="551" y="163" text-anchor="start">95.6</text>
    <text class="sub" x="182" y="177" text-anchor="end">Qwen3.6-35B-A3B</text>
    <rect x="190" y="168" width="275.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="470" y="177" text-anchor="start">87.5</text>
    <text class="sub" x="182" y="191" text-anchor="end">Qwen3.8-27B</text>
    <rect x="190" y="182" width="377.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="572" y="191" text-anchor="start">97.7</text>
    <text class="sub" x="182" y="205" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <rect x="190" y="196" width="388.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="583" y="205" text-anchor="start">98.8</text>
    <text class="sub" x="182" y="219" text-anchor="end">Kimi-K3</text>
    <rect x="190" y="210" width="338.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="533" y="219" text-anchor="start">93.8</text>
    <text class="lab" x="182" y="256" text-anchor="end" font-weight="600">AIME 2026 EN</text>
    <text class="sub" x="182" y="271" text-anchor="end" font-weight="700" fill="var(--ok)">Kolibri-1</text>
    <rect x="190" y="262" width="352.0" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="547" y="271" text-anchor="start">95.2</text>
    <text class="sub" x="182" y="285" text-anchor="end">Qwen3.6-35B-A3B</text>
    <rect x="190" y="276" width="283.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="478" y="285" text-anchor="start">88.3</text>
    <text class="sub" x="182" y="299" text-anchor="end">Qwen3.8-27B</text>
    <rect x="190" y="290" width="356.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="551" y="299" text-anchor="start">95.6</text>
    <text class="sub" x="182" y="313" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <rect x="190" y="304" width="388.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="583" y="313" text-anchor="start">98.8</text>
    <text class="sub" x="182" y="327" text-anchor="end">Kimi-K3</text>
    <rect x="190" y="318" width="352.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="547" y="327" text-anchor="start">95.2</text>
    <text class="lab" x="182" y="364" text-anchor="end" font-weight="600">AIME 2025 DE</text>
    <text class="sub" x="182" y="379" text-anchor="end" font-weight="700" fill="var(--ok)">Kolibri-1</text>
    <rect x="190" y="370" width="281.0" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="476" y="379" text-anchor="start">88.1</text>
    <text class="sub" x="182" y="393" text-anchor="end">Qwen3.6-35B-A3B</text>
    <text class="sub" x="194" y="393">not run</text>
    <text class="sub" x="182" y="407" text-anchor="end">Qwen3.8-27B</text>
    <rect x="190" y="398" width="371.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="566" y="407" text-anchor="start">97.1</text>
    <text class="sub" x="182" y="421" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <rect x="190" y="412" width="388.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="583" y="421" text-anchor="start">98.8</text>
    <text class="sub" x="182" y="435" text-anchor="end">Kimi-K3</text>
    <rect x="190" y="426" width="358.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="553" y="435" text-anchor="start">95.8</text>
    <text class="lab" x="182" y="472" text-anchor="end" font-weight="600">AIME 2026 DE</text>
    <text class="sub" x="182" y="487" text-anchor="end" font-weight="700" fill="var(--ok)">Kolibri-1</text>
    <rect x="190" y="478" width="308.0" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="503" y="487" text-anchor="start">90.8</text>
    <text class="sub" x="182" y="501" text-anchor="end">Qwen3.6-35B-A3B</text>
    <text class="sub" x="194" y="501">not run</text>
    <text class="sub" x="182" y="515" text-anchor="end">Qwen3.8-27B</text>
    <rect x="190" y="506" width="369.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="564" y="515" text-anchor="start">96.9</text>
    <text class="sub" x="182" y="529" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <rect x="190" y="520" width="369.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="564" y="529" text-anchor="start">96.9</text>
    <text class="sub" x="182" y="543" text-anchor="end">Kimi-K3</text>
    <rect x="190" y="534" width="365.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="560" y="543" text-anchor="start">96.5</text>
    <text class="lab" x="182" y="580" text-anchor="end" font-weight="600">IFBench</text>
    <text class="sub" x="182" y="595" text-anchor="end" font-weight="700" fill="var(--ok)">Kolibri-1</text>
    <rect x="190" y="586" width="188.0" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="383" y="595" text-anchor="start">78.8</text>
    <text class="sub" x="182" y="609" text-anchor="end">Qwen3.6-35B-A3B</text>
    <rect x="190" y="600" width="41.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="236" y="609" text-anchor="start">64.1</text>
    <text class="sub" x="182" y="623" text-anchor="end">Qwen3.8-27B</text>
    <rect x="190" y="614" width="217.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="412" y="623" text-anchor="start">81.7</text>
    <text class="sub" x="182" y="637" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <rect x="190" y="628" width="197.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="392" y="637" text-anchor="start">79.7</text>
    <text class="sub" x="182" y="651" text-anchor="end">Kimi-K3</text>
    <rect x="190" y="642" width="165.0" height="11" rx="2" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="360" y="651" text-anchor="start">76.5</text>
    <text class="ax" x="190" y="692" text-anchor="start">score · axis starts at 60 · Kolibri highlighted</text>
  </svg>

  <figcaption>
    Five models through the identical harness, each at its model card's sampling and maximum reasoning effort, all at the same output caps (65k on GPQA, 131k on AIME). The axis starts at 60. Qwen3.6 was not run on German AIME.
  </figcaption>
</figure>

The Qwen runs double as a check on the harness. For Qwen3.8-27B I measured 89.3 / 97.7
/ 81.7 on GPQA / AIME 2025 / IFBench; the report has 89.2 / 97.9 / 81.9 for the same
model. Qwen3.6 lands within three points of its reported numbers too. If the harness
were systematically harder or easier than Aleph Alpha's, the comparators would show
it.

Against its own size class the result is clear: Kolibri beats Qwen3.6-35B-A3B by 7–8
points on English AIME and by 15 on IFBench, and ties it on GPQA. Against the large
models it trails by 6–10 points on GPQA and 5–11 on German AIME, but it matches
Kimi-K3 on English AIME and beats it on IFBench — at 3.46B active parameters against
a model with 2.78T in total.

## German

The German claims are the ones most specific to this model, and they hold. German
AIME runs 4–8 points under English, MMLU-ProX 4 points under MMLU-Pro, the same gaps
the report shows.

The chat template tells the model to think "in the user's language", and it does:
the reasoning trace is German in all 960 German AIME samples and in 97.6% of the
11,759 MMLU-ProX ones.

The tokenizer is where the sovereignty argument becomes a cost argument.

<figure class="qz">
  <svg viewBox="0 0 640 236" role="img" aria-label="UTF-8 bytes per token on German and English web text. Kolibri-1: German 4.87, English 4.54; Qwen3.8-Flash-Next: German 4.15, English 4.43; GLM-5.3: German 3.92, English 4.58; DeepSeek-V4.1-Flash: German 3.73, English 4.56; MiMo-V2.6-Pro: German 3.58, English 4.49.">
    <rect x="170" y="10" width="13" height="8" rx="2" fill="var(--ok)"/><text class="sub" x="188" y="17">German · FineWeb-2</text>
    <rect x="310" y="10" width="13" height="8" rx="2" fill="var(--vl)"/><text class="sub" x="328" y="17">English · FineWeb</text>
    <line x1="170" y1="32" x2="170" y2="204" stroke="var(--grid)"/>
    <text class="sub" x="170" y="216" text-anchor="middle">0</text>
    <line x1="252" y1="32" x2="252" y2="204" stroke="var(--grid)"/>
    <text class="sub" x="252" y="216" text-anchor="middle">1</text>
    <line x1="334" y1="32" x2="334" y2="204" stroke="var(--grid)"/>
    <text class="sub" x="334" y="216" text-anchor="middle">2</text>
    <line x1="416" y1="32" x2="416" y2="204" stroke="var(--grid)"/>
    <text class="sub" x="416" y="216" text-anchor="middle">3</text>
    <line x1="498" y1="32" x2="498" y2="204" stroke="var(--grid)"/>
    <text class="sub" x="498" y="216" text-anchor="middle">4</text>
    <line x1="580" y1="32" x2="580" y2="204" stroke="var(--grid)"/>
    <text class="sub" x="580" y="216" text-anchor="middle">5</text>
    <text class="lab" x="162" y="53" text-anchor="end" font-weight="700">Kolibri-1</text>
    <rect x="170" y="40" width="399.3" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="574" y="49">4.87</text>
    <rect x="170" y="53" width="372.3" height="11" rx="2" fill="var(--vl)"/>
    <text class="val" x="547" y="62">4.54</text>
    <text class="lab" x="162" y="87" text-anchor="end">Qwen3.8-Flash-Next</text>
    <rect x="170" y="74" width="340.3" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="515" y="83">4.15</text>
    <rect x="170" y="87" width="363.3" height="11" rx="2" fill="var(--vl)"/>
    <text class="val" x="538" y="96">4.43</text>
    <text class="lab" x="162" y="121" text-anchor="end">GLM-5.3</text>
    <rect x="170" y="108" width="321.4" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="496" y="117">3.92</text>
    <rect x="170" y="121" width="375.6" height="11" rx="2" fill="var(--vl)"/>
    <text class="val" x="551" y="130">4.58</text>
    <text class="lab" x="162" y="155" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <rect x="170" y="142" width="305.9" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="481" y="151">3.73</text>
    <rect x="170" y="155" width="373.9" height="11" rx="2" fill="var(--vl)"/>
    <text class="val" x="549" y="164">4.56</text>
    <text class="lab" x="162" y="189" text-anchor="end">MiMo-V2.6-Pro</text>
    <rect x="170" y="176" width="293.6" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="469" y="185">3.58</text>
    <rect x="170" y="189" width="368.2" height="11" rx="2" fill="var(--vl)"/>
    <text class="val" x="543" y="198">4.49</text>
    <text class="ax" x="170" y="234" text-anchor="start">UTF-8 bytes per token · higher is better · 5,000 documents each</text>
  </svg>

  <figcaption>
    UTF-8 bytes per token on 5,000 documents of German web text (FineWeb-2) and English web text (FineWeb), the corpora the report uses. Higher means fewer tokens for the same text.
  </figcaption>
</figure>

On German web text Kolibri's tokenizer packs 4.87 bytes into a token, against 3.73
for DeepSeek's. The same German document is about 23% fewer tokens — 23% less
prefill, KV cache and output to pay for — with no meaningful loss on English. My numbers are
within 0.04 of the report's own figures for Kolibri (4.90 / 4.58), DeepSeek and GLM. The Hugging Face model card quotes 4.7 / 4.2, which matches neither.

## Long context

On AA-LCR — 100 questions over document sets of about 100k tokens — Kolibri scores
66.7 against a claimed 68.3.

RULER is murkier than the model card suggests. The card's headline long-context
numbers, 63.2 at 1M tokens and 65.5 at 512k, are for the **base** checkpoint, which
has not been released. For the released model the report only shows a figure. I
measured it with reasoning off, as the report recommends for retrieval:

| context | RULER (mean of NIAH, VT, WE, QA) | needle-in-a-haystack |
|--:|--:|--:|
| 128k | 82.9 | 98.6 |
| 252k | 86.4 | 99.7 |
| 512k, 1M | server crashes | |

Inside the trained window the model holds up: no decay from 128k to the edge of the
window. Beyond it I could not get an answer. Any vLLM server started with
`max_model_len` above 262,144 died with a FlashAttention device-side assert as soon as
prompts approached the window. That may be fixable in the inference stack, but on
the stack Aleph Alpha ships today the 1M-token window is not usable.

## Throughput

Same seven workload shapes as [the H200 model comparison](/blog/llm-benchmarking-on-h200/),
so the numbers sit on the same footing.

<figure class="qz">
  <svg viewBox="0 0 640 296" role="img" aria-label="Output tokens per second on BATCH-D at 64 concurrent requests: Kolibri-1 on 2× H200 6,551, Qwen3.8-Flash-Next FP8 on 8× H200 6,475, DeepSeek-V4.1-Flash on 8× H200 5,700, GLM-5.3-Flash on 8× H200 5,433, MiMo-V2.6-Flash-RL on 8× H200 4,950, Kolibri-1 on 1× H200 4,765, GLM-5.3 on 8× H200 4,009, MiMo-V2.6-Pro-RL on 8× H200 3,198.">
    <line x1="232" y1="20" x2="232" y2="262" stroke="var(--grid)"/>
    <text class="sub" x="232" y="16" text-anchor="middle">0</text>
    <line x1="331" y1="20" x2="331" y2="262" stroke="var(--grid)"/>
    <text class="sub" x="331" y="16" text-anchor="middle">2,000</text>
    <line x1="431" y1="20" x2="431" y2="262" stroke="var(--grid)"/>
    <text class="sub" x="431" y="16" text-anchor="middle">4,000</text>
    <line x1="530" y1="20" x2="530" y2="262" stroke="var(--grid)"/>
    <text class="sub" x="530" y="16" text-anchor="middle">6,000</text>
    <text class="lab" x="224" y="36" text-anchor="end" font-weight="700">Kolibri-1</text>
    <text class="sub" x="224" y="46" text-anchor="end">2× H200 · vLLM TP2</text>
    <rect x="232" y="30" width="325.7" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="564" y="41">6,551</text>
    <text class="lab" x="224" y="66" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="76" text-anchor="end">8× H200 · vLLM balanced</text>
    <rect x="232" y="60" width="321.9" height="14" rx="3" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="560" y="71">6,475</text>
    <text class="lab" x="224" y="96" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="106" text-anchor="end">8× H200 · vLLM DEP8</text>
    <rect x="232" y="90" width="283.4" height="14" rx="3" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="521" y="101">5,700</text>
    <text class="lab" x="224" y="126" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="136" text-anchor="end">8× H200 · vLLM TEP8</text>
    <rect x="232" y="120" width="270.1" height="14" rx="3" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="508" y="131">5,433</text>
    <text class="lab" x="224" y="156" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="166" text-anchor="end">8× H200 · vLLM TEP8</text>
    <rect x="232" y="150" width="246.1" height="14" rx="3" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="484" y="161">4,950</text>
    <text class="lab" x="224" y="186" text-anchor="end" font-weight="700">Kolibri-1</text>
    <text class="sub" x="224" y="196" text-anchor="end">1× H200 · vLLM TP1</text>
    <rect x="232" y="180" width="236.9" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="475" y="191">4,765</text>
    <text class="lab" x="224" y="216" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="226" text-anchor="end">8× H200 · SGLang throughput</text>
    <rect x="232" y="210" width="199.3" height="14" rx="3" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="437" y="221">4,009</text>
    <text class="lab" x="224" y="246" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="256" text-anchor="end">8× H200 · vLLM + DFlash</text>
    <rect x="232" y="240" width="159.0" height="14" rx="3" fill="var(--dim)" opacity=".45"/>
    <text class="val" x="397" y="251">3,198</text>
    <text class="ax" x="232" y="290" text-anchor="start">BATCH-D · 4,096 in / 8,192 out · 64 concurrent · output tok/s</text>
  </svg>

  <figcaption>
    Output tokens per second on BATCH-D (4,096 in → 8,192 out) at 64 concurrent requests. The other models are each one's best configuration from the earlier 8×H200 comparison.
  </figcaption>
</figure>

One H200 serving Kolibri produces 4,765 output tokens per second on BATCH-D at 64
concurrent requests. That is in the same range as what **eight** H200s produced
serving GLM-5.3-Flash, MiMo-V2.6-Flash or DeepSeek-V4.1-Flash. Two GPUs take it to
6,551, the highest of any configuration I have measured on this workload. Sixty-four streams do not saturate
an 8-GPU deployment, so this compares cost at a fixed load, not peak capacity — but
that is the comparison most deployments care about.

| workload | in → out | streams | 1× H200 | 2× H200 |
|---|---|--:|--:|--:|
| CHAT-S | 2,048 → 512 | 1 | 177 | 174 |
| CHAT-S | 2,048 → 512 | 64 | 3,584 | 5,656 |
| BATCH-D | 4,096 → 8,192 | 64 | 4,765 | 6,551 |
| BATCH-D | 4,096 → 8,192 | 256 | 6,024 | 8,733 |
| CODE-I | 16,384 → 2,048 | 64 | 2,490 | 3,726 |
| CHAT-L | 32,768 → 2,048 | 64 | 1,526 | 2,756 |
| CODE-A | 65,536 → 4,096 | 64 | 1,005 | 1,901 |
| DOC-L | 131,072 → 2,048 | 64 | 271 | 506 |

A single stream decodes at about 175 tokens per second on one GPU or two — with only
3.46B parameters active per token there is little for tensor parallelism to split.

The release post also claims Kolibri serves **18 concurrent 256k-token queries on two
H100s**. I capped two H200s at 78 GB each to stand in for H100s. vLLM reports room for
24.7 concurrent 262,144-token requests, and 18 distinct ~242k-token prompts ran
together with no preemption and the KV cache at 55%. The sliding-window attention
in four of every five layers is doing what it was designed to do.

## What went wrong along the way

Three things would have produced wrong numbers without an error, and they are worth
knowing for anyone running these benchmarks.

**The output cap moves scores by three points.** 5.6% of GPQA samples and 9.4% of
LiveCodeBench samples ran past 65,536 tokens. They were not loops — the tails
compress like normal reasoning — just long deliberations on hard items. Re-running
only those samples with a 196k cap took GPQA from 82.6 to 85.6 and LiveCodeBench from
79.1 to 82.2. A report that does not state its cap leaves a 3-point ambiguity.

**Tau2 retail silently called OpenAI.** In the current tau2 release, retail tasks are
scored partly by a natural-language assertion grader that is hard-coded to gpt-4.1.
With a placeholder key, 120 of 342 simulations died during grading and were recorded
with reward 0, for a score of 57.6. Pointing the grader at a local model gave 86.8.

**Tau2 changed underneath the claim.** Version 1.0 rewrote 26 retail tasks and 14
airline ones — ambiguous instructions, impossible constraints, wrong expected
actions. The 86.8 is on the corrected set; the published 69.9 was very likely on an
earlier one, so the two cannot be compared. Telecom and airline were barely touched
and match.

## Conclusion

Kolibri-1 is what Aleph Alpha says it is on the benchmarks that can be checked. Its
reasoning, maths and instruction-following numbers reproduce, it is genuinely better
than the other 3B-active model in the comparison, and it does it while reading
German in about a quarter fewer tokens than the large models it trails. At a fixed
load of 64 streams, one GPU running it delivers what an 8-GPU deployment of a large
MoE did.

It is not a frontier model, and the report does not claim it is: the dense
Qwen3.8-27B and the large MoEs are 6–10 points ahead on GPQA. And the 1M-token window
is, for now, a property of the base checkpoint and of Aleph Alpha's own stack, not of
the release.

## Caveats

- **Different judge and user simulator.** AA-Omniscience, AA-LCR and Tau2 depend on
  a model in the loop; I used DeepSeek-V4.1-Flash for all three. For the same
  reason DeepSeek was not compared on those benchmarks.
- **German AIME prompt.** eval-framework's German template is not public; I
  translated the English one.
- **GPQA source.** The English GPQA Diamond questions come from OpenAI's simple-evals
  copy because the original dataset is gated. They are the same 198 questions.
- **Weights.** I evaluated revision `e52eb46`. The repository has since moved to
  `35bc4d3`, which changes only the README.
- **One serving stack.** Everything here is vLLM on H200 (SM90). The long-context
  crash in particular may be specific to this engine version.
