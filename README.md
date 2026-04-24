# dotnet-engineering-guide

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

An opinionated reference for engineering modern .NET services, compiled from
Microsoft Learn, the .NET / ASP.NET team blogs, `dotnet/runtime` docs, and
widely-respected community voices. Every page ends with a Sources section so
you can follow the citations.

Target: **.NET 10 / C# 13**. Older versions are called out inline when guidance
differs.

## How to read

- **[`foundations.md`](./foundations.md)** — projects & solutions (slnx, Central
  Package Management, `Directory.Build.props`, analyzers, treat-warnings-as-errors),
  C# 13 language features, nullable reference types end-to-end, async (Cleary
  canon), DI & lifetimes.
- **[`aspnetcore.md`](./aspnetcore.md)** — Minimal vs Controllers, model binding
  & validation, ProblemDetails, versioning, OpenAPI, rate-limiting, output
  caching, OpenTelemetry wiring, resilience with `Microsoft.Extensions.Resilience`.
- **[`data.md`](./data.md)** — EF Core (tracking, split queries, compiled
  queries, pooling, migrations, transactions), when to drop to Dapper/raw SQL,
  testing data access.
- **[`testing.md`](./testing.md)** — xUnit v3, FluentAssertions / Shouldly,
  NSubstitute vs Moq, `WebApplicationFactory`, Testcontainers, snapshot testing,
  mutation testing with Stryker.
- **[`performance.md`](./performance.md)** — BenchmarkDotNet methodology,
  allocation analysis, `Span<T>` / `Memory<T>`, pooling, source generators,
  NativeAOT trade-offs, server GC tuning.
- **[`cloud-native.md`](./cloud-native.md)** — .NET Aspire app model, service
  discovery, OTel defaults, AKS liveness/readiness, graceful shutdown, container
  and image best practices.
- **[`client.md`](./client.md)** — Blazor United / SSR / WASM render-mode
  trade-offs, MAUI architecture, MVVM with CommunityToolkit, navigation.
- **[`monorepo-and-antipatterns.md`](./monorepo-and-antipatterns.md)** — .NET
  monorepo strategy, 38 anti-patterns to avoid, positive patterns, GoF design
  patterns through a .NET lens.
- **[`checklist.md`](./checklist.md)** — one-page do/don't list for code review.

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

