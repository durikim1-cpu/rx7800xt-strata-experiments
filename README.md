# Strata on an RX 7800 XT (Windows): 60K-token prompt, first token in 41 s instead of 137 s

Personal experiment notes: running [Strata](https://github.com/Niko1221/Strata) with Qwen3.8-Flash-Next on one AMD
gaming PC (RX 7800 XT 16 GB, Ryzen 5 9600X, 64 GB DDR5, **Windows 11**, HIP), and making it faster without new
hardware or a different model. Not an official Strata result.

| Fresh 60,047-token prompt, 64K context | Prompt tok/s | Decode tok/s | First token |
| --- | ---: | ---: | ---: |
| Strata 0.1.41 as released (FP16 KV, RAM 24 GiB) | 437.0 | 29.9 | 137.5 s |
| after six rounds of changes (int8 KV, RAM 19 GiB) | **1,476.4** | **36.0** | **40.7 s** |

That is a timeline with many changes, not one switch. Decode is still short of my 40–50 goal, MTP stalls still happen in long continuous runs,
and this work makes no accuracy claim (one long-context instruction test still fails). The full write-up for beginners is **[REPORT.md](REPORT.md)**.

## What is in this repository

| Path | What |
| --- | --- |
| [`REPORT.md`](REPORT.md) | the whole story: three kinds of speed, each round, what was rejected, what is still broken |
| [`tables/gfx1101-hipblaslt-100500.txt`](tables/gfx1101-hipblaslt-100500.txt) | a hipBLASLt tuning table for gfx1101 (none ships with Strata): +72% prompt speed at 24K on the 0.1.44.1 release |
| [`results/gfx1101-hipblaslt-table/`](results/gfx1101-hipblaslt-table/) | how the table was made, per-row timings, with/without runs |
| [`results/official-0.1.41-and-file-cache-ab/`](results/official-0.1.41-and-file-cache-ab/) | 0.1.41 baseline and the Windows file-cache A/B, per-run data and configs |
| [`results/history-60k.csv`](results/history-60k.csv) | one row per round at 60K |
| [`patches/r6-windows-cached-readfile.patch`](patches/r6-windows-cached-readfile.patch) | Windows: read experts through the file cache after load (+26% decode on this PC), against 0.1.44.1 |
| [`patches/shared-arena-guard.patch`](patches/shared-arena-guard.patch) | layer-major staging borrowed from the expert cache; applies to my tree after the layer-major port |
| [`scripts/`](scripts/) | the Windows calibration script for the hipBLASLt table |

The full source of my experiment build, one commit per round on top of v0.1.44.1, is the `rx7800xt-experiments`
branch of my Strata fork: https://github.com/durikim1-cpu/Strata/tree/rx7800xt-experiments

## Upstream

- Community benchmark (this PC): https://github.com/Niko1221/Strata/pull/3057
- gfx1101 hipBLASLt table: https://github.com/Niko1221/Strata/pull/3058
- Windows cached ReadFile proposal: https://github.com/Niko1221/Strata/issues/3059
- Layer-major on HIP/Windows (comment on #1670): https://github.com/Niko1221/Strata/issues/1670#issuecomment-6250884201

## Credits

- **Strata** by Niko1221 and the Strata contributors, MIT License.
- **Layer-major prefill** used in round 3 is by **Tomas Vizdal (@vizdatom)** (Strata #1670, his
  `layer-major-0.1.43.1` branch); I only ported and measured it. The HIP prompt-attention patches in that build are by
  **libratechw**.
- Everything else here (measurements, the gfx1101 table and its script, the Windows file-cache change, the shared-arena
  guard, the write-up) is mine.

## License

MIT (see [LICENSE](LICENSE)). The patches modify Strata source and stay under Strata's MIT License as well.
