# splash35b-20g — Model Profile

**Broker alias**: `splash35b-20g`
**Hugging Face ID**: `mlx-community/Qwen3.6-35B-A3B-4bit`
**Backend**: Splash 1.1.0
**Config location**: `~/llm/configs/mlx-broker.yaml` (alias `splash35b-20g`)

---

## Architecture

| Property | Value |
|---|---|
| **Model family** | Qwen3.5 (Mixture of Experts) |
| **Architecture** | `Qwen3_5MoeForConditionalGeneration` |
| **Total parameters** | 35 billion |
| **Active parameters per token** | ~3.5 billion (8 of 256 experts) |
| **Hidden size** | 2,048 |
| **Head dimension** | 256 (keys 128, values 128) |
| **Attention heads** | 16 (GQA: 2 key-value heads) |
| **Hidden layers** | 40 (10 full-attention + 30 linear-attention) |
| **Vocabulary** | 248,320 tokens |
| **RoPE** | MROPE interleaved — `mrope_section: [11, 11, 10]`, theta 10,000,000, partial factor 0.25 |
| **Full attention interval** | Every 4th layer (10 layers); remainder uses linear attention (sub-quadratic scaling) |
| **Quantization** | 4-bit affine (group size 64), MLP gates at 8-bit |
| **Model type** | `qwen3_5_moe_text` |
| **Layer types** | `linear_attention` repeated, with `full_attention` every 4th block |

### Layer Pattern

```
[linear, linear, linear, full,
 linear, linear, linear, full,
 linear, linear, linear, full,
 ...  (repeats 10x) ...]
 = 40 layers total (10 full, 30 linear)
```

This hybrid architecture means **KV cache scales sub-linearly** — only 25% of layers pay quadratic attention cost, making 256K context tractable on consumer hardware.

---

## Context Window

| Property | Value |
|---|---|
| **max_position_embeddings** | 262,144 (256K tokens) |
| **Configured limit** | None (unconstrained — limited by hardware only) |
| **Broker enforcement** | `_reject_if_over_context_window()` rejects at `max_position_embeddings` |
| **Practical sweet spot** | 128K–200K (reasoning quality holds; recall holds up to 260K) |

### Tested Context Limits (Live Benchmarks)

| Tokens | Status | Quality | Notes |
|---|---|---|---|
| 80K | ✅ Perfect | Full recall | Fast response, no degradation |
| 150K | ✅ Perfect | Full recall | Slightly slower TTFT |
| 200K | ✅ Perfect | Full recall | Works well for deep analysis |
| 240K | ✅ Perfect | Full recall | Near theoretical max |
| 250K | ✅ Perfect | Full recall | Perfect recall on content tests |
| 260K | ✅ Success | Full recall | Past 256K — RoPE extrapolation handles it gracefully |
| 280K | ⚠️ Boundary | — | Tokenizer produces ~254K actual tokens before 262K cap |
| 300K+ | ❌ Rejected | — | Broker returns `prompt exceeds the context window` (HTTP 400) |

**512K is not possible** — it exceeds the model's architectural `max_position_embeddings`. The broker rejects requests at the admission layer before the model sees them.

---

## Memory Profile

### CPU Resident Set (RSS)

| Process | RSS | Role |
|---|---|---|
| Splash Engine (35B) | ~7.4 GB | GPU compute worker — Metal residency |
| Splash Server (35B) | ~0.7 GB | HTTP/stdio API layer |
| mlx_lm 9B worker | ~5.2 GB | Always-pinned small model (port 18081) |
| MLX Broker | ~0.1 GB | Request routing / admission |
| **Total CPU RSS** | **~13.4 GB** | |

### GPU / Metal Residency

| Component | Size |
|---|---|
| 35B weights (4-bit) | ~20.7 GB |
| 9B weights (4-bit) | ~5.8 GB |
| KV cache (260K context) | ~8–12 GB (estimated) |
| **Total GPU** | **~35–39 GB (estimated)** |

### Configuration Limits

| Setting | Value |
|---|---|
| `splash_max_memory_gb` | 55 GB (hard GPU ceiling) |
| `expected_rss_gb` | 20.7 GB (expected CPU-side) |
| `splash_max_context_tokens` | **Not set** (no explicit cap) |
| `extra_args` | `--default-reasoning-effort none` |

### System Context (Your Hardware)

- **RAM**: 128 GB Apple Silicon — sufficient for full workloads
- **GPU (Metal)**: Up to 55 GB reservable by Splash — current usage ~35–39 GB with 260K context
- **Disk**: ~389 GB free — no storage pressure

**Note**: Splash's GPU residency is tracked separately from CPU RSS (psutil under-reports GPU allocation). The 55 GB `splash_max_memory_gb` config is the real constraint, not the 20.7 GB expected RSS.

---

## Configuration (broker config)

```yaml
splash35b-20g:
  hf_model_id: mlx-community/Qwen3.6-35B-A3B-4bit
  expected_rss_gb: 20.7
  backend: splash
  splash_max_memory_gb: 55
  extra_args:
    - --default-reasoning-effort
    - none
```

The `--default-reasoning-effort none` flag is critical for large-context workloads. Thinking traces consume context budget and cause the truncation issues documented in the alias (11/16 humaneval items truncated at 1536 code cap when thinking was active).

---

## Comparison to Other Models

| Model | Context Limit | Params (Active) | Suitable Use |
|---|---|---|---|
| **splash35b-20g** | 256K | 35B (3.5B active) | Long-context reasoning, full-codebase analysis |
| qwen38-27b-splash-25g | 256K (estimated) | 27B | Mid-range, balanced speed/capacity |
| qwen35-9b-6g | ~32K (estimated) | 9B (dense) | Fast bounded-work: search, small tasks |

---

## References

- Hugging Face: `mlx-community/Qwen3.6-35B-A3B-4bit`
- Source model family: Qwen3.5 (MoE) — hybrid linear/full attention architecture
- Splash backend: Splash 1.1.0 (fast inference, ~2× decode speed vs mlx_lm)
- Broker source: `src/mlx_broker/core.py:_reject_if_over_context_window()`
- Config: `~/llm/configs/mlx-broker.yaml` (alias `splash35b-20g`)
