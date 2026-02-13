---
module: "index"
design_refs:
  - "docs/design/core-index.md"
code_refs:
  - "src/core/index/**"
validation_gates:
  - "gate:leann-index-latency"
---

# Spec: Index Acceleration Architecture

**Status**: DRAFT
**Layer**: Services (L1-L2)
**Epic**: lev-h038 (Index backends)
**Last Updated**: 2026-02-13

---

## Executive Summary

Make LEANN index embedding computation config-driven with capability-based hardware detection. Instead of hardcoding `sentence-transformers` on CPU, auto-detect available acceleration (MPS, CUDA, ROCm, Vulkan, CPU) and select the optimal embedding mode, model, device, and batch size. Expose configuration via `~/.config/lev/index.yaml` with intelligent `auto` defaults.

---

## Context

### Current State

- `embedding_compute.py` supports 5 modes: `sentence-transformers`, `ollama`, `openai`, `gemini`, `mlx`
- MPS/CUDA/CPU auto-detect exists inside `compute_embeddings_sentence_transformers()` but isn't exposed
- `leann_bridge.py` hardcodes `facebook/contriever` with no config passthrough
- `server.py` uses `embedder/` module (separate from LEANN core) with its own backend system
- No unified config file — selection is scattered across env vars and defaults
- HNSW backend requires custom SWIG-compiled faiss (missing `.so` on this machine)

### Target State

- Single config file (`~/.config/lev/index.yaml`) drives all embedding decisions
- Capability detection runs once at startup, caches results
- `auto` mode selects optimal stack for detected hardware
- HNSW backend gracefully falls back to standard `faiss-cpu` when custom SWIG build unavailable

---

## SWIG vs faiss

| Term | What It Is |
|------|------------|
| **faiss** | Facebook AI Similarity Search — C++ library for vector similarity indexes |
| **faiss-cpu** | Standard pip package — Python bindings via pre-compiled wheels |
| **SWIG** | Simplified Wrapper and Interface Generator — compiles C++ into Python `.so` extensions |
| **LEANN custom faiss** | SWIG-compiled fork with CSR compression, recompute mode, compact HNSW graphs |

LEANN's `HNSWBuilder` imports `from . import faiss` (the custom SWIG build), not `import faiss` (pip). Without the compiled `.so`, `build_index()` fails with `ImportError`. The fix: try custom first, fall back to standard.

---

## Capability Detection

Detection is **capability-driven, not OS-driven**. We probe hardware features, not `platform.system()`.

```
detect_capabilities() → CapabilityProfile
  ├── torch.backends.mps.is_available()       → has_mps (Apple Metal)
  ├── torch.cuda.is_available()               → has_cuda (NVIDIA)
  ├── torch.version.hip is not None           → has_rocm (AMD ROCm)
  ├── shutil.which("vulkaninfo")              → has_vulkan (general GPU)
  ├── shutil.which("ollama")                  → has_ollama
  ├── os.cpu_count()                          → cpu_cores
  └── psutil.virtual_memory().total           → ram_gb
```

### Auto-Selection Waterfall

| Priority | Capability | Embedding Mode | Model | Device | Batch Size |
|----------|-----------|----------------|-------|--------|------------|
| 1 | MPS available | sentence-transformers | facebook/contriever | mps | 128 |
| 2 | CUDA available | sentence-transformers | facebook/contriever | cuda | 256 |
| 3 | Ollama running | ollama | nomic-embed-text | n/a | 32 |
| 4 | CPU fallback | sentence-transformers | all-MiniLM-L6-v2 | cpu | 32 |

**Performance benchmark evidence** (M3 Studio, 2026-02-12):

| Approach | Build (docs/s) | Build (beads/s) | Search P50 |
|----------|---------------|-----------------|------------|
| Contriever/MPS | 68.4 | 896.7 | 30.6ms |
| MiniLM/MPS | 32.9 | 555.6 | 21.8ms |
| Ollama/nomic | 24.7 | 116.9 | 82.6ms |

**Quality benchmark evidence** (MTEB English Retrieval snapshot, 2025-08-31):

| Model | Retrieval Average (NDCG@10) | Source |
|-------|------------------------------|--------|
| nomic-embed-text-v1.5 | 53.01 | MTEB leaderboard (`default.jsonl`) |
| text-embedding-3-small | 51.08 | MTEB leaderboard (`default.jsonl`) |
| all-MiniLM-L6-v2 | 41.95 | MTEB leaderboard (`default.jsonl`) |
| contriever-base-msmarco | 41.88 | MTEB leaderboard (`default.jsonl`) |

Note: the MTEB row is `contriever-base-msmarco` (MS MARCO-tuned), not `facebook/contriever` exactly. Local retrieval quality on Lev corpora should be validated with a labeled evaluation set before locking a quality-first default.

---

## Configuration Schema

File: `~/.config/lev/index.yaml`

```yaml
# LEANN Index Configuration
# All values support "auto" for capability-driven detection

embedding:
  mode: auto                    # auto | sentence-transformers | ollama | openai | gemini | mlx
  model: auto                   # auto | model name (e.g. facebook/contriever)
  device: auto                  # auto | mps | cuda | cpu
  batch_size: auto              # auto | integer
  fp16: true                    # use half-precision (MPS/CUDA)
  warmup_on_start: true         # run dummy embed on server start to eliminate cold-start

index:
  backend: hnsw                 # hnsw | diskann
  faiss_mode: auto              # auto | custom-swig | standard
  distance_metric: mips         # mips | cosine | l2

server:
  port: 50052
  max_workers: 4

# Override per-collection (optional)
collections:
  lev-docs:
    model: facebook/contriever
  lev-beads:
    model: facebook/contriever
```

### Resolution Rules

1. Explicit value → use it
2. `auto` → resolve via `CapabilityProfile`
3. Per-collection override → merge over defaults
4. Environment variables → override config file (`LEANN_EMBEDDING_MODE`, `LEANN_DEVICE`, etc.)

---

## BDD Scenarios

### Scenario: Auto-detect MPS on Apple Silicon

```gherkin
Given a machine with Apple Silicon (M-series chip)
And PyTorch is installed with MPS support
When the index server starts with mode=auto
Then the capability detector reports has_mps=true
And the embedding mode resolves to "sentence-transformers"
And the device resolves to "mps"
And the batch_size resolves to 128
```

### Scenario: Fall back to CPU when no GPU available

```gherkin
Given a machine with no GPU acceleration
And PyTorch MPS and CUDA are both unavailable
When the index server starts with mode=auto
Then the capability detector reports has_mps=false, has_cuda=false
And the embedding mode resolves to "sentence-transformers"
And the device resolves to "cpu"
And the model resolves to "all-MiniLM-L6-v2" (smaller, faster on CPU)
```

### Scenario: Prefer Ollama when available and no GPU

```gherkin
Given a machine with no GPU but Ollama is running
And the ollama process is listening on port 11434
When the index server starts with mode=auto
Then the capability detector reports has_ollama=true
And the embedding mode resolves to "ollama"
And the model resolves to "nomic-embed-text"
```

### Scenario: HNSW faiss fallback

```gherkin
Given the custom SWIG faiss extension is not compiled
And standard faiss-cpu is installed via pip
When a LeannBuilder builds an index with backend=hnsw
Then it falls back to standard faiss-cpu
And logs a warning about missing CSR compression
And the index builds successfully
```

### Scenario: Config file overrides auto-detection

```gherkin
Given a config file at ~/.config/lev/index.yaml
And the config specifies embedding.mode=ollama
When the index server starts
Then it uses ollama mode regardless of detected capabilities
And the model comes from config (not auto-selected)
```

### Scenario: Per-collection model override

```gherkin
Given a config with collections.lev-beads.model="all-MiniLM-L6-v2"
And the default model is "facebook/contriever"
When building the lev-beads collection
Then it uses all-MiniLM-L6-v2 for that collection
And lev-docs still uses facebook/contriever
```

### Scenario: GPU warmup eliminates cold-start

```gherkin
Given warmup_on_start=true in config
And device=mps
When the index server starts
Then it runs a dummy embedding before accepting requests
And the first real query has P50 < 50ms (no 300-500ms cold-start)
```

---

## Dependencies

- `torch >= 2.0` (for MPS/CUDA detection)
- `sentence-transformers` (primary embedding backend)
- `faiss-cpu` (fallback for HNSW index building)
- `pyyaml` (config file parsing)
- `psutil` (optional, for RAM detection)
- `requests` (for Ollama API calls)

---

## Breaking Changes

None. All new functionality is additive. Existing behavior preserved when no config file exists (defaults match current hardcoded values).

---

## Rollback Plan

1. Delete config file → system reverts to hardcoded defaults
2. Revert `capabilities.py` and `config.py` → no impact on existing code
3. HNSW fallback patch is backwards-compatible (custom SWIG still preferred when available)
