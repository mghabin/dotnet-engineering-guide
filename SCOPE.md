# Scope — who this guide is for, and who it is not for

This guide is opinionated.
The opinions only hold inside a specific technical and organisational envelope.
This page names that envelope so readers can decide, in 60 seconds, whether the rest of the guide will help them or mislead them.

If your situation falls outside this envelope, individual chapters may still be useful as background — but the *defaults* will be wrong for you, and you should treat the recommendations as input, not verdict.

---

## 1. Who it's for

Concrete reader profiles.
If two or more apply, you are the target reader.

- A senior or staff backend .NET engineer owning one or more long-lived services on .NET 10 / C# 13.
- A solutions or application architect choosing the default project, hosting, and data shape for a new bounded context.
- A tech lead inheriting a .NET estate and asked to "make it boring" before scaling the team.
- A platform engineer standardising `Directory.Build.props`, Central Package Management, analyzers, and CI templates across many .NET repos ([learn.microsoft.com/nuget/consume-packages/central-package-management](https://learn.microsoft.com/nuget/consume-packages/central-package-management)).
- An engineer wiring a new ASP.NET Core service to AKS, App Service, or Azure Container Apps and picking between Minimal APIs and Controllers ([learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview)).
- An engineer adopting .NET Aspire 9.x for local orchestration, service discovery, and OTel defaults ([learn.microsoft.com/dotnet/aspire/whats-new/dotnet-aspire-9](https://learn.microsoft.com/dotnet/aspire/whats-new/dotnet-aspire-9)).
- An SRE or perf-focused engineer using BenchmarkDotNet, dotnet-counters, and PerfView to investigate allocations, GC, or NativeAOT trade-offs ([devblogs.microsoft.com/dotnet/performance-improvements-in-net-9](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-9)).

---

## 2. Organisational and technical shape it assumes

The defaults assume a *cloud-targeting backend .NET shop* with multiple teams sharing a small number of repos.
Specifically:

- **.NET 10 GA on the current LTS / STS cadence** ([learn.microsoft.com/dotnet/core/releases-and-support](https://learn.microsoft.com/dotnet/core/releases-and-support)).
  Older TFMs are mentioned only when guidance differs.
- **C# 13 language defaults**, NRT enabled, `TreatWarningsAsErrors=true`, analyzers on by default ([learn.microsoft.com/dotnet/csharp/whats-new/csharp-13](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-13)).
- **Cloud as the deployment target.** AKS, Azure App Service, Azure Container Apps, and Azure Functions on the isolated worker model ([learn.microsoft.com/azure/azure-functions/dotnet-isolated-process-guide](https://learn.microsoft.com/azure/azure-functions/dotnet-isolated-process-guide)).
  On-prem IIS is not the default.
- **Multi-team monorepos or bounded-context services**, not single-solution single-team codebases.
  Layout uses `slnx` + Traversal SDK, CPM, and lock files ([devblogs.microsoft.com/dotnet/introducing-slnx-support-dotnet-cli](https://devblogs.microsoft.com/dotnet/introducing-slnx-support-dotnet-cli)).
- **OpenTelemetry as the instrumentation contract** for traces, metrics, and logs, with vendor backends pluggable behind the OTel exporter ([learn.microsoft.com/dotnet/core/diagnostics/observability-with-otel](https://learn.microsoft.com/dotnet/core/diagnostics/observability-with-otel)).
- **Engineering owns production.**
  On-call .NET engineers read their own dashboards, ship their own hotfixes, and own their own probes and graceful shutdown ([learn.microsoft.com/aspnet/core/fundamentals/host/web-host#shutdown-timeout](https://learn.microsoft.com/aspnet/core/fundamentals/host/web-host)).
- **Microsoft Entra ID** (formerly Azure AD) is the assumed identity provider for both workload and user identity ([learn.microsoft.com/entra/identity-platform/v2-overview](https://learn.microsoft.com/entra/identity-platform/v2-overview)).

---

## 3. What this guide IS

- **Opinionated doctrine.**
  Each chapter states a default and the explicit conditions under which the fallback applies.
- **Primary-source-cited.**
  Concrete claims point at Microsoft Learn, the .NET / ASP.NET / EF Core team blogs, `dotnet/runtime` and `dotnet/aspnetcore` docs, or named industry voices when they are the original popularisers ([CONTRIBUTING.md](./CONTRIBUTING.md)).
- **Decisive over hedging.**
  When two paths exist and one wins for the assumed envelope, the guide names the winner and the conditions for the loser to be correct.
- **Targeted at code review.**
  The [`checklist.md`](./checklist.md) and [`patterns/anti-patterns.md`](./patterns/anti-patterns.md) pages exist so reviewers can point at a line, not a vibe.

---

## 4. What this guide is NOT

Stating the negatives so the guide doesn't get blamed for being bad at things it never claimed to be good at.

- **Not a tutorial.**
  Readers are assumed to know what DI, async, EF Core, and a Dockerfile are.
  "What is `IHostedService`" questions belong on Microsoft Learn.
- **Not a beginner's intro to .NET or C#.**
  If you are picking up the language, start with the official tour ([learn.microsoft.com/dotnet/csharp/tour-of-csharp](https://learn.microsoft.com/dotnet/csharp/tour-of-csharp)) and come back when you own a service.
- **Not a Microsoft Learn replacement.**
  Where Learn is already the canonical reference, this guide links to it instead of paraphrasing it.
- **Not an exhaustive API reference.**
  No method-by-method walkthroughs of `HttpClient`, EF Core, or the BCL.
- **Not a framework comparison.**
  No "ASP.NET Core vs Node vs Go" benchmarks; the framework choice is assumed.
- **Not a regulated-industry compliance manual.**
  Auth patterns are sketched at the Entra level; mapping to FedRAMP, HIPAA, PCI-DSS, or SOC 2 is downstream.
- **Not a data engineering or analytics guide.**
  EF Core, Dapper, and raw ADO.NET for OLTP are in scope; ETL, Spark, Synapse, and Fabric pipelines are not.

---

## 5. Explicit non-goals

These are deliberately out of scope.
Call them out so readers don't expect them.

- **Xamarin.**
  End-of-support since May 2024; use .NET MAUI instead, covered in [`docs/07-client.md`](./docs/07-client.md) ([devblogs.microsoft.com/dotnet/xamarin-support-ends-in-may-2024](https://devblogs.microsoft.com/dotnet/xamarin-support-ends-in-may-2024)).
- **WPF and WinForms internals.**
  Mentioned only as legacy desktop hosts that still receive servicing ([learn.microsoft.com/dotnet/desktop/wpf](https://learn.microsoft.com/dotnet/desktop/wpf)); no XAML, control-template, or MVVM-on-WPF guidance.
- **WCF.**
  Out of support on modern .NET; use gRPC or ASP.NET Core minimal APIs instead ([learn.microsoft.com/dotnet/core/additional-tools/wcf-web-service-reference-guide](https://learn.microsoft.com/dotnet/core/porting/index)).
  CoreWCF exists but is not a recommended default for new services.
- **Blazor Hybrid deep-dive.**
  Render-mode trade-offs for Blazor United / SSR / WASM are in [`docs/07-client.md`](./docs/07-client.md); Blazor-in-MAUI / Blazor-in-WPF hybrid hosting is acknowledged but not detailed ([learn.microsoft.com/aspnet/core/blazor/hybrid](https://learn.microsoft.com/aspnet/core/blazor/hybrid)).
- **Unity and game development.**
  Unity ships its own forked runtime and tooling; nothing here transfers.
- **Workflow Foundation (WF) and BizTalk.**
  Legacy; not on modern .NET.
- **Client-side mobile UX patterns beyond MAUI defaults.**
  Navigation shells, gestures, platform-specific renderers, store submission, and accessibility on iOS/Android are out of scope past the architecture sketch in [`docs/07-client.md`](./docs/07-client.md).

---

## 6. Reading paths

Pick the path that matches why you opened the guide.
Each path is ordered; do not skip steps.

### 6.1 First 90 days on a new .NET 10 service

- Start at [`patterns/anti-patterns.md`](./patterns/anti-patterns.md) for the decision trees implied by the rejected patterns.
- Then [`checklist.md`](./checklist.md) as the one-page review card.
- Then [`docs/01-foundations.md`](./docs/01-foundations.md) for solution layout, CPM, analyzers, async, and DI.

### 6.2 Architect choosing the shape of a new bounded context

- Start at [`patterns/anti-patterns.md`](./patterns/anti-patterns.md) and [`patterns/patterns.md`](./patterns/patterns.md) for the rejected and preferred shapes.
- Then [`docs/02-aspnetcore.md`](./docs/02-aspnetcore.md) for Minimal vs Controllers, ProblemDetails, versioning, OpenAPI, resilience.
- Then [`docs/06-cloud-native.md`](./docs/06-cloud-native.md) for Aspire, AKS probes, container hygiene.

### 6.3 SRE / performance engineer chasing latency or cost

- Start at [`docs/05-performance.md`](./docs/05-performance.md) for BenchmarkDotNet methodology, allocation analysis, `Span<T>`, pooling, source generators, NativeAOT.
- Then [`docs/06-cloud-native.md`](./docs/06-cloud-native.md) for OTel defaults, probes, limits, graceful shutdown.
- Then [`checklist.md`](./checklist.md) for the perf-relevant review items.

---

## 7. Maintenance posture

- **Targets the current GA stable .NET.**
  At time of writing, .NET 10 (STS / LTS per [learn.microsoft.com/dotnet/core/releases-and-support](https://learn.microsoft.com/dotnet/core/releases-and-support)).
  When .NET 11 ships, guidance moves with it; the previous LTS is called out inline only where defaults differ.
- **Targets the current Aspire GA, currently 9.x** ([learn.microsoft.com/dotnet/aspire/whats-new/dotnet-aspire-9](https://learn.microsoft.com/dotnet/aspire/whats-new/dotnet-aspire-9)).
- **Pre-GA features are labeled.**
  Anything behind a preview feature flag, an `[Experimental]` attribute, or a `RequiresPreviewFeatures` gate is marked `preview` inline and is not a default.
- **EF Core, ASP.NET Core, and runtime release notes are the source of truth** for breaking changes between minor versions ([learn.microsoft.com/ef/core/what-is-new](https://learn.microsoft.com/ef/core/what-is-new), [learn.microsoft.com/aspnet/core/release-notes](https://learn.microsoft.com/aspnet/core/release-notes)).
- **Citations age out.**
  A claim with only a single recent team-blog post is a heuristic, not a rule; promote it to `must` only when a second primary source confirms it ([CONTRIBUTING.md](./CONTRIBUTING.md)).

---

## 8. Glossary of disclaimers used in chapters

The guide tries to be honest about where opinions are strong and where they are conditional.
The recurring qualifiers:

- **"default unless …"** — the recommendation holds for the assumed envelope (§2); the listed exceptions are the only sanctioned reasons to deviate.
- **`must` / `should` / `prefer` / `avoid`** — strength markers defined in [`CONTRIBUTING.md`](./CONTRIBUTING.md).
  `must` is load-bearing; `should` is a strong default with a written-reason escape hatch; `prefer` is a taste call backed by evidence; `avoid` is explicitly rejected with a named replacement.
- **"preview"** — the API or feature is not GA on the targeted runtime; do not adopt as a default.
- **"legacy"** — the technology is in support but not recommended for new code; the named replacement is the default.
- **"Concrete check:"** — turns an opinion into something a reviewer can verify in a PR or repo.
  If a recommendation has no concrete check, treat it as a heuristic, not a rule.
- **"Primary source"** — Microsoft Learn, `dotnet/*` repo docs, .NET / ASP.NET / EF Core team blogs, or named industry voices when they are the original popularisers.
  A single recent blog post does not qualify on its own ([CONTRIBUTING.md](./CONTRIBUTING.md)).

If a chapter ever drops these qualifiers and asserts a universal, treat it as a bug and open an issue.
