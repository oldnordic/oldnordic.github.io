---
layout: post
title: "Transformer X-Ray: Attention Commitment Depth Across 6 Architectures"
date: 2026-06-20
categories: mechanistic-interpretability llama-cpp
---

Cross-architecture attention analysis using llama.cpp tensor callbacks and JS-divergence.

---

## Overview

In my [previous post]({{ site.baseurl }}{% post_url 2026-06-16-transformer-xray %}) I showed how attention geometry changes between correct and wrong predictions in Qwen2.5-7B. That was one model. This post scales it to **6 architectures** — a standard transformer, a sliding-window hybrid, a liquid MoE, and two sizes of the same family — using a direct llama.cpp integration that captures raw `kq_soft_max` tensors during inference.

The central question: **does architecture or model size determine how deeply a model processes context before committing to an answer?**

The answer is architecture. By a large margin.

---

## Method

### Tensor extraction via llama.cpp callback

llama.cpp exposes `cb_eval` — a per-tensor callback that fires after every operation in the compute graph. By disabling Flash Attention (`LLAMA_FLASH_ATTN_TYPE_DISABLED`), the intermediate `kq_soft_max-{layer}` tensors are materialized and captured:

```cpp
ctx_params.flash_attn_type = LLAMA_FLASH_ATTN_TYPE_DISABLED;
ctx_params.cb_eval = tensor_capture_callback;
```

Each captured tensor has shape `[n_kv, n_tokens, n_head]` — the full softmax attention weight matrix per layer. Written to binary files, then parsed in Rust for analysis.

**GPU note:** tensor values are bit-identical between CPU and GPU inference — verified at 0.000% relative difference across all layers. All runs used GPU offload for speed.

### Models tested

| Model | Layers | Heads | Architecture |
|-------|--------|-------|-------------|
| Gemma-4-E2B Q4_0 | 35 | 8 (GQA, kv=1) | SWA hybrid — 28/35 layers use sliding window |
| LFM2.5-8B Q6_K | **6** | 32 | Liquid MoE — only 6 standard attention layers |
| Llama-3.2-1B Q4_0 | 16 | 32 | Standard transformer |
| Phi-3-mini Q4 | 32 | 32 | Standard transformer (Microsoft) |
| Qwen2.5-0.5B Q4_0 | 24 | 14 | GQA |
| Qwen2.5-7B Q4_K_M | 28 | 28 | GQA |

### Prompts

40 factual completion prompts across 6 categories: `capital_city`, `location_fact`, `memorized_phrase`, `entity_completion`, `science_fact`, `historical_year`.

Each prompt has a **wrong suffix** — e.g. `"The capital of France is"` → wrong: `" Berlin"`. Both variants are run through each model and the attention distributions are compared layer by layer using **Jensen-Shannon divergence**.

```
JS-divergence(layer) = divergence between attention distribution
                       on [neutral prompt] vs [prompt + wrong suffix]
```

High JS-divergence at layer N means: adding the wrong suffix disrupts attention most at layer N. That layer is where the model *commits* to context.

---

## Results

### 1. Entropy trajectory — architecture fingerprint

![Entropy by layer](/assets/images/fig_a_entropy_trajectory.png)

Three distinct regimes:

- **LFM2.5-8B** (dots, 6 attention layers): highest entropy of all (0.826). When a model only uses attention 6 times across the full depth, those layers distribute attention broadly — the liquid/Mamba layers do the recurrence work in between.
- **Gemma-4-E2B**: high entropy (0.810) but a continuous trajectory. SWA layers cannot sink to position 0 within the sliding window — forces distributed attention compared to standard transformers.
- **Llama, Phi-3, Qwen**: lower entropy (0.47–0.55), stronger position-0 sink. Standard dense attention converges toward BOS token across all layers.

### 2. Attention sink strength

![Sink weight by layer](/assets/images/fig_e_sink_weight.png)

BOS token (position 0) sink strength reveals the same split:

| Model | Mean sink weight |
|-------|----------------|
| Llama-3.2-1B | **0.835** — strongest |
| Phi-3-mini | 0.793 |
| Qwen2.5-0.5B | 0.711 |
| Qwen2.5-7B | 0.684 |
| Gemma-4-E2B | 0.593 |
| LFM2.5-8B | **0.445** — weakest |

LFM's attention layers don't need to be BOS sinks. The liquid layers maintain state across positions without a dedicated sink token.

### 3. Commitment depth — where does attention commit?

![Commitment depth](/assets/images/fig_b_commitment_depth.png)

This is the main result. Each bar shows at which layer the JS-divergence between neutral and wrong context peaks — the *commitment depth*.

**Qwen (both sizes) commits at layer 0.** Adding " Berlin" to "The capital of France is" disrupts attention immediately, in the first layer. The model pattern-matches factual associations in the embedding + first attention pass.

**Gemma-4 commits at layer 6 for capitals, layer 14 for everything else.** The SWA architecture forces deeper processing — facts are retrieved gradually across layers.

**LFM commits at layers 6–18** — spread across its 6 available attention layers. With only 6 checkpoints, commitment happens progressively at each one.

**Phi-3 is the outlier for science facts: commits at layer 31** (the last layer). Science facts (`"DNA stands for deoxyribonucleic"`) require full-depth processing in Phi-3 — the wrong suffix isn't resolved until the final attention pass.

### 4. Size doesn't change commitment depth

![Qwen 0.5B vs 7B](/assets/images/fig_d_qwen_size_comparison.png)

Qwen2.5-0.5B and 7B produce nearly identical commitment curves — same peak layer, very similar JS-divergence magnitude (0.5B slightly higher, meaning it's *more sensitive* to wrong context).

Scale shifts the magnitude of disruption, not where it happens. The commitment depth is baked into the architecture and training, not the parameter count.

### 5. JS-divergence by layer — all models

![JS-divergence trajectory](/assets/images/fig_c_divergence_trajectory.png)

`historical_year` shows the highest JS-divergence across all models (5–10× higher than other categories). Historical years like `"WWII ended in"` → wrong: `" 1950"` create the strongest attention disruption — these facts are the hardest to inject wrong context into.

| Category | strongest disruption in |
|----------|------------------------|
| historical_year | Qwen2.5-0.5B: JSD=0.109 at layer 0 |
| science_fact | Phi-3-mini: JSD=0.030 at layer 31 |
| capital_city | Early (<layer 6) for all models |

---

## Key Takeaways

1. **Architecture >> size for commitment depth.** Qwen2.5-7B commits at the same layer as Qwen2.5-0.5B. Gemma-4 commits 14 layers deeper regardless.

2. **Hybrid architectures distribute commitment.** LFM's liquid layers process context between attention checkpoints — commitment is smeared across the 6 available layers rather than spiking at one.

3. **SWA prevents early sinking.** Gemma-4's sliding window layers can't always reach position 0, resulting in higher entropy and later commitment compared to standard transformers of similar depth.

4. **Phi-3 needs full depth for science.** Science facts process through all 32 layers in Phi-3 before attention commits — the deepest commitment of any model/category pair in this dataset.

5. **Historical facts are the hardest to corrupt.** Every model shows peak JS-divergence on `historical_year` — these associations are most robustly encoded and most disrupted by wrong suffixes.

---

## Code

All code is in [`geographdb-experiments`](https://github.com/oldnordic/geographdb-experiments):

- `extract_tensors.cpp` — llama.cpp tensor callback, captures `kq_soft_max-{layer}` to binary
- `src/bin/xray_llama.rs` — Rust: entropy, sink, JS-divergence analysis; `compare` subcommand
- `scripts/run_xray_variants.sh` — runs neutral + wrong variants, calls compare
- `scripts/xray_article_plots.py` — generates figures

Build:
```bash
g++ -std=c++17 -O2 extract_tensors.cpp -I include -I ggml/include \
    -L /usr/lib -lllama -lggml-base -lggml -o extract_xray

cargo build --bin xray_llama

./extract_xray model.gguf "The capital of France is" /tmp/xray_dump
cargo run --bin xray_llama -- /tmp/xray_dump
cargo run --bin xray_llama -- compare /tmp/neutral /tmp/wrong --id france_capital
```

---

## What's Next

This analysis has two known limitations worth addressing directly.

**40 prompts is statistically thin.** Per-category that's 4–8 prompts — enough to show a signal, not enough to claim a stable distribution. Some findings (Phi-3 science_fact at layer 31, LFM commitment spread) rest on small samples. The next run will scale to 500–1000 prompts per category using automatically generated factual completions from Wikidata and common benchmarks (TriviaQA, NaturalQuestions), giving enough data to report confidence intervals on commitment depth.

**Categories are hand-crafted.** Six categories designed by hand don't cover the full space of factual knowledge types. The interesting distinction — *pattern-matching facts* (capitals, symbols) vs *compositional facts* (historical causality, scientific relationships) — is real but not cleanly separated in the current dataset. Next step: use an LLM to auto-generate prompts stratified by reasoning depth (single-hop vs multi-hop retrieval), then verify the commitment depth split holds.

Both improvements are buildable on the same infrastructure. The `xray_prompt_pairs_v1.jsonl` format and the `run_xray_variants.sh` pipeline are designed to scale — swapping in a larger prompt file requires no code changes.

---

*Data: 6 models × 40 prompts × 2 variants = 480 inference runs. All on CPU+GPU hybrid (ROCm/AMD RX 7900 XT). Attention weights are GPU/CPU bit-identical — verified at 0.000% relative error.*
