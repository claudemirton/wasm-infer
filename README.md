# wasm-infer

> A portable CLI for running small Hugging Face models, exported to **ONNX**, inside a sandboxed **WebAssembly (WASI)** runtime. No Python, no CUDA, no native dependencies.

![status](https://img.shields.io/badge/status-planning-blue)
![license](https://img.shields.io/badge/license-MIT-green)
![language](https://img.shields.io/badge/built%20with-Rust-orange)

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Quick Start](#quick-start)
- [CLI Reference](#cli-reference)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Supported Models and Tasks](#supported-models-and-tasks)
- [Security Model](#security-model)
- [Timeline and Milestones](#timeline-and-milestones)
- [Success Metrics](#success-metrics)
- [Risks and Mitigations](#risks-and-mitigations)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

Running even a small model locally usually means installing Python, PyTorch, and the right native libraries. Deploying to edge devices, CI pipelines, or restricted environments is fragile, and executing untrusted model code is hard to sandbox.

`wasm-infer` addresses this by combining three open standards:

| Technology | Role |
|---|---|
| **Hugging Face Hub** | Source of small, task-specific models and tokenizers |
| **ONNX** | Portable model format (export, validation, quantization) |
| **WebAssembly / WASI** | Sandboxed, cross-platform execution environment |

The result is a single binary that pulls, converts, and runs models identically on macOS, Linux, and Windows.

## Features

- **One-command model management:** pull models from the Hugging Face Hub and cache them locally.
- **ONNX conversion pipeline:** export, validate opset and shapes, and quantize (int8/fp16).
- **Sandboxed inference:** runs inside a Wasm runtime with capability-based permissions.
- **Numerical parity checks:** compare ONNX output against the PyTorch reference.
- **Built-in benchmarking:** latency, memory, and binary size reports.
- **Optional local HTTP API:** serve a model with `wasm-infer serve`.
- **Zero Python dependency at runtime.**

## Quick Start

> Planned interface. Commands marked with a milestone are not implemented yet.

```bash
# Install (planned: prebuilt binaries + cargo)
cargo install wasm-infer

# 1. Fetch a model from the Hugging Face Hub
wasm-infer pull sentence-transformers/all-MiniLM-L6-v2

# 2. Convert to ONNX with int8 quantization (M2)
wasm-infer convert sentence-transformers/all-MiniLM-L6-v2 --opset 17 --quantize int8

# 3. Run inference inside the Wasm sandbox
wasm-infer run sentence-transformers/all-MiniLM-L6-v2 \
  --task embeddings \
  --input "WebAssembly makes inference portable."

# 4. Benchmark (M4)
wasm-infer bench sentence-transformers/all-MiniLM-L6-v2

# 5. Serve a local HTTP API (post-MVP)
wasm-infer serve sentence-transformers/all-MiniLM-L6-v2 --port 8080
```

## CLI Reference

| Command | Description | Milestone |
|---|---|---|
| `pull <model>` | Download model and tokenizer files into the local cache | M1 |
| `list` | List cached models and their ONNX variants | M1 |
| `convert <model>` | Export to ONNX, validate, and optionally quantize | M2 |
| `run <model>` | Execute inference in the Wasm sandbox | M3 |
| `verify <model>` | Check numerical parity against the reference model | M4 |
| `bench <model>` | Measure latency, throughput, and memory | M4 |
| `serve <model>` | Expose a local HTTP inference endpoint | Post-MVP |

Common flags: `--task`, `--input`, `--file`, `--quantize`, `--opset`, `--backend`, `--max-memory`, `--timeout`, `--json`.

## Architecture

```text
┌──────────────────────────────────────────────────────────┐
│                         CLI (Rust)                       │
│   arg parsing · config · output formatting · caching     │
└───────┬───────────────┬──────────────────┬───────────────┘
        │               │                  │
┌───────▼──────┐ ┌──────▼───────┐ ┌────────▼─────────────┐
│  Hub Client  │ │ ONNX Toolkit │ │   Wasm Host Runtime  │
│ pull / cache │ │ export       │ │  Wasmtime (WASI)     │
│ HF Hub API   │ │ validate     │ │  capability policy   │
└──────────────┘ │ quantize     │ │  memory/time limits  │
                 └──────┬───────┘ └────────┬─────────────┘
                        │                  │
                        │        ┌─────────▼──────────────┐
                        │        │  Inference Module(.wasm)│
                        │        │  tokenizers + ONNX      │
                        └───────►│  WASI-NN backend or     │
                                 │  ONNX Runtime (Wasm+SIMD)│
                                 └─────────────────────────┘
```

### Components

| Module | Responsibility |
|---|---|
| **`cli`** | Command definitions, argument parsing, logging, progress output |
| **`hub`** | Hugging Face Hub client, authentication, download and cache management |
| **`onnx`** | Orchestrates Optimum export, graph validation, quantization, and parity checks |
| **`runtime`** | Embeds the Wasm runtime, enforces permissions and resource limits |
| **`backend`** | Abstraction over WASI-NN (ONNX) and ONNX Runtime compiled to Wasm |
| **`tasks`** | Task pipelines: pre-processing, inference, post-processing |
| **`tokenizer`** | Hugging Face `tokenizers` compiled to Wasm so text handling stays sandboxed |
| **`bench`** | Benchmark harness and report generation |
| **`serve`** | Optional local HTTP server (post-MVP) |

## Repository Structure

```text
wasm-infer/
├── Cargo.toml                  # Workspace manifest
├── README.md
├── LICENSE
├── crates/
│   ├── cli/                    # Binary entry point and commands
│   ├── hub/                    # Hugging Face Hub client + cache
│   ├── onnx/                   # Export, validate, quantize, parity
│   ├── runtime/                # Wasm host, WASI config, limits
│   ├── backend/                # WASI-NN / ORT-Wasm backend abstraction
│   ├── tasks/                  # Embeddings, classification, NER pipelines
│   └── bench/                  # Benchmark harness
├── wasm/
│   └── inference-module/       # Guest module compiled to wasm32-wasi
├── scripts/
│   └── export/                 # Python helper for Optimum export (build-time only)
├── models/
│   └── registry.toml           # Verified model list and settings
├── tests/
│   ├── parity/                 # Reference vs ONNX output tests
│   ├── integration/            # End-to-end CLI tests
│   └── fixtures/
├── benchmarks/                 # Results and methodology
├── docs/                       # Guides, architecture, model support
└── .github/
    └── workflows/              # CI: build, test, release (3 OS matrix)
```

> Export needs Python and Optimum once, at conversion time. Inference never requires Python. Pre-converted ONNX artifacts can be pulled directly to skip this step.

## Supported Models and Tasks

**MVP tasks**

| Task | Example model |
|---|---|
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` |
| Text classification | `distilbert-base-uncased-finetuned-sst-2-english` |
| Token classification (NER) | `dslim/bert-base-NER` (or a distilled variant) |

The verified list lives in [`models/registry.toml`](models/registry.toml). Target: 5–8 models in the MVP.

**Planned:** small quantized generative models (sub-1B parameters) with streaming output.

## Security Model

- **Deny by default:** no network or filesystem access inside the sandbox unless explicitly granted.
- **Preopened directories only:** the guest sees just the model cache directory, read-only.
- **Resource limits:** configurable memory cap and execution timeout.
- **Integrity:** model files are checksummed on download and on load.
- **No arbitrary code execution:** only ONNX graphs and the project's own signed inference module are executed.

## Timeline and Milestones

Estimated duration for the MVP: **6 weeks**, working part-time and remote.

| Milestone | Target | Deliverables |
|---|---|---|
| **M0: Foundation** | Week 1 | Repo and workspace setup, CI (macOS/Linux/Windows), CLI skeleton, README and docs scaffold |
| **M1: Model management** | Week 2 | `pull` and `list`, Hub client, local cache, checksum verification |
| **M2: ONNX pipeline** | Week 3 | `convert` with export, opset validation, int8 quantization; first 3 models converted |
| **M3: Wasm inference** | Weeks 4–5 | Wasmtime host, guest inference module, `run` for embeddings and classification, NER pipeline, sandbox limits |
| **M4: Quality and benchmarks** | Week 6 | `verify` parity suite, `bench` command, cross-OS benchmark report, 5–8 verified models |
| **M5: MVP release** | End of week 6 | v0.1.0, prebuilt binaries, quickstart docs, public announcement |

### Post-MVP roadmap

| Phase | Scope |
|---|---|
| **v0.2** | `serve` HTTP API, model pack format (`.wasm` + weights in a single file) |
| **v0.3** | Small generative models with streaming output |
| **v0.4** | GPU backends (WebGPU / WASI-NN GPU) |
| **v0.5** | Browser and edge targets (Cloudflare Workers, Fermyon Spin) |

## Success Metrics

| Metric | Target |
|---|---|
| Cold start (small model) | < 1 second |
| Numerical parity with reference | ≥ 95% of tested models within tolerance |
| Install | Single command, no Python required |
| Platform support | Identical behavior on macOS, Linux, and Windows |
| Test coverage | Parity and integration tests in CI for every supported model |

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| WASI-NN ONNX support differs between runtimes | Inconsistent behavior | Backend abstraction plus ONNX Runtime-on-Wasm fallback |
| Wasm inference slower than native | Poor UX | SIMD, quantization, honest published benchmarks |
| Some ops fail to export to ONNX | Model not supported | Verified-models registry and clear error messages |
| Large model download size | Slow first run | Caching, quantized variants, resumable downloads |
| Scope creep | Missed MVP date | Fixed MVP task list, post-MVP items tracked separately |

## Development

### Prerequisites

- Rust (stable) with the `wasm32-wasip1` target
- Python 3.10+ (only for `convert`, via Hugging Face Optimum)

### Build and test

```bash
rustup target add wasm32-wasip1

# Build the guest module, then the CLI
cargo build --release -p inference-module --target wasm32-wasip1
cargo build --release

# Run tests
cargo test --workspace
```

### Development workflow

This project is developed with an AI coding assistant (**Claude Code**) for scaffolding, debugging, and test writing. All generated code is reviewed and tested by the maintainer before merging.

## Contributing

Contributions are welcome once the MVP is released.

1. Open an issue to discuss the change.
2. Fork the repo and create a feature branch.
3. Add tests, including a parity test for any new model.
4. Open a pull request.

## License

Released under the [MIT License](LICENSE).
