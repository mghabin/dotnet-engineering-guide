# dotnet-engineering-guide

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/mghabin/dotnet-engineering-guide/badge)](https://securityscorecards.dev/viewer/?uri=github.com/mghabin/dotnet-engineering-guide)

An opinionated **.NET 10 / C# 14 / Aspire** doctrine for senior backend engineers who own long-lived services in production, compiled from primary sources (Microsoft Learn, the .NET / ASP.NET / EF Core team blogs, `dotnet/runtime` docs, IETF RFCs, the OpenTelemetry spec) and a small set of named industry voices, with every chapter ending in a Sources block so you can follow the citations.

Aspire baseline is split by target framework: **Aspire 9.x GA** for `net8.0` / `net9.0` services, **Aspire 13 GA** for `net10.0` services (the version line jumped 9.x → 13.0 to align with the .NET 10 wave and the rebrand to "Aspire" with docs at <https://aspire.dev>).

## Read this first

These five pages are the synthesis.
The numbered chapters are doctrine you reach for *when the decision tree points you there*.

- [`docs/decision-trees.md`](./docs/decision-trees.md) — one-screen decision trees for the questions .NET teams actually argue about (Minimal APIs vs Controllers, EF Core vs Dapper / Cosmos vs SQL, EF migration deploy, Blazor render mode, client platform, test type, assertion library, NativeAOT, GC, `IHttpClientFactory`, auth policy shape, cache tier, K8s QoS, resilience, Aspire scope, Cosmos partition key).
- [`checklist.md`](./checklist.md) — one-page do/don't card for code review, every line a `must`-severity rule.
- [`SCOPE.md`](./SCOPE.md) — who this guide is for, who it isn't, and the technical and organisational envelope the defaults assume.
- [`glossary.md`](./glossary.md) — canonical, opinionated definitions for every term used normatively, with primary-source citations where the ecosystem disagrees.
- [`coverage-map.md`](./coverage-map.md) — what each chapter owns, what it defers to a sibling chapter, and what is intentionally out of scope.

## Chapters (doctrine)

Read in depth when the decision tree or checklist points you here.
The numbered order also works as a linear read for engineers new to the .NET 10 / C# 14 / Aspire stack.

- [`docs/01-foundations.md`](./docs/01-foundations.md) — language baseline (C# 14, NRT, analyzers), the `async`/`await` rules (Cleary canon), DI lifetimes, and runtime configuration defaults for the Generic Host.
- [`docs/02-aspnetcore.md`](./docs/02-aspnetcore.md) — Minimal APIs as the default, ProblemDetails (RFC 9457) for errors, Microsoft Entra bearer auth with `scp` vs `roles` modelled as separate delegated and app-only policies, and OpenTelemetry wired from day one.
- [`docs/03-data.md`](./docs/03-data.md) — EF Core 10 patterns, Cosmos DB modelling, zero-downtime migrations (expand-contract), and the cache decision matrix from in-memory through HybridCache to CDN.
- [`docs/04-testing.md`](./docs/04-testing.md) — xUnit v3 on Microsoft.Testing.Platform (MTP), `WebApplicationFactory` for in-process integration, Testcontainers for real dependencies, and Aspire `DistributedApplicationTestingBuilder` for full App Host coverage.
- [`docs/05-performance.md`](./docs/05-performance.md) — BenchmarkDotNet methodology, allocation analysis with pooling (`Span<T>`, `ArrayPool`, `RecyclableMemoryStream`), NativeAOT trade-offs, and server GC / DATAS tuning.
- [`docs/06-cloud-native.md`](./docs/06-cloud-native.md) — .NET Aspire (9.x for net8/9, 13 for net10) as the local-orchestration and deployment-manifest authority, Kubernetes probes and QoS classes, service discovery, and Data Protection keyring management in cloud.
- [`docs/07-client.md`](./docs/07-client.md) — Blazor render-mode decision tables (Static SSR, Server, WASM, Auto) on .NET 10 / C# 14, MAUI Window lifecycle, and MVVM with the CommunityToolkit source generators.

## Patterns

Separate from the chapter docs because these pages are about **how a .NET codebase ages well**, not about a single subsystem.

- [`patterns/monorepo.md`](./patterns/monorepo.md) — when a .NET monorepo is the right call and how to lay it out (`slnx` + Traversal SDK, CPM, lock files, `dotnet-affected`, package source mapping).
- [`patterns/anti-patterns.md`](./patterns/anti-patterns.md) — patterns rejected in code review, each with the failure mode and the right replacement.
- [`patterns/patterns.md`](./patterns/patterns.md) — the positive patterns to reach for instead, framed for idiomatic .NET 10.
- [`patterns/references.md`](./patterns/references.md) — curated source list shared across the chapters and patterns pages above.

## How this is maintained

- Opinionated by design: where the ecosystem disagrees, this guide picks a side and names it; chapters do not enumerate every alternative.
- Primary-source citations only: Microsoft Learn, the .NET / ASP.NET / EF Core / runtime team blogs, IETF RFCs, the OpenTelemetry spec, the CNCF, and a small set of named industry voices.
- Every chapter ends with a per-page **Sources** block listing the exact pages cited inline; replace a citation, not a chapter, when the underlying source moves.
- CI runs markdownlint, lychee link-check, and actionlint on every PR (see [`.github/workflows/ci.yml`](./.github/workflows/ci.yml)).
- Every third-party GitHub Action is pinned to a commit SHA and verified by `pinact` in CI.
- Supply-chain posture is published via OpenSSF Scorecard (badge above).

## Contributing

Issues and PRs that improve clarity, fix factual errors, or add primary-source citations are very welcome.
PRs that broaden scope toward "everything about .NET" are usually not — see [`CONTRIBUTING.md`](./CONTRIBUTING.md) and [`SCOPE.md`](./SCOPE.md).

## License

[MIT](./LICENSE) © 2026 Mohammad Ghabin
