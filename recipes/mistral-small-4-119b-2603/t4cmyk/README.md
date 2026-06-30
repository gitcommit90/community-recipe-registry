# Mistral-Small-4-119B-2603 — FP8 (2-node) & NVFP4 (single-node)

Recipes for [Mistral Small 4](https://mistral.ai/news/mistral-small-4/) — Mistral's unified reasoning, coding, and multimodal MoE (Apache 2.0). 119B total params, 6B active per token, 128 experts with 4 active.

Two variants:
- **`…-fp8-vllm`** — official FP8 checkpoint (`mistralai/Mistral-Small-4-119B-2603`), 2-node with TP=2.
- **`…-nvfp4-vllm`** — NVFP4 (4-bit) quant (`mistralai/Mistral-Small-4-119B-2603-NVFP4`), single-node with TP=1.

Both use the `TRITON_MLA` attention backend with Mistral's native `tool_call_parser` and `reasoning_parser` for agentic workflows.

## Runtime / VRAM

| Variant | Nodes | TP | Total weights | Per-node |
|---------|-------|----|---------------|----------|
| FP8     | 2     | 2  | ~121 GB       | ~60 GB   |
| NVFP4   | 1     | 1  | ~71 GB        | ~71 GB   |

Both use `gpu_memory_utilization 0.8` with 262K context.

## Notes

Reasoning is enabled by default (`reasoning_effort: "high"`). To disable, override:
```
sparkrun run @community/mistral-small-4-119b-2603-fp8-vllm-t4cmyk \
  --default-chat-template-kwargs '{"reasoning_effort": "none"}'
```

## Run

```
sparkrun run @community/mistral-small-4-119b-2603-fp8-vllm-t4cmyk
sparkrun run @community/mistral-small-4-119b-2603-nvfp4-vllm-t4cmyk
```

## Performance

Both variants handle agentic coding comfortably — tool calling, multi-turn reasoning, and code generation all sit in the **20–50 t/s** range on DGX Spark. I haven't run formal benchmarks, but in practice the NVFP4 single-node setup is snappy enough for interactive use, while the FP8 2-node gives you the full-precision weights if you need them.

## Author

[@t4cmyk](https://github.com/t4cmyk)
