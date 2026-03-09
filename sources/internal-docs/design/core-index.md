# 04 - Core: Index, Search & Polyglot

**Status**: 🟡 RECONCILED (current-state + target-state boundary)
**Epic**: lev-h038 (Index backends), lev-w5rx (Primary index)
**Last Updated**: 2026-02-13

---

## Scope

**What this covers:**
- Semantic search (LEANN)
- Code search (ck - code kernel)
- Polyglot search (mgrep, ripgrep)
- Index backends and daemon/process binding boundaries

**Modules:**
```
~/lev/core/
├── index/              # Search orchestration + find-daemon business logic
├── polyglot-runners/   # Poly binder + registry + process lifecycle
└── daemons/            # Legacy daemon implementations (migration target)
```

---

## LOCKED DECISIONS

### Q1: index/ naming ✅
**Decision**: Keep `index/` — "Find anything, anywhere, fast"
**SRP**: "Polyglot search orchestration" (3 words)

### Q2: daemon naming ✅
**Decision**: `polyglot-runners/` is the process supervisor; daemon business logic lives under `core/daemon/`.

**Current reconciliation target:**
- `core/polyglot-runners/` owns lifecycle (`start/stop/restart/health`, adapter to PMDaemon).
- `core/daemon/` is the runtime location for daemon applications/services.
- legacy daemon logic is migrated into owning modules/plugins with `poly` declarations.

### Q3: watch/ location ✅
**Decision**: NO separate watch/ module. File watching = Watcher pattern (reactive layer).
- FSWatcher emits LevEvents to events/ Bus
- Part of reactive layer, not separate module

### Q4: Polyglot search ✅
**Decision**: `polyglot-runners/` — Cross-language SDK with 3 search backends
- 144 tests passing
- 4 lang SDKs (TS, Python, Go, Rust)
- Production Go binary

---

## Architecture

### Three Search Engines

| Engine | Type | Purpose | Speed |
|--------|------|---------|-------|
| **LEANN** | Semantic | Embeddings-based understanding | ~500ms/10k docs |
| **ck** | Code | AST-aware code search | ~100ms/1M LOC |
| **mgrep** | Text | Ripgrep wrapper, regex | ~50ms/1M files |

### Index/Poly Integration Issues (Active Work)

1. **Registry build output mismatch**:
   `core/polyglot-runners/src/registry-builder.js` used a legacy shorthand `.build` target instead of the canonical `core/polyglot-runners/.build` path.
2. **Port collision debt in static registry**:
   `core/polyglot-runners/registry.yaml` defines overlapping ports (for example `9850` and `9852` used by multiple daemon entries).
3. **Index daemon bypass path**:
   `core/index/src/daemon/find-daemon.js` and `core/index/src/daemon/daemon-client.js` run with local daemon fallback behavior that is not fully normalized through module-level `poly.daemon` declarations.

### Daemon Management Pattern

```
Module declares lifecycle intent in config.yaml (poly.daemon / daemon)
    → registry builder merges declaration into runtime registry
    → poly process adapter (PMDaemon) starts and supervises process
    → index clients consume service via declared endpoint/protocol
```

---

## Embedding Acceleration Architecture

### SWIG vs faiss

| Term | What It Is |
|------|------------|
| **faiss** | Facebook AI Similarity Search — C++ library for vector similarity |
| **faiss-cpu** | Standard pip package (`pip install faiss-cpu`) — pre-compiled Python bindings |
| **SWIG** | C-to-Python binding generator — compiles custom C++ extensions into `.so` files |
| **LEANN custom faiss** | SWIG-compiled fork with CSR compression, recompute mode, compact HNSW |

**Fallback chain:** Custom SWIG → standard `faiss-cpu` → ImportError. Builder works with both; Searcher requires custom SWIG for CSR/recompute features.

### Capability Detection (capability-driven, not OS-driven)

Detection probes hardware features, not `platform.system()`:

```
detect_capabilities() → CapabilityProfile
  ├── torch.backends.mps.is_available()    → has_mps (Apple Metal)
  ├── torch.cuda.is_available()            → has_cuda (NVIDIA)
  ├── torch.version.hip is not None        → has_rocm (AMD)
  ├── shutil.which("vulkaninfo")           → has_vulkan (general GPU)
  └── requests.get("localhost:11434")      → has_ollama
```

### Auto-Selection Waterfall

| Priority | Capability | Mode | Model | Device | Speed |
|----------|-----------|------|-------|--------|-------|
| 1 | MPS | sentence-transformers | facebook/contriever | mps | ~900/s |
| 2 | CUDA | sentence-transformers | facebook/contriever | cuda | ~2000+/s |
| 3 | Ollama | ollama | nomic-embed-text | n/a | ~120/s |
| 4 | CPU | sentence-transformers | MiniLM-L6-v2 | cpu | ~30/s |

### Retrieval Quality Benchmarks (MTEB, English Retrieval)

| Model | Retrieval Average (NDCG@10) | Notes |
|-------|------------------------------|-------|
| nomic-embed-text-v1.5 | 53.01 | Highest quality of compared local/cloud options |
| text-embedding-3-small | 51.08 | Cloud API; close to nomic |
| all-MiniLM-L6-v2 | 41.95 | Small and fast, lower retrieval quality |
| contriever-base-msmarco | 41.88 | Similar retrieval score to MiniLM in this snapshot |

Source snapshot: MTEB leaderboard `boards_data/en/data_tasks/Retrieval/default.jsonl` (2025-08-31). `contriever-base-msmarco` is not identical to `facebook/contriever`; Lev-specific labeled retrieval testing should be used for final model policy decisions.

### Configuration

File: `~/.config/lev/index.yaml` — all values support `auto` for capability-driven resolution.

```yaml
embedding:
  mode: auto          # auto | sentence-transformers | ollama | openai | gemini | mlx
  model: auto         # auto | model name
  device: auto        # auto | mps | cuda | cpu
  batch_size: auto    # auto | integer
index:
  backend: hnsw       # hnsw | diskann
  faiss_mode: auto    # auto | custom-swig | standard
```

Resolution: explicit value → `auto` via CapabilityProfile → env var override → defaults.

### Implementation

| Module | Path | Purpose |
|--------|------|---------|
| capabilities.py | `core/index/python/embedder/capabilities.py` | Hardware detection |
| config.py | `core/index/python/embedder/config.py` | Config loader + auto-resolution |
| embedding_compute.py | `leann-core/src/leann/embedding_compute.py` | 5-mode compute (ST, Ollama, OpenAI, Gemini, MLX) |
| hnsw_backend.py | `leann-backend-hnsw/.../hnsw_backend.py` | SWIG → faiss-cpu fallback |

**Spec**: [spec-index-acceleration.md](file:///Users/jean-patricksmith/digital/leviathan/docs/specs/spec-index-acceleration.md)

---

## Integration

**Used by**:
- `lev find` (CLI command)
- FlowMind compiler (schema discovery)
- Agent harness (context loading)

**Epics**:
- lev-h038 — Index backends (ACTIVE)
- lev-h038.1 — Backend submodules
- lev-h038.2 — llm-tldr benchmarking POC
- lev-ggbk — Phase 2 index overhaul
- lev-w5rx — Primary index epic (72.7%)

---

## Status

✅ 144 tests passing in polyglot-runners
✅ Capability detection module (MPS/CUDA/ROCm/Vulkan/Ollama)
✅ Config-driven acceleration (`~/.config/lev/index.yaml`)
✅ HNSW faiss fallback (custom SWIG → standard faiss-cpu)
⚠️ Registry output path drift (legacy shorthand path vs canonical polyglot-runners path)
⚠️ Port collision cleanup needed in static registry
⚠️ Index daemon lifecycle still partially outside `poly` declaration flow
