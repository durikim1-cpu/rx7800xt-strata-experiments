# How the same PC started answering a 60K-token prompt in 41 s instead of 137 s

A personal experiment with [Strata](https://github.com/Niko1221/Strata) and Qwen3.8-Flash-Next on one AMD gaming PC,
written for people who are new to local LLMs. Measured 2026-10-08 to 2026-10-18. Nothing here is an official Strata
result; one machine, small test sets, and every number is given with the condition it was measured under.

**Short version.** Without buying hardware or changing the model, changing *how the engine reads and moves data* cut
the wait for the first token of a fresh 60,047-token prompt from **137.5 s to 40.7 s** (about −70%). Prompt reading went
from 437 to 1,476 tokens/s. Writing the answer (decode) went from 29.9 to 36.0 tokens/s, which is **not** above the
41.6 tokens/s an early measurement recorded under different settings. Faster is not smarter: accuracy was not the goal and was not compared like for like, a long-context instruction failure is still there, and MTP stalls in continuous runs are still open (see "What is still broken").

## 1. Three speeds, not one

A local LLM answers in three steps, and each has its own speed:

| Step | Name | What it means here | Better |
| --- | --- | --- | --- |
| reading the prompt | prefill, "prompt tok/s" | how many prompt tokens per second the engine processes | higher |
| waiting for the first word | TTFT, seconds | from pressing send to the first generated token; for a long prompt this is mostly reading | lower |
| writing the answer | decode, "decode tok/s" | generated tokens per second | higher |

Reading a book twice as fast does not make you talk twice as fast. In this report 1,476 tokens/s is *reading*;
36.0 tokens/s is *writing*. A "token" is a piece of text, not a character or a word; "60K input" is a name and the
actual prompt is 60,047 tokens. A 64K **context** is the size of the desk (65,536 tokens of room for prompt, history and
answer), not the size of every prompt.

Other terms used below:

- **Quantization (IQ3_XXS, IQ3_S):** smaller storage formats for the model weights. The letter S does not mean better.
- **int8 KV:** the format of the conversation memory (KV cache), not of the model.
- **MTP:** a draft of the next tokens that the main model checks; it can speed up writing. It is not "thinking mode".
- **MoE / experts:** the model picks a few "experts" (blocks of weights) per token. Flash-Next has 48 layers × 512
  experts; the expert file alone is 42,912,972,800 bytes (about 39.97 GiB), so it cannot fit in 16 GB of VRAM. The
  engine keeps some experts in VRAM, more in RAM, and reads the rest from the SSD.

## 2. The machine and the model

| Part | This PC |
| --- | --- |
| GPU | AMD Radeon RX 7800 XT, 16 GB VRAM, gfx1101 (RDNA3). No NVIDIA GPU. HIP backend. |
| CPU | AMD Ryzen 5 9600X, 6 cores / 12 threads |
| RAM | 64 GB DDR5 (observed DDR5-5600); Windows reports about 61.6 GiB usable |
| Storage | SK hynix Gold P31 1 TB NVMe |
| OS | **Windows 11** (native, no WSL2), ASRock B650M Pro RS |

Model: Qwen3.8-Flash-Next GSQ-RCO IQ3_XXS (native pack), and Swift 1.5 IQ3_XXS for some comparisons. Thinking off,
MTP on, temperature 0. Speed prompts are a synthetic English ledger with a summary request, output cap 384 tokens,
three runs per setting (the middle value is reported), and every run is a **fresh** prompt (0 reused tokens) unless
stated.

### Where the targets came from

- **Prompt 1,500 tokens/s:** a user report in Strata issue #1304 of about 1,400–1,500 prompt tokens/s on one
  RTX 5060 Ti. Its model, quantization, prompt length and repetitions are not stated, so it is a reference point, not
  a like-for-like comparison. NVIDIA reaching 1,500 does not prove that this AMD card must.
- **Decode 40–50 tokens/s:** my own stretch goal.
- For context, a community report on an RTX 5070 Ti (Windows 10, Ryzen 9 5900X, IQ3_XXS, 0.1.39) measured
  1,653 / 2,482 prompt tokens/s and 72.5 / 73.7 decode at 4K / 32K prompts. Different GPU, CPU, memory and prompt.

## 3. Starting point (official releases)

First comparison at a 32K context (decode tokens/s, MTP off → on, medians of 3). The four rows also differ in KV type
and memory settings, so this is not a pure model-architecture comparison.

| Model | ~1K prompt | ~8K prompt | ~24K prompt | 12-question set |
| --- | --- | --- | --- | --- |
| Qwen3.8-27B IQ4_XS + Q8 KV | 30.3 → 45.3 | 29.5 → 43.3 | 27.9 → 38.2 | 10/12 |
| Flash-Next IQ3_XXS | 35.6 → 39.7 | 34.1 → 40.2 | 34.0 → 37.8 | 11/12 |
| Flash-Next IQ3_S | 27.7 → 32.1 | 27.2 → 32.6 | 25.9 → 32.5 | 11/12 |
| Swift 1.5 IQ3_XXS | 34.0 → 40.9 | 31.7 → 42.9 | 32.6 → 39.7 | 11/12 |

At a 64K context with a 60,047-token prompt, Flash-Next IQ3_XXS still wrote at 41.6 tokens/s but needed 131.5 s before
the first token. That wait became the main target. Measured again just before round 1 on Strata 0.1.41 as released:
437.0 prompt tokens/s, 29.9 decode, **137.48 s** TTFT. The 41.6 and the later 29.9 are different measurement points and
both are kept.

What the 12 questions missed: all four models failed the same arithmetic question (12,000 won × 3 with 15% off plus
2,500 won shipping; correct 33,100, answers 36,100 / 38,300 / 38,500 / 38,500); the 27B also wrapped a correct order in
an object instead of returning a bare JSON array.

## 4. Rounds 1–6: what changed and where it got to

Same prompt (60,047 tokens, 0 reused), 64K context, medians of 3 ([`results/history-60k.csv`](results/history-60k.csv)).
This is a **timeline**, not a one-change-at-a-time table: engine, KV type, RAM budget and time differ between rows.

| Step | Prompt tok/s | Decode tok/s | TTFT s | Expert RAM GiB | What changed |
| --- | ---: | ---: | ---: | ---: | --- |
| before round 1 | 437.0 | 29.9 | 137.48 | 24 | 0.1.41 release, FP16 KV |
| round 1 | 915.4 | 36.9 | 65.67 | 28 | new engine, gfx1101 hipBLASLt table, WMMA attention, int8 KV, … |
| round 2 | 1,031.1 | 33.1 | 58.32 | 26 | compact MMQ scratch, selected kernels |
| round 3 layer-major | 1,177.4 | 27.0 | 51.10 | 20 | layer-major prefill (see credits) |
| round 3 shared arena | 1,408.9 | 27.5 | 42.70 | 23 | layer-major staging borrowed from the expert cache |
| round 5 fetch + IO | 1,413.4 | 33.0 | 42.56 | 23 | persistent expert fetch, CPU/IO overlap |
| round 6 adopted | **1,476.4** | **36.0** | **40.74** | 19 | Windows file cache for expert reads |

Prompt reading: 437.0 → 1,476.4, about 3.38x (about 3.23x if the first 64K measurement, 457.0, is the start).
"3x faster" without saying which is the start misleads. The final 1,476.4 is about 1.6% short of 1,500, and 36.0 is
short of 40–50.

### Round 1: the AMD way to do the math

- **A hipBLASLt tuning table for gfx1101** (none ships for this card): which matrix-multiply kernel is fastest for each
  matrix shape on this GPU. 70 rows; per multiply 0.81x to 15.9x plain hipBLAS (median 6.45x). On the unmodified
  0.1.44.1 release with only the table switched on (Swift, 24K): prompt **404.6 → 695.2** tokens/s (+71.8%), TTFT
  59.51 → 34.67 s, decode 29.9 → 29.2, 11/12 in both. *Reading got faster; writing did not.*
  Details: [`results/gfx1101-hipblaslt-table/`](results/gfx1101-hipblaslt-table/).
- WMMA attention, int8 KV, MTP side stream, expert-cache adaptation every 16 tokens, a larger prompt batch.
- Separately, RAM 24 → 32 GiB raised Swift decode from 29.9 to 38.6, but cut the Windows commit margin, so it was not
  adopted. More RAM for experts is not automatically better.
- Rejected: **Dense MMQ** (prompt +3.9% at 24K, but decode 38.2 → 35.1 and the 12-question score 11/12 → 10/12, a
  latest-record function returned ids instead of records).

### Round 2: less "seat stealing" in VRAM

A big temporary buffer during prompt reading pushes experts out of VRAM, and they must be fetched again later. A compact
MMQ scratch buffer shrank one buffer from about 600 to 120 MiB; borrowed expert-cache space fell from 3.37 to 2.90 GiB
(about 283 more expert slots kept). Same-campaign A/B (RAM 26): prompt 910.1 → 1,031.1, TTFT 66.06 → 58.32 s, decode
33.8 → 33.1.

### Round 3: read each layer once for many chunks (layer-major)

Before: for every 512-token chunk the engine walked all layers, opening the same "drawers" (expert data) again and again.
After: open a layer's drawer once and process a whole super-chunk of the prompt before moving on. **This code is
@vizdatom's** (Strata #1670, his `layer-major-0.1.43.1` branch); I ported it to 0.1.44.1 and ran it on HIP/Windows.

Same day, RAM 20 GiB: control 896.6 → layer-major **1,177.4** (+31.3%) → control again 901.4 prompt tokens/s; TTFT
67.06 → 51.10 s; decode 26.0 → 27.0 (control again 27.5, so no clear decode gain). The engine's logical expert reads
fell from about 171 to 82 GB per request; GPU busy rose from 71.7% to 84.5%.

My own addition on top: let that pass borrow storage from the expert cache instead of allocating a new buffer, with a
guard on the borrowed capacity. With a 32K super-chunk at 23 GiB: 1,408.9 / 27.5 / 42.70 s. A separately preallocated 32K buffer (RAM 20 GiB, a different setting) reached 1,416.1 prompt tokens/s but only 20.7 decode. A 26 GiB budget was faster in API tests
but ran out of commit margin in real coding use, so it is not recommended.

### Round 4: decode got faster, code got worse → rejected

CPU/IO overlap together with an MTP draft min-p of 0.7 (instead of 0.5) raised decode by 21.9% after a warm-up request (29.7 → 36.2, medians of 5 pairs; answers 384 vs 358 tokens), less on a server's first long request (27.7 → 30.2). But a real coding evaluation dropped from
18/18 to **16/18** functional (empty-input and blank-id edge cases), so the combination was dropped. Two settings changed
together, so neither is blamed alone.

### Round 5: cheaper fetching, and a 2 GB buffer that failed the memory check

A persistent fetch team and CPU/IO overlap without the extra buffer: 1,413.4 / 33.0 / 42.56 s. In a separate same-build comparison, growing the expert read-ahead pool to 2 GB in total raised decode 32.5 → 35.6, but follow-up runs dropped Windows' free commit to 3.73 GiB (1 GB buffer:
3.51 GiB), below the 4 GiB safety line. Not adopted.

### Round 6: let Windows reuse what it already read

In my runs (`STRATA_UNBUFFERED_LOAD=1`, set because the buffered path stopped on my memory guard on this PC: with the 0.1.41 defaults, and with `STRATA_UNBUFFERED_LOAD=0` on the round-5 build at 19 GiB) the engine reads experts from the SSD with the Windows file cache bypassed. The change keeps that for the
initial load, then reopens the expert file so later reads can come from the Windows file cache
([`patches/r6-windows-cached-readfile.patch`](patches/r6-windows-cached-readfile.patch), opt-in). Same binary, same day,
only this switch changed:

| | prompt tok/s | decode tok/s | TTFT s | total s |
| --- | ---: | ---: | ---: | ---: |
| file cache off | 1,399.8 | 28.5 | 43.08 | 56.19 |
| file cache on | 1,476.4 | 36.0 | 40.74 | 51.20 |
| change | +5.5% | **+26.3%** | −5.4% | −8.9% |

The answers were identical pair by pair (345 / 381 / 340 tokens), so it is not faster by answering shorter. Physical
SSD reads fell from 947.9 to 306.7 MiB/s while reading the prompt and from 678.4 to **24.7 MiB/s** while writing, while
the engine's requested read rate did not drop (947.3 → 986.7 and 678.3 → 817.6 MiB/s): most of the data was served by the Windows file cache instead of the SSD (whole-drive counter; the cache size itself was not measured).
Off ran before on and the OS cache was not flushed, so this is not a cold-SSD test.

Why the RAM budget went down to 19 GiB: at 23 GiB a real coding task stopped on the commit guard (3.97 GiB free while
14.83 GiB of physical RAM was still free); 21 GiB stopped after two tasks.

## 5. Same-condition comparisons (the ones that show cause)

| Change | Scope | Prompt | Decode | TTFT |
| --- | --- | --- | --- | --- |
| hipBLASLt table | Swift 24K, RAM 24, 0.1.44.1 release | 404.6 → 695.2 (+71.8%) | 29.9 → 29.2 | 59.51 → 34.67 s |
| layer-major | Flash 60K, RAM 20, same build lineage | 896.6 → 1,177.4 (+31.3%) | 26.0 → 27.0 | 67.06 → 51.10 s |
| Windows file cache | Flash 60K, RAM 19, same binary | 1,399.8 → 1,476.4 (+5.5%) | 28.5 → 36.0 (+26.3%) | 43.08 → 40.74 s |

Three different bottlenecks: the math kernel, the order of data supply, and file-read reuse. Do not add or multiply
these percentages; they were measured under different conditions.

## 6. Did it get smarter? Not shown. What is still broken

This work targeted speed, not accuracy, and accuracy was not compared like for like. Final Flash-Next settings (file cache on), each a separate test:

| Test | Result | Meaning / limit |
| --- | --- | --- |
| fixed set (8 short + 4 long × 3) | 20/20 answers | 12 distinct questions |
| real coding agent (6 tasks × 3) | 18/18 functional, 15/18 strict | same in the file-cache-off arm |
| continuous use (save, restore, cancel …) | 32/33 scored | one wrong answer: the ORBIT rule below |
| long-context system rule | ALPHA **0/3**, BETA 3/3 | see below |

- **Long-context system rule:** the system prompt says to answer `ORBIT` for ALPHA; the 60K document says something
  else. The model followed the document for ALPHA. Repeating the trusted system rule at the end of the prompt gave 3/3.
  It also fails on 0.1.41 as released with FP16 KV at a 23 GiB RAM budget (the original 24 GiB stopped on commit margin before answering), so it is not new; it is a limit of this model/engine combination,
  not something this work fixed.
- **MTP stalls:** in some coding runs the engine stalled on an MTP GPU wait until it was restarted: 3 times with
  the file cache off, 1 time on. Not fixed.
- The strict-check misses are one task: the agent read the v2 file directly instead of hitting the planned v1 error
  first. The code was right; the planned recovery path was not exercised.
- The arithmetic question is now answered correctly (33,100) in the final fixed set, but it was also correct with the
  file cache off, so the cache did not "make it better at math".

## 7. What it feels like when coding

Same binary, file cache off vs on, 6 real coding tasks × 3 (wall time per task, median of 3):

| Task | off s | on s | change |
| --- | ---: | ---: | ---: |
| moving average | 57.32 | 46.36 | −19.1% |
| billing total | 56.52 | 45.93 | −18.7% |
| latest record | 94.15 | 74.73 | −20.6% |
| pagination | 74.48 | 54.29 | −27.1% |
| add tests | 60.16 | 52.37 | −12.9% |
| restore a missing contract | 59.37 | 47.30 | −20.3% |

Median over all 18 runs: 61.45 → 50.31 s (−18.1%). The sum fell 35.7%, mostly because slow stall-and-restart runs
weigh more in a sum; it does not mean coding is 35.7% faster.

## 8. First read vs. follow-up questions

On a follow-up question about the same document the engine reuses the conversation memory: 60,035 of 60,064 tokens
were reused and only 29 were new, and the answer started after **1.09 s** (file cache on; 1.74 s off). That is not
"reading 60K in one second"; the first read of the document still took 42.3 s (on) / 44.1 s (off).

## 9. Does it carry over to Swift 1.5?

Same optimized binary, RAM 19, fresh prompts, file cache off → on (medians of 3):

| Prompt | prompt tok/s | decode tok/s | TTFT s |
| --- | --- | --- | --- |
| 1K | 163.0 → 200.1 | 26.8 → 33.9 | 6.59 → 5.36 |
| 8K | 658.5 → 879.3 | 26.6 → 33.8 | 12.28 → 9.22 |
| 24K | 1,211.7 → 1,321.6 | 28.7 → 32.9 | 19.96 → 18.43 |
| 60K | 1,366.2 → 1,468.3 | 23.5 → 31.1 | 44.04 → 40.99 |

Speed carried over, but both arms hit MTP stalls during continuous quality runs (completed answers 12/13 off,
14/15 on; the miss is the same arithmetic question), so I kept Swift on its previous settings. Swift's decode is still
below the 38.2 tokens/s of the very first measurement under different settings.

## 10. Things that did not work out

- **More GPU expert slots (2,600 requested → 3,456 actual, about 5.64 GiB):** loaded, then warned LOW VRAM on the first
  request. Windows' VRAM budget was 11,935 MiB with 12,030 MiB in use, so the engine saw 0 free even though the card
  was not physically full. Kept 2,200 requested (2,918 actual).
- **Reading the file tier through the memory-mapped file cache (`STRATA_UNBUFFERED_LOAD=0`, round-5 build, RAM 19 GiB):** stopped on the 4 GiB free-RAM guard (3.93 GiB free) during the first 60K request.
- **llama.cpp (official b11669) with this model on this PC:** 9 HIP and Vulkan attempts, none finished three speed
  runs within the memory guard. So there is no valid llama.cpp number here, which is not the same as "0 tokens/s".
- **About 100K tokens:** one earlier run at a 128K context (actual 100,055 tokens, RAM 22) read at 1,185.1 tokens/s with
  3/3 retrieval; the final settings are not validated above 64K.
- **Linux / WSL2:** not tested; no claim that it would be faster.

## 11. Final settings (Flash-Next, this PC)

| Setting | Value | Plain meaning |
| --- | --- | --- |
| context / KV | 65,536 / int8 | room for the conversation / its memory format |
| MTP | spec 4, min-p 0.5 | draft tokens; the 0.7 variant was rejected |
| prompt batch / layer-major super-chunk | 4,096 / 32,768 | how the prompt is cut and walked |
| expert RAM budget | 19 GiB | system RAM, not VRAM |
| GPU expert cache | 2,200 requested (2,918 actual) | the engine grows it itself |
| arena guard / extra read-ahead buffer | on / none | the 1–2 GB buffer was rejected |
| Windows file cache for experts | on (private build) | reopen the expert file cached after load |
| also on | hipBLASLt table, WMMA attention, expert MMQ, MTP side stream, adapt 16 | each checked above |

Several switches exist only in my private build (see the experiment branch). They are not official Strata options.

## 12. Words to avoid when you retell this

| Don't say | Why | Say instead |
| --- | --- | --- |
| "The AI got 3x smarter" | reading speed is not accuracy | "prompt reading got about 3.4x faster (vs. 0.1.41 here)" |
| "It writes 1,500 tokens/s" | 1,500 was the *reading* goal | "it reads 1,476 and writes 36 tokens/s" |
| "It reads 60K in one second" | that was a follow-up with 60,035 reused tokens | "a follow-up on the same document started after about 1.1 s" |
| "This is stock Strata" | rounds 2–6 use private changes | "Strata plus local changes, measured on one PC" |
| "MTP stalls are fixed" | still seen once with the final settings | "fewer stalls in this run; not fixed" |

## Credits

- Strata by Niko1221 and the Strata contributors (MIT License).
- Layer-major prefill by Tomas Vizdal (@vizdatom), ported here; HIP prompt-attention patches by libratechw.
- Measurements, the gfx1101 table, the Windows file-cache change, the shared-arena guard and this write-up by durikim1-cpu.
