+++
title = "Search, Don't Generate: +16% on a Tesla P40 by Changing One Number"
description = "The fashionable move is asking an LLM to write CUDA kernels. On a 2016 GPU, the right move was letting agents run a disciplined discrete search — and the winning diff is one field wide."
date = 2026-08-20
[extra]
lead = "What a day of agent-orchestrated autotuning found in llama.cpp's Pascal config table — and why the method matters more than the number."
+++

There's a Tesla P40 in my basement. It went into my Dell R730 for Jellyfin
transcoding, but it's a 24 GB CUDA card and it seemed a shame not to make it
earn more of its keep. So: local LLM inference, llama.cpp, and a question —
how much performance is actually on that table?

The fashionable answer in 2026 is to point a frontier model at the CUDA and ask
it to write faster kernels. For this card, that's the wrong tool — and knowing
*why* it's wrong shaped everything that follows.

## Why not kernel generation

The P40 is sm_61 — Pascal, 2016. Modern kernel-gen tooling mostly can't even
target it: Triton's minimum compute capability is 8.0, so Pascal isn't a
degraded target, it's *not a target at all*. And the published results in the
LLM-kernel-generation literature have a dominant, well-documented failure mode:
fast-but-wrong kernels that sail through weak test harnesses. (Foreshadowing.)

There's also a physics argument. On sm_61 the fast path for quantized matmul is
`__dp4a` — a 4-way int8 dot product — and the ceiling is the integer issue
pipe. Every 4-bit unpack instruction displaces a `dp4a` one-for-one. There is
no clever tile layout that escapes this; the published llama.cpp baseline on
this card already sits around 30% of the dp4a roofline, and the realistic prize
for *any* method is measured in tens of percent, not multiples.

But llama.cpp's MMQ (quantized matmul) kernels are *configurable*. Each GPU
architecture gets a config table — `mmq-config-pascal.cuh` — with 242 rows,
each row a tuple of launch parameters for one (quant type, tile width) case:
threads per block, target occupancy, tile height `I`, and a work-decomposition
flag `stream_k`. And here's the thing: **every row of the Pascal table held the
same inherited tuple** — `(256, 2, 64, false)` — while Ampere's table
disagrees with it in exactly two fields (`I=128`, `stream_k=true`). Nobody had
published evidence that anyone ever tried the alternatives on Pascal.

That's not a code-generation problem. That's a **discrete search problem** —
five parameters, hard constraints, a cheap analytic filter, and an expensive
oracle (build + test + benchmark). Search does the optimizing. The interesting
question is what role an LLM plays, and my answer was: *orchestration*. Claude
agents ran the runbook, built the harness, interpreted the profiler's
complaints, and kept the measurement discipline honest. No model wrote a line
of CUDA.

## Gate B: a falsifiable bet

Before committing months to a general autotuner, I wanted one day and one
number. The runbook — written in advance — called it Gate B:

> Does *any* parameter combination beat the inherited tuple by more than 5% on
> a hot shape? If no, the constant is accidentally near-optimal, the project
> has no thesis, stop and say so. Do not soften the result.

Pre-registered kill criteria: under +2%, write up the negative result and quit.
+2–5%, marginal, extend before committing. Over +5%, go. The deliverable is
identical either way: the results table, the telemetry proving clocks held,
paired correctness for every number, and one sentence of verdict.

## Measurement discipline is the whole game

A 5% effect on a consumer-grade rig disappears into noise unless you're
paranoid, so the runbook's hygiene rules were non-negotiable. Some of them we
learned the hard way, the same day:

**The card is power-capped, not clock-capped.** Under sustained load a P40
pulls its full 248 W and settles wherever the power governor puts it
(1430–1468 MHz on mine). Locking clocks *above* that band gives you fake
determinism — the cap still governs. You lock *below* it. Except…

**`nvidia-smi -lgc` doesn't work on Pascal.** It exits 0 and does nothing —
the locked-clocks API is Volta+. The agent running environment setup caught
this because it read the clock back under load instead of trusting the exit
status, and fell back to the legacy application-clocks API
(`nvidia-smi -ac 3615,1404`). Every valid run that day held 1404 MHz exactly,
and every run's telemetry window was checked; any clock deviation invalidated
the run.

**Never build while measuring.** A 48-thread compile heats the chassis the GPU
breathes from. The sweep ran strictly serial: patch → build → settle until CPU
package temps returned to idle → correctness → benchmark → restore. It costs
throughput. It's why the numbers are comparable.

**Control for drift.** The stock config was re-measured at the start and end of
the sweep day: 1009.52 ± 0.42 vs 1010.41 ± 2.26 t/s, 5.1 hours apart. Flat.

**And control for the household.** The P40 also serves Jellyfin (an LXC on the
same host shares it for NVENC). On a power-capped card, someone's transcode
steals power budget and silently sags your clocks. The harness stops the
Jellyfin container for every benchmark window and restarts it after — even on
a crash. My family's TV access was a confounding variable and it got the same
treatment as the others.

The reproduced baseline landed at **1009.52 t/s** prefill (pp512) on Llama-2-7B
Q4_0 — within 0.2% of the published scoreboard figure for this exact card.
That's when you know the environment is sound and any delta you find is real.

## The oracle would have lied to us

Rule three of the runbook: never report a speedup without a paired correctness
result. So before sweeping, we pointed the correctness suite at the GPU:

```
$ test-backend-ops -b CUDA -o MUL_MAT
2/2 backends passed  ✓
```

Except the GPU backend in this build registers as `CUDA0`, not `CUDA`. The flag
matched nothing, the suite silently skipped everything, and reported a **vacuous
pass with zero GPU work**. The agent caught it because the telemetry showed 0%
GPU utilization during a supposed load test — the "passing" run never touched
the card.

It got better. The one large-shape test case in the upstream suite — the only
case that would exercise the wide-tile kernel variant (J=64) that real prefill
traffic actually uses — turned out to be **dead code behind `#if 0`**. Upstream's
correctness suite has never tested the J=64 path for *any* quant type. We added
an active 8192×512×5120 Q4_0 case, and later in the day the sweep demonstrated
exactly why this matters: **24 of 72 configurations passed the full stock
correctness suite and then crashed with CUDA launch errors the moment real
pp512 traffic hit them.** If our harness had trusted the stock oracle — like
most published kernel-generation harnesses trust theirs — those configs would
have been reportable "results."

This is the fast-but-wrong failure mode of the kernel-gen literature,
reproduced in miniature, in an afternoon, in the *reference* test suite of the
most popular inference engine in the world.

## The sweep

The harness (Python, stdlib, ~1300 lines, written by an implementer agent and
put through a real code review by a reviewer agent — which caught a bug where
one malformed benchmark response would have aborted the entire unattended sweep)
ran 72 candidates over about four hours: four thread counts × three occupancy
targets × three tile heights × stream-k on/off. Each candidate: patch one row,
70-second incremental rebuild, thermal settle, 1,187-case correctness gate,
10-rep benchmark, telemetry check, restore.

<figure>
<svg viewBox="0 0 680 420" role="img" aria-label="Horizontal bar chart of prefill throughput for ten representative MMQ configurations. The winner, 256 threads occupancy 2 tile 128, reaches 1174 tokens per second, 16 percent above the stock 1009. The worst configuration collapses to 44.">
  <title>pp512 throughput by MMQ config (Llama-2-7B Q4_0, Tesla P40, clocks locked at 1404 MHz)</title>
  <style>
  .cfg { font: 500 12px var(--font-mono, monospace); fill: var(--muted, #6b7280); }
  .val { font: 600 12px var(--font-sans, sans-serif); fill: var(--ink, #14161c); }
  .note { font: 400 11px var(--font-sans, sans-serif); fill: var(--muted, #6b7280); }
  </style>
  <!-- baseline rule at stock 1009.5 t/s -->
  <line x1="570" y1="28" x2="570" y2="390" stroke="var(--muted, #6b7280)" stroke-width="1" stroke-dasharray="4 4" opacity="0.6"/>
  <text class="note" x="570" y="18" text-anchor="middle">stock · 1009.5</text>
  <!-- bars: x=200 origin, 0.36667 px per t/s -->
  <g>
  <text class="cfg" x="192" y="53" text-anchor="end">256,2,128</text>
  <rect x="200" y="40" width="430.6" height="18" rx="3" fill="var(--accent, #6d4aff)"/>
  <text class="val" x="637" y="53">1174.3 ← winner</text>
  <text class="cfg" x="192" y="85" text-anchor="end">256,1,128</text>
  <rect x="200" y="72" width="430.5" height="18" rx="3" fill="var(--accent, #6d4aff)" opacity="0.55"/>
  <text class="val" x="637" y="85">1174.2</text>
  <text class="cfg" x="192" y="117" text-anchor="end">128,1,64</text>
  <rect x="200" y="104" width="409.5" height="18" rx="3" fill="var(--accent, #6d4aff)" opacity="0.55"/>
  <text class="val" x="616" y="117">1116.8</text>
  <text class="cfg" x="192" y="149" text-anchor="end">128,2,64,sk</text>
  <rect x="200" y="136" width="408.5" height="18" rx="3" fill="var(--accent, #6d4aff)" opacity="0.55"/>
  <text class="val" x="615" y="149">1114.1</text>
  <text class="cfg" x="192" y="181" text-anchor="end">256,1,64</text>
  <rect x="200" y="168" width="377.3" height="18" rx="3" fill="var(--accent, #6d4aff)" opacity="0.55"/>
  <text class="val" x="584" y="181">1029.0</text>
  <text class="cfg" x="192" y="213" text-anchor="end">256,2,64 (stock)</text>
  <rect x="200" y="200" width="370.1" height="18" rx="3" fill="var(--muted, #6b7280)" opacity="0.7"/>
  <text class="val" x="577" y="213">1009.5</text>
  <text class="cfg" x="192" y="245" text-anchor="end">512,2,64</text>
  <rect x="200" y="232" width="338.6" height="18" rx="3" fill="var(--accent, #6d4aff)" opacity="0.55"/>
  <text class="val" x="545" y="245">923.4</text>
  <text class="cfg" x="192" y="277" text-anchor="end">128,1,128</text>
  <rect x="200" y="264" width="333.8" height="18" rx="3" fill="var(--accent, #6d4aff)" opacity="0.55"/>
  <text class="val" x="540" y="277">910.4</text>
  <text class="cfg" x="192" y="309" text-anchor="end">256,4,128</text>
  <rect x="200" y="296" width="157.0" height="18" rx="3" fill="var(--accent, #6d4aff)" opacity="0.55"/>
  <text class="val" x="363" y="309">428.2</text>
  <text class="cfg" x="192" y="341" text-anchor="end">512,4,128</text>
  <rect x="200" y="328" width="16.0" height="18" rx="3" fill="var(--accent, #6d4aff)" opacity="0.55"/>
  <text class="val" x="222" y="341">43.6</text>
  </g>
  <text class="note" x="200" y="380">pp512, tokens/s → (config: nthreads, occupancy, I; sk = stream_k on)</text>
</svg>
<figcaption>Ten of the 46 configurations that survived both gates. The full
72-row dataset, with paired correctness results and per-run telemetry, is in
the repo.</figcaption>
</figure>

The surface has structure, and it has cliffs:

| Finding | Evidence |
|---|---|
| Ampere's tile height wins on Pascal | `I=128` at ≤2 occupancy: **+16.3%** |
| Halving threads also wins | the `128,*,64` cluster: ~+10% |
| Occupancy is mostly dead weight | smem caps residency anyway; forcing it spills registers (428 → 43.6 t/s floor) |
| stream_k never helps here | neutral at best; at `(256,2,64)` it's *incorrect*, not slow |
| 24 configs can't launch at all | pass the J=8 oracle, crash on real J=64 traffic |

Both halves of the original hypothesis got their answer the same afternoon:
Ampere's `I=128` was the single biggest lever in the table, and Ampere's
`stream_k` failed correctness outright on Pascal. One field, not two.

## Verification, because the sweep number doesn't count

The winner got the full battery on a from-scratch build: the 1,188-case
correctness suite including our new J=64 case; `compute-sanitizer racecheck`
over the whole matmul suite (clean on the tuned path — the 16 hazards it did
report are a pre-existing race in an fp32 fallback kernel our change can't
reach); a 20-rep benchmark at locked clocks; and perplexity over wikitext-2,
which came out **5.9685 ± 0.0335 — identical to the stock build, to every
printed digit**. A config that changes perplexity is wrong, not fast.

Final: **1174.21 ± 1.61 t/s vs 1009.52 ± 0.42 — +16.31%**, decode unregressed.
The diff:

```diff
-    CASE(GGML_TYPE_Q4_0, 256, 2, 64,  64, ..., false, false);
+    CASE(GGML_TYPE_Q4_0, 256, 2, 128, 64, ..., false, false);
```

That moves this card from ~30% to ~35% of its dp4a roofline. It's a 1.16×
single-row result against the project's 1.2–1.5× full-table expectation — the
gate asked whether the table moves at all, and the answer is emphatically yes:
241 rows to go.

## What the agents actually did

I want to be precise about the division of labor, because "AI did it" and "AI
is hype" are both wrong here.

**The search did the optimizing.** No model proposed `I=128` from intuition;
it was one of 72 grid points, and its win was discovered by benchmark, not
predicted.

**The agents did the orchestration** — and it was real engineering. A fresh
subagent per runbook step, each with a written brief; a reviewer agent gating
each step; a fix loop when review found a real bug (it did — the
sweep-aborting JSON parse); a ledger of every judgment call with its
justification. The catches that saved the day's integrity — the vacuous
correctness pass, the `-lgc` no-op, the dead `#if 0` test case — were all
agents refusing to trust an exit code and checking the physical evidence
instead: GPU utilization, clock readback, whether the card actually got warm.

**The human decided.** Whether Jellyfin could be stopped during measurement
windows. Whether the racecheck hazards on an unrelated kernel blocked the
verdict. What ships publicly. And — per llama.cpp's own contributor policy,
which flatly forbids autonomous-agent PRs — the one-line diff goes upstream
by my hands or not at all. The agents never touch `git push` on someone else's
project. That rule was in the runbook from line one.

## The dataset is the point

A +16% one-liner is a nice souvenir, but the artifact I actually care about is
the dataset: a complete 72-point sweep of a real kernel-config space on sm_61,
every point with paired correctness results, locked-clock telemetry, build
times, and failure modes — including the 24 pass-the-tests-crash-in-production
configs and two that are silently *numerically wrong*. The kernel-tuning
literature almost never publishes the full map, and never for hardware this
old. It's all in the repo, CSV and gzipped telemetry included:

**[github.com/cycle-five/pascal-mmq-autorunner](https://github.com/cycle-five/pascal-mmq-autorunner)**

Next: Tier 1 — the general autotuner across all 242 rows and the other quant
types, with the J-coverage correctness cases it turns out everyone needed. The
P40 has 241 more numbers to give up, and a basement to heat while it does.
