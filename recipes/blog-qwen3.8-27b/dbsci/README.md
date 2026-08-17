# blog-qwen3.8-27b — one command, three engines

A matched set of three recipes serving **Qwen3.8-27B** on a single DGX Spark (GB10) under
**vLLM**, **SGLang** and **llama.cpp**, built so that the only thing that changes between
runs is the recipe name.

To reproduce the original post's tables, run the companion blog-parity profile:

```bash
sparkrun benchmark perf @community/blog-qwen3.8-27b-fp8-vllm-dbsci        --profile blog-qwen3.8-27b-dbsci
sparkrun benchmark perf @community/blog-qwen3.8-27b-fp8-sglang-dbsci      --profile blog-qwen3.8-27b-dbsci
sparkrun benchmark perf @community/blog-qwen3.8-27b-q4kxl-llama-cpp-dbsci --profile blog-qwen3.8-27b-dbsci
```

To put results on the leaderboard instead, name no profile at all:

```bash
sparkrun arena login
sparkrun benchmark perf @community/blog-qwen3.8-27b-fp8-vllm-dbsci --arena
```

`--arena` selects the current Spark Arena profile and uploads the result, so it stays correct
as that profile is revised — which is why it is better than spelling a version out here.

Either way, one command launches the workload, waits for it to come up healthy, runs the
ladder, stops the workload and writes a results YAML.

## Why these exist

They are transcribed from [Saiyam Pathak's *Running Qwen3.8-27B on DGX
Spark*](https://blog.kubesimplify.com/qwen3-8-27b-on-dgx-spark) (2026-08-17), which measured
all three engines on a GB10 the week the weights dropped — but drove sparkrun only for the
vLLM leg, and hand-rolled `docker run` and per-engine benchmark tooling for the others. This
set is the same work expressed as three recipes, so the comparison is reproducible by anyone
with a Spark and one command per engine.

The blog-parity profile (`benchmarking/blog-qwen3.8-27b-dbsci.yaml` in this registry) is a
transcription of that post's own llama-benchy call: pp 2048, tg 128, depths 0 / 16384 / 32768,
concurrency 1 / 2 / 5 / 10, prefix caching on. Twelve cells — deliberately the original grid,
so these runs land beside the post's numbers rather than beside a different experiment.

## What is held identical

The recipes are matched, not tuned. A difference in the numbers should be a difference in the
engine, so everything that *can* be the same is:

| | value |
|---|---|
| Nodes | 1 |
| Port | 8000 |
| Served name | `qwen3.8-27b` |
| `benchmark.tokenizer` | `Qwen/Qwen3.8-27B` |
| Context | 131072 (vLLM / SGLang) |
| GPU memory utilization | 0.80 (vLLM / SGLang) |

`benchmark.tokenizer` is the one that is easy to leave out and quietly ruins the comparison.
The served name is not a Hugging Face id, so without the pin llama-benchy falls back to its
GPT-2 tokenizer when building the depth prompts, and "32768 tokens deep" means a different
amount of text in each run. It matters most for the llama.cpp recipe, where the GGUF repo
carries no HF tokenizer at all.

## What is deliberately *not* identical

Three honest asymmetries. Each is a case where forcing parity would mean benchmarking a
configuration nobody runs.

**1. llama.cpp serves different weights.** vLLM and SGLang both serve
`Qwen/Qwen3.8-27B-FP8` (~28.75 GB). llama.cpp serves `unsloth/Qwen3.8-27B-GGUF:Q4_K_XL`
(16.68 GiB), because a Q4 GGUF is what people actually run under llama.cpp and an FP8 one
does not exist. On a bandwidth-bound box this dominates single-stream decode: every token
streams the entire weight set, so at GB10's ~273 GB/s the 16.7 GB quant has a ceiling near
16 t/s where the FP8 checkpoint sits near 9. llama.cpp leading single-stream is arithmetic,
not engine quality.

**2. SGLang ships with NEXTN speculation on, the other two ship without.** It roughly doubles
single-stream decode and it is the configuration a person actually wants to serve, so it is
on by default. For the strictly matched three-way run, copy the file and delete the four
`speculative_*` keys from `defaults:` along with the four matching `--speculative-*` lines
from `command:` — `-o` overrides a value and cannot remove a flag written into the
`command:` template.

**3. llama.cpp's context is 409600, not 131072 — and that is not a bigger window.** In
llama-server, `--ctx-size` is the *total* context divided evenly across `--parallel` slots,
not a per-request limit like vLLM's `--max-model-len`. The profile's deepest cell is depth
32768 at concurrency 10, so each of 10 slots needs ~32768 + 2048 + 128 ≈ 34,944 tokens:
10 × 40960 = 409600 total, or about 40K of usable window per request. `ctx_size` and
`parallel` have to move together — raising one alone fails the deep cells or the concurrent
ones respectively.

This is the one number in the set that is derived rather than transcribed, because the source
post never hit it: its llama.cpp leg was measured with `llama-bench`, which has no
concurrency dimension. It is also the number most likely to be wrong; if the deep concurrent
cells fail, this is where to look. Note that the Spark Arena profile reaches deeper than the
blog-parity one, so `--arena` on the llama.cpp recipe needs this arithmetic redone before it
will complete.

## Published numbers (source post, not measured here)

⚠ **These are the source post's measurements, reproduced for reference. They have not been
re-measured on Spark Arena hardware.** The grid matches, but the llama.cpp row does not come
from llama-benchy at all — it is `llama-bench`, run in-container against the GGUF, which is
why its concurrency cell is empty. Treat them as the shape of the answer, not as this set's
results. Submit your own with `--arena`.

pp2048 / tg128 at depth 0, single DGX Spark GB10:

| Engine | Weights | Prefill c=1 | Decode c=1 | Decode c=10 agg |
|---|---|---:|---:|---:|
| vLLM FP8 | 28.75 GB | 1,914 t/s | 8.2 t/s | 57.9 t/s |
| SGLang FP8 | 28.75 GB | 1,225 t/s | 7.7 t/s | 54.3 t/s |
| SGLang FP8 + NEXTN | 28.75 GB | — | 13.4 t/s | 71.0 t/s |
| llama.cpp Q4_K_XL | 16.68 GiB | 837 t/s | 11.6 t/s | n/a (`llama-bench`) |

Context flatness is the architectural headline and it holds across engines: 48 of the 64
layers are Gated DeltaNet with constant-size state, so decode barely sags with depth (vLLM
FP8: 8.2 t/s at depth 0, 7.9 t/s at 32K).

## Status and pitfalls

- **Unverified on Spark Arena hardware.** These are faithful transcriptions of a published
  configuration, validated for schema and command rendering only. Nothing here has been
  booted on a GB10 by the recipe author.
- **The SGLang image tag is deliberately floating** (`lmsysorg/sglang:latest-cu130`). The
  source post's first attempt used the pinned dev build the qwen3.5 recipe carried, and it
  loaded the day-zero checkpoint with no warning, passed the health check, answered every
  request, and generated token soup. Upstream fixed it instantly. On a day-zero checkpoint
  the pin is the hazard — coherence-test before trusting a number, and do not reach for
  `-b skip_coherence=true` on this recipe.
- **Unified memory is shared.** Stop one engine before starting the next. A resident
  container holding 18 GB shows up as `Free memory 74.82/121.69 GiB is less than desired 0.8
  utilization` at launch. `nvidia-smi` cannot report memory on GB10 — use `free -h`.
- For peak throughput on this checkpoint rather than a matched comparison, see
  `@official/qwen3.8-27b-fp8-mtp-vllm` and the NVFP4 / DSpark official recipes. They are
  faster and they are not comparable to this set.

## Credits

Configuration and measurements from [Saiyam Pathak](https://blog.kubesimplify.com/qwen3-8-27b-on-dgx-spark).
Packaged as recipes by [@dbsci](https://forums.developer.nvidia.com/u/dbsci/summary).
