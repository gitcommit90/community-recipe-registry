# Nemotron-3-Super-120B-A12B-NVFP4 + MTP, single node (vLLM)

Single-Spark serving config for NVIDIA's 120B flagship, derived from the leaderboard's current-best single-node entry for this model with two changes, both measured:

- **`gpu_memory_utilization: 0.88`** — the lineage's 0.75 fails engine init on the tested build (vLLM 0.23.1rc1.dev1408): after the ~80 GB weights + MTP head there is no memory left for the KV cache blocks ("No available memory for the cache blocks").
- **`num_speculative_tokens: 3`** (the lineage runs 1). Acceptance holds as depth increases: mean acceptance length 1.61 (nst=1) → 2.64 (nst=3).

## Measured (single GB10, tg128 c1 @ d0, pp2048, prefix caching on)

| config | tg128 (c1) | notes |
|---|---|---|
| no MTP (same shape) | 16.48 ± 0.002 (N=3) | matches the board's no-MTP entries (15.2–16.6) |
| MTP nst=1 (the lineage) | 23.2 ± 1.6 (N=3) | reproduces the board-best entry (23.71) at 98% |
| **MTP nst=3 (this recipe)** | **23.6 ± 1.7 (N=5)** | statistically indistinguishable from the published 23.71 — a parity claim, not a faster one |

The throughput delta between nst=1 and nst=3 is within run variance at these Ns. One baseline note: the 23.71 board-best entry runs MTP at nst=1 per its own recipe; the measured no-MTP baseline for this model is 16.5, matching the board's low mode (15.2–16.6).

## Notes

- `max-num-seqs 4` is inherited from the lineage entry; this is a single-user-lane config — concurrent aggregate caps near 40 t/s.
- Long sustained sweeps at depth will thermally throttle a desk Spark: a 120B's deep cells run minutes of near-max load each. Short bursts and shallow cells measure clean; budget cool-down gaps if you benchmark the full official profile.
- The container is pinned to the tested tag `ghcr.io/spark-arena/dgx-vllm-eugr-nightly:2026072302` (vLLM 0.23.1rc1.dev1408): the 0.88 memory setting is build-sensitive, so a floating tag can re-break engine init. The lineage entry ran an upstream `vllm-openai` image; the comparisons above cross that runtime difference.
- Host: Rocky Linux 10.2, CIQ 6.18 kernel, open driver 610.43.03. Receipts and raw per-run data:
  [reproduce-Nemotron-3-Super-NVFP4-mtp-2026-07-26.txt](https://github.com/maxspevack/spark-rocky/blob/main/receipts/reproduce-Nemotron-3-Super-NVFP4-mtp-2026-07-26.txt)
  (matrix + headline N=5) and
  [reproduce-Nemotron-3-Super-NVFP4-probes-2026-07-25.txt](https://github.com/maxspevack/spark-rocky/blob/main/receipts/reproduce-Nemotron-3-Super-NVFP4-probes-2026-07-25.txt)
  (MTP decomposition + acceptance data).
