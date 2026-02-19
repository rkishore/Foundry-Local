# Foundry Local — Claims Verification Report

**Date:** 2026-02-19
**Branch:** `claude/verify-foundry-local-claims-EOkiw`
**Sources:** Codebase analysis (`sdk_v2/cs/`, `sdk_v2/js/`, `sdk/`) + Microsoft Learn documentation

---

## Summary Table

| Claim | Verdict | Confidence |
|---|---|---|
| 1.1 Sequential inference, not designed for concurrent serving | **Confirmed** | High |
| 1.2 No dynamic or continuous batching | **Confirmed** | High |
| 1.3 No health/readiness endpoints or serving primitives | **Largely confirmed** | High |
| 1.4 No OpenTelemetry metrics or traces | **Confirmed** | High |
| 1.5 Docs direct teams to Azure Foundry for multi-user/spiky traffic | **Confirmed** | High |
| 1.6 Self-contained C# SDK, no Python, NVIDIA/AMD/Intel/Qualcomm/Apple Silicon support | **Partially accurate** | High |
| 1.7 Windows Server 2025 supported; "not optimized for multi-user serving" | **Confirmed** | High |

---

## Claim 1.1 — Sequential Inference (No Concurrent Multi-User Serving)

**Verdict: CONFIRMED**

### Documentation Evidence

The [Windows Server 2025 FAQ](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/reference/windows-server-frequently-asked-questions?view=foundry-classic) states verbatim:

> "Foundry Local isn't optimized to serve multiple users as a shared on-premises endpoint and **doesn't support concurrent inference requests. It processes requests sequentially.** As concurrent requests increase, throughput drops and latency increases."

### Codebase Evidence

`sdk_v2/cs/src/Detail/AsyncLock.cs` implements a `SemaphoreSlim(1, 1)` — a strict single-entry mutex — used throughout `Catalog.cs` and `FoundryLocalManager.cs` to serialize all async operations. There is no thread-pool dispatch, request fan-out, or parallel execution path anywhere in the SDK.

```csharp
// AsyncLock.cs — single-entry semaphore used for all serialization
private readonly SemaphoreSlim _semaphore = new SemaphoreSlim(1, 1);
```

### Caveat

The actual sequential scheduling lives inside the closed-source `Microsoft.AI.Foundry.Local.Core` native library. The SDK-level evidence is circumstantial but consistent with the documented claim. A live concurrent-request throughput test (not run here — requires Foundry CLI) would provide empirical confirmation.

---

## Claim 1.2 — No Dynamic or Continuous Batching

**Verdict: CONFIRMED**

### Documentation Evidence

The same [Windows Server 2025 FAQ](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/reference/windows-server-frequently-asked-questions?view=foundry-classic) states:

> "There's **no continuous batching** in the Local runtime, so request coalescing doesn't happen under load."

### Codebase Evidence

Zero references to "batch", "coalesce", or any multi-request aggregation logic exist in `sdk_v2/cs/src/` or `sdk_v2/js/src/`. Each inference request goes through a single serial async call path with no queuing layer.

### Caveat

Runtime batching behavior inside `Microsoft.AI.Foundry.Local.Core` cannot be observed from the SDK alone. Profiling ONNX session calls (e.g., with WinPerf or dotnet-trace) would provide definitive confirmation.

---

## Claim 1.3 — No Health/Readiness Endpoints or Operational Serving Primitives

**Verdict: LARGELY CONFIRMED**

### Codebase Evidence

After searching all files in `sdk_v2/cs/src/` and `sdk/`:

| Symbol searched | Found in SDK? |
|---|---|
| `IHealthCheck` | No |
| Readiness / liveness probe | No |
| Health HTTP endpoint | No |
| Drain API | No |

### What Does Exist (partial primitives)

| API | File | Notes |
|---|---|---|
| `StopWebServiceAsync()` | `FoundryLocalManager.cs:131–135` | Graceful stop of the embedded HTTP service |
| `Dispose()` / `DisposeAsync()` | `FoundryLocalManager.cs:275–308` | Cleanup on teardown; disposes catalog, lock, model manager |
| `IsLoadedAsync()` / `IsCachedAsync()` | `IModel.cs:19–20` | Model-state queries, not a health endpoint |
| `CancellationToken` | All public APIs | Cancellation support, not readiness |

None of these constitute production serving primitives in the sense required for Kubernetes readiness/liveness probes or load-balancer drain integration.

---

## Claim 1.4 — No OpenTelemetry Metrics or Traces

**Verdict: CONFIRMED**

### Codebase Evidence

Full search of `sdk_v2/cs/src/` and `sdk_v2/js/src/`:

| OTel symbol | Found in SDK source? |
|---|---|
| `ActivitySource` | No |
| `System.Diagnostics.Metrics` / `Meter` | No (README example only, not in implementation) |
| `OpenTelemetry` | No |
| `DiagnosticSource` | No |
| Any named OTel meter or activity | No |

### NuGet Dependencies (`Microsoft.AI.Foundry.Local.csproj`)

```xml
<PackageReference Include="Microsoft.AI.Foundry.Local.Core" ... />
<PackageReference Include="Betalgo.Ranul.OpenAI" Version="9.1.0" />
<PackageReference Include="Microsoft.Extensions.Logging" Version="9.0.9" />
```

No OpenTelemetry packages. Logging uses only `Microsoft.Extensions.Logging`.

The `appName` field in `sdk_v2/js/src/configuration.ts` is documented as *"used for identifying the application in logs and telemetry"* — this is metadata passed to the native runtime, not SDK-emitted OTel instrumentation.

---

## Claim 1.5 — Microsoft Docs Direct Teams to Azure Foundry for Multi-User / Spiky Traffic

**Verdict: CONFIRMED**

### Documentation Evidence

[Windows Server 2025 FAQ](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/reference/windows-server-frequently-asked-questions?view=foundry-classic):

> "For **multiple users or spiky traffic**, Microsoft recommends moving to **Microsoft Foundry**."

The [Best Practices guide](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/reference/reference-best-practice?view=foundry-classic) and the [What is Foundry Local?](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/what-is-foundry-local?view=foundry-classic) overview page both frame Foundry Local as an on-device/edge tool, reinforcing the same positioning.

### Caveat

The product is currently in **public preview**. These statements could be revised at GA.

---

## Claim 1.6 — Self-Contained C# SDK, No Python, Supports NVIDIA / AMD / Intel / Qualcomm / Apple Silicon

**Verdict: PARTIALLY ACCURATE**

### Sub-claim: C# SDK v2 is self-contained — CORRECT

`sdk_v2/cs/` has no dependency on the Foundry CLI. End-users do not need any external binary installed. The ONNX Runtime is bundled via the `Microsoft.AI.Foundry.Local.Core` NuGet package.

### Sub-claim: "No Python" — INCORRECT

A full Python SDK exists at `sdk/python/` (PyPI: `foundry-local-sdk`). It covers SDK v1. The statement is only true when scoped specifically to SDK v2.

### Sub-claim: Hardware support — CORRECT (with driver caveats)

| Vendor | Execution Provider | Codebase Evidence | Docs |
|---|---|---|---|
| NVIDIA | `CUDAExecutionProvider`, `NvTensorRTRTXExecutionProvider` | `Utils.cs:340` | Yes |
| Qualcomm | `QNNExecutionProvider` | `Utils.cs:272,293` | Yes |
| Intel | `OpenVINOExecutionProvider` | Docs only | Yes — requires Intel NPU driver |
| AMD | `VitisAIExecutionProvider` | Docs only | Yes |
| Apple Silicon | `WebGpuExecutionProvider` (Dawn → Metal) | `csproj:17` (`osx-arm64`) | Yes |
| CPU | `CPUExecutionProvider` | `Utils.cs:227,248` | Yes |

**Important caveats:**
- Intel NPU requires installing the **Intel NPU driver** separately — not fully zero-external-dependency on that target.
- Apple Silicon uses WebGPU via Dawn/Metal, not a dedicated `CoreMLExecutionProvider`.
- AMD and Intel execution providers appear in documentation but are not exercised in the repo's own test data; only NVIDIA, Qualcomm, WebGPU, and CPU are covered in `Utils.cs`.
- The WinML variant (`Microsoft.AI.Foundry.Local.WinML`) is Windows-only (`win-x64;win-arm64`) and targets `net8.0-windows10.0.26100.0`.

### Runtime Identifiers (`Microsoft.AI.Foundry.Local.csproj:17`)

```xml
<RuntimeIdentifiers>win-x64;win-arm64;linux-x64;linux-arm64;osx-arm64</RuntimeIdentifiers>
```

---

## Claim 1.7 — Windows Server 2025 Supported; "Not Optimized for Multi-User Serving"

**Verdict: CONFIRMED**

### Documentation Evidence

Microsoft maintains a dedicated page titled *"Foundry Local on Windows Server 2025 — Frequently Asked Questions"* ([link](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/reference/windows-server-frequently-asked-questions?view=foundry-classic)), confirming Windows Server 2025 is a documented and supported target.

The exact quote on that page:

> "Foundry Local isn't optimized to serve multiple users as a shared on-premises endpoint and doesn't support concurrent inference requests."

### Codebase Evidence

- `.csproj` targets `net8.0` with `win-x64;win-arm64` runtime identifiers — which covers Windows Server 2025.
- WinML variant targets `net8.0-windows10.0.26100.0` (Windows 10 build 26100 = Windows Server 2025 kernel).
- No server-specific features or server-optimized code paths exist in the SDK.

### Nuance

The FAQ page exists because there is real interest in deploying Foundry Local on Windows Server 2025 (e.g., as a shared internal endpoint). Microsoft explicitly acknowledges the use case while stating its limitations, rather than blocking the scenario outright.

---

## Methodology

| Method | Applied to |
|---|---|
| Source code search (Grep/Glob) | All claims |
| File reading (`sdk_v2/cs/src/`, `sdk_v2/js/src/`, `sdk/python/`) | Claims 1.3, 1.4, 1.6 |
| Microsoft Learn documentation search | Claims 1.1, 1.2, 1.5, 1.7 |
| NuGet `.csproj` dependency analysis | Claims 1.4, 1.6 |
| Live concurrency testing | **Not performed** — requires Foundry CLI (not installed) |
| ONNX session profiling | **Not performed** — requires live runtime |

---

## References

- [Foundry Local on Windows Server 2025 FAQ](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/reference/windows-server-frequently-asked-questions?view=foundry-classic)
- [Foundry Local Best Practices Guide](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/reference/reference-best-practice?view=foundry-classic)
- [What is Foundry Local?](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/what-is-foundry-local?view=foundry-classic)
- [Get Started with Foundry Local](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/get-started?view=foundry-classic)
- [Apple Silicon support discussion](https://github.com/microsoft/Foundry-Local/discussions/156)
- `sdk_v2/cs/src/Microsoft.AI.Foundry.Local.csproj` — NuGet dependencies, runtime identifiers
- `sdk_v2/cs/src/Detail/AsyncLock.cs` — `SemaphoreSlim(1,1)` concurrency primitive
- `sdk_v2/cs/src/IModel.cs` — Public model state API surface
- `sdk_v2/cs/test/FoundryLocal.Tests/Utils.cs` — Execution provider test data
