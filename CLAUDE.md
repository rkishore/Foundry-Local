# CLAUDE.md — Foundry Local Codebase Guide

This document provides a comprehensive guide for AI assistants working in the Foundry Local repository. It covers structure, development workflows, build commands, testing, and code conventions.

## Project Overview

**Foundry Local** is a Microsoft open-source project that brings Azure AI Foundry capabilities to local devices — no Azure subscription required. It enables on-device inference via an OpenAI-compatible REST API, backed by ONNX Runtime and hardware acceleration.

Key capabilities:
- Run generative AI models locally (Windows/macOS)
- OpenAI-compatible API surface
- SDK support for JavaScript/TypeScript, Python, C#, and Rust
- Hardware-aware model variant selection (CUDA GPU, Qualcomm NPU, CPU)

**License:** Microsoft Software License Terms (see `LICENSE`)
**Contributing:** Requires a Contributor License Agreement (CLA) — see `CONTRIBUTING.md`

---

## Repository Structure

```
Foundry-Local/
├── .github/workflows/       # GitHub Actions CI/CD
├── docs/                    # API reference docs + integration guides
│   └── cs-api/              # Generated C# API markdown (19 files)
├── licenses/                # Third-party model licenses (Mistral, DeepSeek, Phi)
├── media/icons/             # SVG branding assets
├── samples/                 # Sample applications
│   ├── cs/                  # C# samples (GettingStarted)
│   ├── electron/            # Electron chat app
│   ├── js/                  # JavaScript samples
│   ├── python/              # Python samples (hello, summarize, function calling)
│   ├── rag/                 # RAG (Retrieval-Augmented Generation) sample
│   └── rust/                # Rust sample
├── sdk/                     # SDK v1 — original SDK
│   ├── cs/                  # C# SDK v1
│   ├── js/                  # JavaScript/TypeScript SDK v1 (npm: foundry-local-sdk)
│   ├── python/              # Python SDK v1 (PyPI: foundry-local-sdk)
│   └── rust/                # Rust SDK v1
├── sdk_v2/                  # SDK v2 — next-generation SDK
│   ├── cs/                  # C# SDK v2 (self-contained, no CLI dependency)
│   └── js/                  # JavaScript/TypeScript SDK v2
└── www/                     # Official website (SvelteKit + Vercel)
```

---

## SDK Versions

There are **two SDK generations** in this repo:

| Feature | SDK v1 (`sdk/`) | SDK v2 (`sdk_v2/`) |
|---|---|---|
| Languages | JS, Python, C#, Rust | JS, C# |
| CLI dependency | Required (Foundry CLI must be in PATH) | C# is self-contained; JS uses native bindings |
| JS test runner | Vitest | Mocha |
| JS build tool | unbuild | tsc |
| C# platform | .NET 9.0 | .NET 9.0 |
| Native bindings | No | Yes (koffi for JS) |
| Chat/Audio APIs | Via OpenAI HTTP | In-process (C# SDK v2) |

---

## Development Workflows

### JavaScript/TypeScript — SDK v1 (`sdk/js/`)

```bash
cd sdk/js
npm install

npm run build           # Build (uses unbuild, also auto-generates version.ts)
npm run lint            # ESLint on src/
npm run format          # Prettier (write)
npm run format:check    # Prettier (check only)
npm test                # Unit tests (Vitest)
npm run integration-test  # Integration tests (requires Foundry CLI running)
```

**Build outputs:** `dist/index.cjs`, `dist/index.mjs`, `dist/index.d.ts`
**Browser export:** `dist/base.cjs`, `dist/base.mjs`, `dist/base.d.ts`

### JavaScript/TypeScript — SDK v2 (`sdk_v2/js/`)

```bash
cd sdk_v2/js
npm install

npm run build           # TypeScript compile (tsc -p tsconfig.build.json)
npm test                # Mocha tests (tsx loader)
npm run docs            # Generate TypeDoc documentation
npm run example         # Run example chat-completion script
```

### Python — SDK v1 (`sdk/python/`)

```bash
cd sdk/python
pip install -e ".[dev]"   # Install in editable mode with dev dependencies

python -m pytest test/                # Unit tests
python -m pytest integration_test/   # Integration tests (requires Foundry CLI)

ruff check .            # Lint
ruff format .           # Format
```

**Python version support:** 3.9, 3.10, 3.11, 3.12, 3.13
**Key runtime dependencies:** `httpx`, `pydantic>=2.0.0`, `tqdm`

### C# — SDK v1 (`sdk/cs/`)

```bash
cd sdk/cs
dotnet build
dotnet test
dotnet pack
```

### C# — SDK v2 (`sdk_v2/cs/`)

```bash
cd sdk_v2/cs
dotnet restore
dotnet build Microsoft.AI.Foundry.Local.SDK.sln
dotnet test
dotnet pack --configuration Release -p:PackageVersion=<version>
```

**Note:** SDK v2 C# is self-contained — end users do NOT need the Foundry Local CLI installed.

### Rust — SDK v1 (`sdk/rust/`)

```bash
cd sdk/rust
cargo build
cargo test
cargo test --features integration-tests   # Integration tests
cargo fmt --check                          # Format check (enforced by CI)
cargo fmt                                  # Auto-format
```

### Website (`www/`)

```bash
cd www
npm install    # or: bun install

npm run dev    # Dev server (Vite)
npm run build  # Production build (SvelteKit + Vercel adapter)
npm run preview  # Preview production build
```

---

## Testing

### Unit Tests

| Component | Command | Test Runner |
|---|---|---|
| JS SDK v1 | `npm test` in `sdk/js/` | Vitest |
| JS SDK v2 | `npm test` in `sdk_v2/js/` | Mocha + tsx |
| Python SDK | `python -m pytest test/` in `sdk/python/` | pytest |
| C# SDK v2 | `dotnet test` in `sdk_v2/cs/` | dotnet test |
| Rust SDK | `cargo test` in `sdk/rust/` | cargo test |

### Integration Tests

Integration tests require the **Foundry Local CLI** to be installed and running.

| Component | Command |
|---|---|
| JS SDK v1 | `npm run integration-test` in `sdk/js/` |
| Python SDK | `python -m pytest integration_test/` in `sdk/python/` |
| Rust SDK | `cargo test --features integration-tests` in `sdk/rust/` |

### Test File Locations

- `sdk/js/test/` — JS v1 unit tests (`client.test.ts`, `base.test.ts`, `service.test.ts`, `index.test.ts`)
- `sdk/js/integration_test/` — JS v1 integration tests
- `sdk_v2/js/test/` — JS v2 tests (`model.test.ts`, `catalog.test.ts`, `foundryLocalManager.test.ts`, `chatClient.test.ts`, `audioClient.test.ts`)
- `sdk/python/test/` — Python unit tests (`test_api.py`, `test_client.py`, `test_service.py`, `test_models.py`)
- `sdk/python/integration_test/` — Python integration test (`test_integration.py`)
- `sdk_v2/cs/test/FoundryLocal.Tests/` — C# v2 tests
- `sdk/rust/tests/` — Rust integration tests (`integration_tests.rs`, `test_api.rs`)

---

## CI/CD Pipelines

All workflows are in `.github/workflows/`:

| Workflow | Trigger | What it does |
|---|---|---|
| `foundry-local-sdk-build.yml` | PRs/pushes to `sdk_v2/**`, manual | Builds & tests C# and JS SDK v2 on Windows + macOS (including WinML variant) |
| `build-js-steps.yml` | Reusable workflow | Installs Node 20.x, runs `npm test`, `npm run build`, uploads tarball artifacts |
| `build-cs-steps.yml` | Reusable workflow | Sets up .NET 9.0, restores NuGet, builds, tests, packs NuGet package |
| `rustfmt.yml` | PRs/pushes | Validates Rust code formatting with `cargo fmt` |

**Version scheme:** SDK v2 uses `0.9.0.<run_number>` (auto-incremented by CI).

**CI test data:** JavaScript CI fetches test data (including `Recording.mp3`) from an Azure DevOps repo using a PAT secret (`AZURE_DEVOPS_PAT`).

---

## Code Conventions

### All Languages

- Every source file must begin with the Microsoft copyright header:
  ```
  // Copyright (c) Microsoft Corporation. All rights reserved.
  // Licensed under the MIT License.
  ```
  This is enforced by ESLint (`eslint-plugin-header`) in JS/TS code.

### JavaScript/TypeScript

- **Formatter:** Prettier — tabs (not spaces), single quotes, 100-character line width
- **Linter:** ESLint with `@typescript-eslint` and `eslint-plugin-import`
- **Module format:** SDK v1 exports both CJS and ESM; SDK v2 is ESM-only (`"type": "module"`)
- **Target:** SDK v1 targets ES6; SDK v2 targets ES2022
- **Strict mode:** TypeScript strict mode enabled
- No relative imports across package boundaries

### Python

- **Formatter/Linter:** Ruff (replaces Black + isort + flake8)
- **Line length:** 120 characters
- **Target version:** Python 3.9+
- **Docstring convention:** Google style
- **Imports:** Absolute imports only — relative imports are banned (`ban-relative-imports = "all"`)
- **Rules enabled:** bugbear, comprehensions, pydocstyle, pyflakes, isort, logging, naming, numpy, perflint, etc. (see `pyproject.toml` for full list)

### C#

- **.NET version:** 9.0
- **Style:** Follows `.editorconfig` rules in the repo root
- **Packages:** Published to NuGet; internal packages use `Microsoft.AI.Foundry.Local` namespace

### Rust

- **Edition:** 2021
- **Formatting:** `rustfmt` is enforced by CI (`rustfmt.yml`)
- **Async runtime:** Tokio with full features
- **HTTP client:** Reqwest with `rustls-tls` (no native TLS dependency)
- **Serialization:** Serde + serde_json
- Integration tests are gated behind the `integration-tests` feature flag

---

## Key Source Files

### JavaScript SDK v1 (`sdk/js/src/`)

| File | Purpose |
|---|---|
| `index.ts` | Main entry point; exports `FoundryLocalManager` (Node.js) |
| `base.ts` | `FoundryLocalManagerBase` — browser-compatible core logic |
| `service.ts` | CLI interaction: detecting, starting the Foundry service |
| `types.ts` | Shared TypeScript types (`DeviceType`, `FoundryModelInfo`, etc.) |

### JavaScript SDK v2 (`sdk_v2/js/src/`)

| File | Purpose |
|---|---|
| `index.ts` | Barrel export for all public API |
| `foundryLocalManager.ts` | Top-level manager (singleton pattern) |
| `catalog.ts` | Model catalog browsing |
| `model.ts` | Model instance (download, load, unload) |
| `modelVariant.ts` | Hardware-specific model variant |
| `configuration.ts` | SDK configuration types |
| `openai/chatClient.ts` | In-process chat completions client |
| `openai/audioClient.ts` | In-process audio transcription client |
| `detail/coreInterop.ts` | Native library interop (koffi) — internal |
| `detail/modelLoadManager.ts` | Low-level model load/unload logic — internal |

### Python SDK (`sdk/python/foundry_local/`)

| File | Purpose |
|---|---|
| `__init__.py` | Package entry point and public API |
| `version.py` | Package version (referenced by `pyproject.toml`) |

---

## Environment & Prerequisites

### For SDK v1 (JS, Python, Rust)

- Foundry Local CLI must be installed and in PATH
- Install via:
  - **Windows:** `winget install Microsoft.FoundryLocal`
  - **macOS:** `brew install microsoft/foundrylocal/foundrylocal`

### For SDK v2 C#

- No external dependencies — the SDK is fully self-contained

### For SDK v2 JS

- Uses native platform bindings via `koffi`
- Platform packages (`@foundry-local-core/<platform>`) are optional dependencies resolved at install time

### For Website (`www/`)

- Node.js (version specified in `www/.nvmrc`)
- Optional: `VERCEL_ANALYTICS_ID`, `PUBLIC_FOUNDRY_API_ENDPOINT`, `PUBLIC_BASE_URL` env vars (see `www/.env.example`)

---

## Website Architecture (`www/`)

Built with **SvelteKit 2.x** and deployed to **Vercel**.

```
www/src/
├── app.html              # HTML shell template
├── routes/
│   ├── +layout.svelte    # Root layout (nav, footer)
│   ├── +page.svelte      # Homepage
│   ├── docs/             # Documentation pages (mdsvex-powered markdown)
│   └── models/           # Models catalog page
└── lib/
    ├── components/       # 13+ UI components (bits-ui, shadcn-svelte, lucide-svelte)
    └── ...
```

Key config files:
- `svelte.config.js` — SvelteKit + Vercel adapter + mdsvex preprocessing
- `vite.config.ts` — Vite with vendor chunk splitting
- `tailwind.config.ts` — Custom Tailwind theme with brand colors
- `www/static/` — Static assets (logos, favicons, `manifest.json`)

---

## Samples

Located in `samples/`, each sub-directory contains a standalone project:

| Directory | Description |
|---|---|
| `samples/js/hello-foundry-local/` | Basic JS quickstart |
| `samples/js/copilot-sdk-foundry-local/` | GitHub Copilot SDK integration |
| `samples/python/hello/` | Basic Python quickstart |
| `samples/python/summarize/` | Text summarization sample |
| `samples/python/function-calling/` | Function calling sample |
| `samples/rust/hello-foundry-local/` | Basic Rust quickstart |
| `samples/cs/GettingStarted/` | C# getting-started sample |
| `samples/electron/foundry-chat/` | Electron-based chat UI |
| `samples/rag/` | Retrieval-Augmented Generation sample |

---

## Common Pitfalls

1. **SDK v1 vs SDK v2:** The two SDK versions coexist in the repo. Make sure you are editing the correct one — `sdk/` vs `sdk_v2/`.

2. **CLI dependency:** SDK v1 requires the Foundry Local CLI in PATH. Tests will fail without it installed. Integration tests are separate from unit tests and skip-able.

3. **Version file:** JS SDK v1 auto-generates `src/version.ts` during `npm run build` (via `genversion`). Do not edit this file manually.

4. **Native bindings in SDK v2 JS:** The `koffi` dependency requires native compilation. Ensure platform-specific optional packages are present.

5. **Rust feature flags:** Rust integration tests only run with `--features integration-tests`. Do not add integration-test logic to regular tests.

6. **Rust formatting:** `cargo fmt` is enforced by CI. Always run `cargo fmt` before committing Rust code.

7. **Python relative imports:** All relative imports are banned by Ruff. Use absolute imports everywhere in the Python SDK.

8. **Copyright headers:** All JS/TS source files must include the Microsoft copyright header. ESLint enforces this via `eslint-plugin-header`.
