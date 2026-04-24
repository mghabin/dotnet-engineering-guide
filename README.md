# dotnet-engineering-guide

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

An opinionated reference for engineering modern .NET services, compiled from
Microsoft Learn, the .NET / ASP.NET team blogs, `dotnet/runtime` docs, and
widely-respected community voices. Every page ends with a Sources section so
you can follow the citations.

Target: **.NET 10 / C# 13**. Older versions are called out inline when guidance
differs.

## Reading order

If you're new to the codebase or to .NET 10, follow the numbered order under
[`docs/`](./docs). Otherwise, jump straight to the page you need.

| # | Page | What's in it |
|---|------|--------------|
| 01 | [`docs/01-foundations.md`](./docs/01-foundations.md) | Solutions & projects (slnx, Central Package Management, `Directory.Build.props`, analyzers, treat-warnings-as-errors), C# 13, NRT, async (Cleary canon), DI & lifetimes. |
| 02 | [`docs/02-aspnetcore.md`](./docs/02-aspnetcore.md) | Minimal vs Controllers, model binding & validation, ProblemDetails, versioning, OpenAPI, OpenTelemetry wiring, resilience, rate-limiting, output caching. |
| 03 | [`docs/03-data.md`](./docs/03-data.md) | EF Core (tracking, split queries, compiled queries, pooling, migrations, transactions), when to drop to Dapper/raw SQL, testing data access. |
| 04 | [`docs/04-testing.md`](./docs/04-testing.md) | xUnit v3, FluentAssertions / Shouldly, NSubstitute vs Moq, `WebApplicationFactory`, Testcontainers, snapshot, mutation testing. |
| 05 | [`docs/05-performance.md`](./docs/05-performance.md) | BenchmarkDotNet methodology, allocation analysis, `Span<T>`, pooling, source generators, NativeAOT trade-offs, server GC tuning. |
| 06 | [`docs/06-cloud-native.md`](./docs/06-cloud-native.md) | .NET Aspire, service discovery, OTel defaults, AKS probes/limits/shutdown, container best practices. |
| 07 | [`docs/07-client.md`](./docs/07-client.md) | Blazor United / SSR / WASM render-mode trade-offs, MAUI architecture, MVVM with CommunityToolkit. |

## Patterns & anti-patterns

A separate folder because these pages are about **how a codebase ages well**,
not about a single subsystem:

- [`patterns/monorepo.md`](./patterns/monorepo.md) — when a .NET monorepo is the
  right call and how to lay it out (slnx + Traversal SDK, CPM, lock files,
  `dotnet-affected`, package source mapping, build hygiene).
- [`patterns/anti-patterns.md`](./patterns/anti-patterns.md) — 38 patterns we
  reject in code review, each with the failure mode and the right replacement.
- [`patterns/patterns.md`](./patterns/patterns.md) — the positive patterns to
  reach for instead, plus a short Gang-of-Four refresher framed for idiomatic
  .NET 10.
- [`patterns/references.md`](./patterns/references.md) — curated source list
  shared across the three pages above (Microsoft docs, team blogs, community
  thought-leaders, tooling, conference talks).

## The card

- [`checklist.md`](./checklist.md) — one-page do/don't list for code review.
  Print it, paste it into your PR template, or use it as the cliff-notes for
  the rest of this guide.

## A worked example

These practices are applied end-to-end in a small Microsoft Entra (Azure AD)
auth-patterns sample:
👉 **[mghabin/entra-auth-patterns-dotnet](https://github.com/mghabin/entra-auth-patterns-dotnet)**

Among other things, that sample demonstrates:

- Central Package Management (`Directory.Packages.props`) with locked-mode restore
- `Directory.Build.props` with `TreatWarningsAsErrors=true`, NuGet audit, and
  analyzers (`Microsoft.CodeAnalysis.NetAnalyzers`, `Meziantou.Analyzer`,
  `SonarAnalyzer.CSharp`)
- Source-generated `ILogger` (`[LoggerMessage]`) throughout
- OpenTelemetry traces + metrics
- ProblemDetails + `IExceptionHandler`
- Typed `HttpClient` + `AddStandardResilienceHandler`
- Options pattern with `ValidateDataAnnotations().ValidateOnStart()`
- Generic Host workers built on `BackgroundService` + `IHostApplicationLifetime`

## Contributing

Issues and PRs that improve clarity, fix factual errors, or add citations are
very welcome. PRs that broaden scope toward "everything about .NET" are usually
not — see [`CONTRIBUTING.md`](./CONTRIBUTING.md).

## License

[MIT](./LICENSE) © 2026 mghabin
