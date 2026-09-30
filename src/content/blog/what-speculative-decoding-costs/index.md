---
title: 'What speculative decoding actually costs'
description: 'Speculation off versus on for five mechanisms on 8×H200 — and why it loses money under load.'
publishDate: 2026-09-30
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

Every large model shipped in the last year comes with a speculative decoder —
EAGLE, MTP, NEXTN, DSpark, DFlash — and every published serving recipe turns it
on. It is presented as free throughput: draft a few tokens cheaply, verify them
in one pass, emit whatever the model would have emitted anyway.

It is not free. On a loaded server it is a **net loss** for every mechanism I
measured, and switching it off is worth 5–36%.

This post is the A/B test behind that claim, on [the same 8×H200 node and the
same workload shapes](/blog/llm-benchmarking-on-h200/) as the model comparison.

## The measurement

Six model-and-mechanism pairs. Each speculation-off configuration is an
existing, already-measured configuration with **only its speculative flags
removed** — same engine, same image, same tensor / expert / data parallelism,
same memory fraction, same parsers. Nothing else moves, so the pair isolates
the mechanism instead of comparing two differently-tuned servers.

| model | engine | mechanism |
|---|---|---|
| DeepSeek-V4.1-Flash | vLLM | DSpark, 5 draft tokens |
| GLM-5.3 | SGLang | EAGLE, 5 steps / 6 draft tokens |
| GLM-5.3 | vLLM | MTP, 5 draft tokens |
| GLM-5.3-Flash | SGLang | EAGLE, 5 steps / 6 draft tokens |
| GLM-5.3-Flash | vLLM | MTP, 5 draft tokens |
| Qwen3.8-Flash-Next FP8 | SGLang | NEXTN, 3 steps / 4 draft tokens |

One flag deserves a note: `--linear-attn-verify-backend` configures the *verify*
pass of speculative decoding, so it comes out with NEXTN even though its name
does not say "speculative". Leaving it in would have meant a speculation-off
config still carrying speculation machinery.

Throughput is the aggregate output rate — all output tokens over the measured
window, not GuideLLM's per-request mean.

### Acceptance statistics come free

Both engines already log acceptance on every reporting interval, and the
benchmark harness brackets each workload with a UTC timestamp, so every sample
can be attributed to the workload that produced it. No instrumentation was
needed and no extra GPU time was spent — the speculation-**on** numbers were
mined from logs written weeks earlier.

```text
vLLM    SpecDecoding metrics: Mean acceptance length: 3.31,
        Accepted throughput: 3435.81 tokens/s, Drafted throughput: 7422.95 tokens/s,
        Per-position acceptance rate: 0.719, 0.547, 0.423, 0.342, 0.285,
        Avg Draft acceptance rate: 46.3%

SGLang  Decode batch, #running-req: 22, token usage: 0.99,
        accept len: 3.02, accept rate: 0.41, gen throughput (token/s): 1415.06
```

## The result

<figure class="qz">
  <svg viewBox="0 0 640 354" role="img" aria-label="Throughput change from enabling speculative decoding, for six model and mechanism pairs, in three regimes. DeepSeek-V4.1-Flash vLLM DSpark-5: 1 stream, CHAT-S +110 percent, 64 streams, CHAT-S -16 percent, 64 streams, BATCH-D +43 percent; GLM-5.3 SGLang EAGLE-5: 1 stream, CHAT-S +84 percent, 64 streams, CHAT-S -26 percent, 64 streams, BATCH-D +29 percent; GLM-5.3 vLLM MTP-5: 1 stream, CHAT-S +65 percent, 64 streams, CHAT-S -5 percent, 64 streams, BATCH-D +30 percent; GLM-5.3-Flash SGLang EAGLE-5: 1 stream, CHAT-S +55 percent, 64 streams, CHAT-S -30 percent, 64 streams, BATCH-D +1 percent; GLM-5.3-Flash vLLM MTP-5: 1 stream, CHAT-S +13 percent, 64 streams, CHAT-S -33 percent, 64 streams, BATCH-D +15 percent; Qwen3.8-Flash-Next FP8 SGLang NEXTN-3: 1 stream, CHAT-S +35 percent, 64 streams, CHAT-S -36 percent, 64 streams, BATCH-D +2 percent.">
    <rect x="250" y="12" width="13" height="3" rx="1.5" fill="var(--ok)"/>
    <text class="sub" x="268" y="16.5">1 stream, CHAT-S</text>
    <rect x="367" y="12" width="13" height="3" rx="1.5" fill="var(--bad)"/>
    <text class="sub" x="385" y="16.5">64 streams, CHAT-S</text>
    <rect x="495" y="12" width="13" height="3" rx="1.5" fill="var(--vl)"/>
    <text class="sub" x="513" y="16.5">64 streams, BATCH-D</text>
    <line x1="250" y1="32" x2="250" y2="308" stroke="var(--grid)"/>
    <text class="sub" x="250" y="28" text-anchor="middle">-40%</text>
    <line x1="293" y1="32" x2="293" y2="308" stroke="var(--grid)"/>
    <text class="sub" x="293" y="28" text-anchor="middle">-20%</text>
    <line x1="336" y1="32" x2="336" y2="308" stroke-width="1.5" stroke="var(--dim)"/>
    <text class="sub" x="336" y="28" text-anchor="middle">+0%</text>
    <line x1="379" y1="32" x2="379" y2="308" stroke="var(--grid)"/>
    <text class="sub" x="379" y="28" text-anchor="middle">+20%</text>
    <line x1="422" y1="32" x2="422" y2="308" stroke="var(--grid)"/>
    <text class="sub" x="422" y="28" text-anchor="middle">+40%</text>
    <line x1="465" y1="32" x2="465" y2="308" stroke="var(--grid)"/>
    <text class="sub" x="465" y="28" text-anchor="middle">+60%</text>
    <line x1="508" y1="32" x2="508" y2="308" stroke="var(--grid)"/>
    <text class="sub" x="508" y="28" text-anchor="middle">+80%</text>
    <line x1="551" y1="32" x2="551" y2="308" stroke="var(--grid)"/>
    <text class="sub" x="551" y="28" text-anchor="middle">+100%</text>
    <line x1="594" y1="32" x2="594" y2="308" stroke="var(--grid)"/>
    <text class="sub" x="594" y="28" text-anchor="middle">+120%</text>
    <text class="lab" x="236" y="58" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="236" y="70" text-anchor="end">vLLM DSpark-5</text>
    <rect x="336.2" y="46" width="235.5" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="578" y="55" text-anchor="start">+110%</text>
    <rect x="301.4" y="59" width="34.8" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="295" y="68" text-anchor="end">-16%</text>
    <rect x="336.2" y="72" width="92.4" height="11" rx="2" fill="var(--vl)"/>
    <text class="val" x="435" y="81" text-anchor="start">+43%</text>
    <text class="lab" x="236" y="104" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="236" y="116" text-anchor="end">SGLang EAGLE-5</text>
    <rect x="336.2" y="92" width="180.0" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="522" y="101" text-anchor="start">+84%</text>
    <rect x="281.4" y="105" width="54.8" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="275" y="114" text-anchor="end">-26%</text>
    <rect x="336.2" y="118" width="62.9" height="11" rx="2" fill="var(--vl)"/>
    <text class="val" x="405" y="127" text-anchor="start">+29%</text>
    <text class="lab" x="236" y="150" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="236" y="162" text-anchor="end">vLLM MTP-5</text>
    <rect x="336.2" y="138" width="140.3" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="482" y="147" text-anchor="start">+65%</text>
    <rect x="325.0" y="151" width="11.2" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="319" y="160" text-anchor="end">-5%</text>
    <rect x="336.2" y="164" width="64.1" height="11" rx="2" fill="var(--vl)"/>
    <text class="val" x="406" y="173" text-anchor="start">+30%</text>
    <text class="lab" x="236" y="196" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="236" y="208" text-anchor="end">SGLang EAGLE-5</text>
    <rect x="336.2" y="184" width="117.0" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="459" y="193" text-anchor="start">+55%</text>
    <rect x="271.2" y="197" width="65.0" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="265" y="206" text-anchor="end">-30%</text>
    <rect x="336.2" y="210" width="3.2" height="11" rx="2" fill="var(--vl)"/>
    <text class="val" x="345" y="219" text-anchor="start">+1%</text>
    <text class="lab" x="236" y="242" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="236" y="254" text-anchor="end">vLLM MTP-5</text>
    <rect x="336.2" y="230" width="27.5" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="370" y="239" text-anchor="start">+13%</text>
    <rect x="264.8" y="243" width="71.4" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="259" y="252" text-anchor="end">-33%</text>
    <rect x="336.2" y="256" width="32.3" height="11" rx="2" fill="var(--vl)"/>
    <text class="val" x="375" y="265" text-anchor="start">+15%</text>
    <text class="lab" x="236" y="288" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="236" y="300" text-anchor="end">SGLang NEXTN-3</text>
    <rect x="336.2" y="276" width="75.1" height="11" rx="2" fill="var(--ok)"/>
    <text class="val" x="417" y="285" text-anchor="start">+35%</text>
    <rect x="259.2" y="289" width="77.0" height="11" rx="2" fill="var(--bad)"/>
    <text class="val" x="253" y="298" text-anchor="end">-36%</text>
    <rect x="336.2" y="302" width="5.2" height="11" rx="2" fill="var(--vl)"/>
    <text class="val" x="347" y="311" text-anchor="start">+2%</text>
    <text class="ax" x="250" y="346" text-anchor="start">change in output tok/s when speculation is switched on</text>
  </svg>
  <figcaption>
    Change in output throughput when speculation is switched on, everything else
    held constant. Positive is faster with speculation. The middle bar is the
    one that matters for most production traffic.
  </figcaption>
</figure>

At one stream speculation earns its reputation: **+110% on
DeepSeek-V4.1-Flash**, +84% on GLM-5.3 under EAGLE, and +13% to +65% across the
rest — the weakest being GLM-5.3-Flash under MTP. This is the regime every
recipe is tuned for and every benchmark headline is drawn from.

At 64 concurrent streams on the same chat-shaped workload, **all six pairs go
negative** — from −5% on GLM-5.3 under MTP to −36% on Qwen3.8 under NEXTN. Not
"less helpful". Slower than not doing it.

Decode-heavy batch work is the exception: BATCH-D (4,096 in → 8,192 out) stays
positive everywhere, from +1.5% to +43%.

## Why it inverts

The obvious hypothesis is that drafts stop being accepted once the batch is
busy — that the drafter gets worse. That is measurable, and it is wrong.

<figure class="qz">
  <svg viewBox="0 0 640 300" role="img" aria-label="Mean accepted tokens per step at 1 stream versus 64 streams. DeepSeek-V4.1 vLLM DSpark-5: 2.86 then 2.88; GLM-5.3 SGL EAGLE-5: 2.81 then 2.77; GLM-5.3 vLLM MTP-5: 2.74 then 2.69; GLM-5.3-Flash SGL EAGLE-5: 2.68 then 2.68; GLM-5.3-Flash vLLM MTP-5: 2.67 then 2.63; Qwen3.8 FP8 SGL NEXTN-3: 2.07 then 2.17.">
    <line x1="104" y1="232" x2="388" y2="232" stroke="var(--grid)"/>
    <text class="sub" x="94" y="235" text-anchor="end">2.0</text>
    <line x1="104" y1="151" x2="388" y2="151" stroke="var(--grid)"/>
    <text class="sub" x="94" y="154" text-anchor="end">2.5</text>
    <line x1="104" y1="70" x2="388" y2="70" stroke="var(--grid)"/>
    <text class="sub" x="94" y="74" text-anchor="end">3.0</text>
    <text class="sub" x="104" y="38" text-anchor="middle">1 stream</text>
    <text class="sub" x="388" y="38" text-anchor="middle">64 streams</text>
    <line x1="104" y1="92" x2="388" y2="90" stroke="var(--vl)" stroke-width="2"/>
    <circle cx="104" cy="92" r="3.5" fill="var(--vl)"/>
    <circle cx="388" cy="90" r="3.5" fill="var(--vl)"/>
    <line x1="104" y1="100" x2="388" y2="107" stroke="var(--vl)" stroke-width="2"/>
    <circle cx="104" cy="100" r="3.5" fill="var(--vl)"/>
    <circle cx="388" cy="107" r="3.5" fill="var(--vl)"/>
    <line x1="104" y1="112" x2="388" y2="121" stroke="var(--vl)" stroke-width="2"/>
    <circle cx="104" cy="112" r="3.5" fill="var(--vl)"/>
    <circle cx="388" cy="121" r="3.5" fill="var(--vl)"/>
    <line x1="104" y1="123" x2="388" y2="121" stroke="var(--vl)" stroke-width="2"/>
    <circle cx="104" cy="123" r="3.5" fill="var(--vl)"/>
    <circle cx="388" cy="121" r="3.5" fill="var(--vl)"/>
    <line x1="104" y1="123" x2="388" y2="130" stroke="var(--vl)" stroke-width="2"/>
    <circle cx="104" cy="123" r="3.5" fill="var(--vl)"/>
    <circle cx="388" cy="130" r="3.5" fill="var(--vl)"/>
    <line x1="104" y1="220" x2="388" y2="205" stroke="var(--vl)" stroke-width="2"/>
    <circle cx="104" cy="220" r="3.5" fill="var(--vl)"/>
    <circle cx="388" cy="205" r="3.5" fill="var(--vl)"/>
    <text class="sub" x="397" y="94">DeepSeek-V4.1 · vLLM DSpark-5</text>
    <text class="sub" x="397" y="110">GLM-5.3 · SGL EAGLE-5</text>
    <text class="sub" x="397" y="124">GLM-5.3 · vLLM MTP-5</text>
    <text class="sub" x="397" y="137">GLM-5.3-Flash · SGL EAGLE-5</text>
    <text class="sub" x="397" y="150">GLM-5.3-Flash · vLLM MTP-5</text>
    <text class="sub" x="397" y="208">Qwen3.8 FP8 · SGL NEXTN-3</text>
    <text class="ax" x="104" y="290" text-anchor="start">mean accepted tokens per speculative step</text>
  </svg>
  <figcaption>
    Mean accepted tokens per speculative step, one stream versus 64. The lines
    are flat: draft quality is essentially unchanged by concurrency, while the
    throughput effect swings from +110% to −36%.
  </figcaption>
</figure>

Acceptance moves by at most 0.10 tokens per step between the two regimes. The
drafter is just as good at 64 streams as at one. What changes is what the GPU
was doing with the time.

At one stream the GPU is latency-bound: it finishes a decode step and waits for
the next token to be sampled before it can start the next. Speculation fills
that idle time, and anything the drafter gets right is free.

At 64 streams the batch already saturates the GPU. There is no idle time to
fill, so the draft and verify passes are not free any more — they consume
compute that would otherwise be serving real requests. A mechanism that
accepts 2.7 tokens per step but costs more than 2.7 tokens' worth of extra
compute is a loss, and that is what the middle bars are showing.

### Hold concurrency fixed and acceptance matters again

Once concurrency is not the variable, acceptance is exactly what separates the
workloads. BATCH-D generates 8,192 tokens per request against CHAT-S's 512, and
every mechanism accepts more per step on it.

<figure class="qz">
  <svg viewBox="0 0 640 330" role="img" aria-label="Acceptance uplift from CHAT-S to BATCH-D against the difference in throughput gain, six pairs, correlation r = 0.75. DeepSeek-V4.1-Flash vLLM DSpark-5: +0.69 tokens, +59 points; GLM-5.3 SGLang EAGLE-5: +0.12 tokens, +55 points; GLM-5.3 vLLM MTP-5: +0.07 tokens, +35 points; GLM-5.3-Flash SGLang EAGLE-5: -0.02 tokens, +32 points; GLM-5.3-Flash vLLM MTP-5: +0.39 tokens, +48 points; Qwen3.8-Flash-Next FP8 SGLang NEXTN-3: +0.24 tokens, +38 points.">
    <line x1="115" y1="40" x2="115" y2="274" stroke="var(--grid)"/>
    <text class="sub" x="115" y="290" text-anchor="middle">+0.00</text>
    <line x1="216" y1="40" x2="216" y2="274" stroke="var(--grid)"/>
    <text class="sub" x="216" y="290" text-anchor="middle">+0.25</text>
    <line x1="317" y1="40" x2="317" y2="274" stroke="var(--grid)"/>
    <text class="sub" x="317" y="290" text-anchor="middle">+0.50</text>
    <line x1="418" y1="40" x2="418" y2="274" stroke="var(--grid)"/>
    <text class="sub" x="418" y="290" text-anchor="middle">+0.75</text>
    <line x1="66" y1="249" x2="430" y2="249" stroke="var(--grid)"/>
    <text class="sub" x="57" y="253" text-anchor="end">30pp</text>
    <line x1="66" y1="188" x2="430" y2="188" stroke="var(--grid)"/>
    <text class="sub" x="57" y="191" text-anchor="end">40pp</text>
    <line x1="66" y1="126" x2="430" y2="126" stroke="var(--grid)"/>
    <text class="sub" x="57" y="130" text-anchor="end">50pp</text>
    <line x1="66" y1="65" x2="430" y2="65" stroke="var(--grid)"/>
    <text class="sub" x="57" y="68" text-anchor="end">60pp</text>
    <line x1="66" y1="233" x2="430" y2="53" stroke="var(--dim)" stroke-width="1.5" stroke-dasharray="4 3"/>
    <circle cx="394" cy="69" r="5" fill="var(--ok)"/>
    <text class="sub" x="404" y="73">DeepSeek-V4.1-Flash · vLLM DSpark-5</text>
    <circle cx="164" cy="96" r="5" fill="var(--ok)"/>
    <text class="sub" x="174" y="100">GLM-5.3 · SGLang EAGLE-5</text>
    <circle cx="144" cy="218" r="5" fill="var(--ok)"/>
    <text class="sub" x="154" y="221">GLM-5.3 · vLLM MTP-5</text>
    <circle cx="105" cy="238" r="5" fill="var(--ok)"/>
    <text class="sub" x="115" y="242">GLM-5.3-Flash · SGLang EAGLE-5</text>
    <circle cx="273" cy="136" r="5" fill="var(--ok)"/>
    <text class="sub" x="283" y="140">GLM-5.3-Flash · vLLM MTP-5</text>
    <circle cx="211" cy="198" r="5" fill="var(--ok)"/>
    <text class="sub" x="221" y="202">Qwen3.8-Flash-Next FP8 · SGLang NEXTN-3</text>
    <text class="ax" x="66" y="320" text-anchor="start">acceptance uplift, CHAT-S → BATCH-D (tokens/step) · r = 0.75</text>
  </svg>
  <figcaption>
    For each pair, the acceptance gained going from CHAT-S to BATCH-D against
    the throughput advantage gained on the same move, both at 64 concurrent
    requests. Six independent model-and-mechanism pairs, r = 0.75.
  </figcaption>
</figure>

DSpark picks up the most acceptance on BATCH-D (+0.69 tokens/step) and the most
throughput (+59 points). EAGLE on GLM-5.3-Flash picks up none (−0.02) and the
least (+32 points). With six points this is a suggestive fit rather than a
fitted law, but the direction is consistent and the mechanism is the obvious
one.

So the answer to *"is it acceptance or is it the longer decode phase?"* is
**both, on different axes**. Across concurrency it is saturation and acceptance
plays no part. Across workload at fixed concurrency it is acceptance.

## Where the draft length ceiling comes from

Every one of these recipes drafts 4–7 tokens. The per-position acceptance rates
show why nobody drafts 20.

<figure class="qz">
  <svg viewBox="0 0 640 320" role="img" aria-label="Acceptance rate by draft position on BATCH-D at 64 concurrent requests.">
    <line x1="64" y1="211" x2="432" y2="211" stroke="var(--grid)"/>
    <text class="sub" x="55" y="214" text-anchor="end">25%</text>
    <line x1="64" y1="156" x2="432" y2="156" stroke="var(--grid)"/>
    <text class="sub" x="55" y="160" text-anchor="end">50%</text>
    <line x1="64" y1="101" x2="432" y2="101" stroke="var(--grid)"/>
    <text class="sub" x="55" y="104" text-anchor="end">75%</text>
    <line x1="64" y1="46" x2="432" y2="46" stroke="var(--grid)"/>
    <text class="sub" x="55" y="50" text-anchor="end">100%</text>
    <text class="sub" x="64" y="284" text-anchor="middle">1</text>
    <text class="sub" x="156" y="284" text-anchor="middle">2</text>
    <text class="sub" x="248" y="284" text-anchor="middle">3</text>
    <text class="sub" x="340" y="284" text-anchor="middle">4</text>
    <text class="sub" x="432" y="284" text-anchor="middle">5</text>
    <polyline points="64,74 156,126 248,170 340,191 432,205" fill="none" stroke="var(--ok)" stroke-width="2" stroke-linejoin="round"/>
    <circle cx="64" cy="74" r="3.5" fill="var(--ok)"/>
    <circle cx="156" cy="126" r="3.5" fill="var(--ok)"/>
    <circle cx="248" cy="170" r="3.5" fill="var(--ok)"/>
    <circle cx="340" cy="191" r="3.5" fill="var(--ok)"/>
    <circle cx="432" cy="205" r="3.5" fill="var(--ok)"/>
    <text class="sub" x="442" y="208">DeepSeek-V4.1 · DSpark</text>
    <polyline points="64,113 156,158 248,187 340,207 432,220" fill="none" stroke="var(--bad)" stroke-width="2" stroke-linejoin="round"/>
    <circle cx="64" cy="113" r="3.5" fill="var(--bad)"/>
    <circle cx="156" cy="158" r="3.5" fill="var(--bad)"/>
    <circle cx="248" cy="187" r="3.5" fill="var(--bad)"/>
    <circle cx="340" cy="207" r="3.5" fill="var(--bad)"/>
    <circle cx="432" cy="220" r="3.5" fill="var(--bad)"/>
    <text class="sub" x="442" y="223">GLM-5.3-Flash · MTP</text>
    <polyline points="64,118 156,172 248,200 340,220 432,233" fill="none" stroke="var(--vl)" stroke-width="2" stroke-linejoin="round"/>
    <circle cx="64" cy="118" r="3.5" fill="var(--vl)"/>
    <circle cx="156" cy="172" r="3.5" fill="var(--vl)"/>
    <circle cx="248" cy="200" r="3.5" fill="var(--vl)"/>
    <circle cx="340" cy="220" r="3.5" fill="var(--vl)"/>
    <circle cx="432" cy="233" r="3.5" fill="var(--vl)"/>
    <text class="sub" x="442" y="236">GLM-5.3 · MTP</text>
    <text class="ax" x="64" y="310" text-anchor="start">draft position · BATCH-D · 64 concurrent requests</text>
  </svg>
  <figcaption>
    Probability that the draft token at each position is accepted, on BATCH-D at
    64 concurrent requests. vLLM reports this directly; SGLang does not.
  </figcaption>
</figure>

DSpark's first draft token is accepted 87.5% of the time and its fifth 27.8%.
GLM-5.3's MTP decays faster — 67.3% to 15.2%. Each extra position costs a full
verify slot for a rapidly shrinking chance of being right, so the curve sets
the useful draft length, and the recipes have all landed in the same 4–7 band.

It also explains the ordering in the first chart. DSpark's decay is the
shallowest of the three, which is why DeepSeek-V4.1-Flash gets the biggest
single-stream gain in the set.

## What to actually do

- **Serving one request at a time** — interactive coding, a local assistant, a
  latency-sensitive endpoint with light traffic: leave speculation on. It is the
  single largest lever available, worth up to a doubling.
- **Serving loaded chat traffic** — dozens of concurrent conversations: turn it
  off, and take back 5–36%. Every published recipe has it on; this is the one
  place they are wrong for the job.
- **Serving decode-heavy batch work** — long generations, offline pipelines:
  leave it on. Up to +43%.
- **In between**: measure. The crossover is somewhere between 1 and 64 streams
  and this campaign did not locate it. It will differ by model.

The broader point is that speculative decoding is a *latency* optimisation that
the ecosystem markets as a throughput one. It buys idle GPU time back. If your
GPU has no idle time, there is nothing for it to buy.

## Caveats

- **One loaded concurrency point.** Everything above at 64 streams is a single
  point; the crossover is not located.
- **CHAT-S runs warm.** The chat workload sits at a 59.5% prefix-cache hit rate
  because a 2,048-token workload runs before it with the same synthetic prompts,
  so its prefill is unusually cheap. That makes it an unusually clean decode
  test, which suits this question, but the absolute numbers are not cold-cache.
- **Qwen has no published vLLM speculative cell**, so NEXTN is measured on
  SGLang only.
- **Per-position rates are vLLM-only.** SGLang reports mean acceptance length
  but not the position breakdown.
- Results are specific to 8×H200 (SM90). On Blackwell, where several of these
  models get hardware paths they do not have here, the balance may differ.
