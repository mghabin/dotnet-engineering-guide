<!-- markdownlint-disable MD024 -->
# Coverage map — who owns what across the .NET 10 guide

Companion to [`README.md`](./README.md) and [`checklist.md`](./checklist.md).
Every concept is owned by exactly one chapter.
Sibling chapters reference (link) the owner; they do not re-decide.
This page is the master "who owns what" matrix — if two chapters look like they overlap, the rule in [Conflict resolution](#conflict-resolution) names the owner.

Primary sources for ownership: the chapters themselves under [`docs/`](./docs/), the supporting volume under [`patterns/`](./patterns/), and the operational reference [`checklist.md`](./checklist.md).

---

## Conflict resolution

- If two chapters appear to overlap, the **earlier chapter owns the foundation**, and the **later chapter owns the specialization**.
- Example: [`01-foundations`](./docs/01-foundations.md) owns DI lifetimes as a runtime concept; [`02-aspnetcore`](./docs/02-aspnetcore.md) owns the DI surface inside Minimal API endpoints and filters.
- Example: [`01-foundations`](./docs/01-foundations.md) owns `Microsoft.Extensions.Logging` primitives and source-generated loggers; [`06-cloud-native`](./docs/06-cloud-native.md) owns OTLP exporter wiring and collector topology.
- Example: [`02-aspnetcore`](./docs/02-aspnetcore.md) owns the OTel HTTP wiring on the request pipeline; [`05-performance`](./docs/05-performance.md) owns OTel application metrics and meter design; [`06-cloud-native`](./docs/06-cloud-native.md) owns OTLP exporters and Aspire `ServiceDefaults`.
- Example: [`03-data`](./docs/03-data.md) owns the transactional outbox pattern at the data tier; [`06-cloud-native`](./docs/06-cloud-native.md) references it from the messaging integration story.
- Example: [`04-testing`](./docs/04-testing.md) owns `WebApplicationFactory`, Testcontainers, and Aspire `DistributedApplicationTestingBuilder`; chapters that mention integration testing link here.
- Example: [`05-performance`](./docs/05-performance.md) owns NativeAOT, GC tuning, and container env-vars (`DOTNET_*`, `DOTNET_GCHeapCount`, tiered/PGO); [`06-cloud-native`](./docs/06-cloud-native.md) references the env-vars from the Dockerfile baseline.
- A chapter may **demonstrate** a deferred concept in a code sample, but it must not redefine the rule. The rule lives in the owner.

---

## Chapter 01 — foundations

Source: [`docs/01-foundations.md`](./docs/01-foundations.md).

### Owns

- Solution and project layout for a .NET 10 codebase (`src/`, `tests/`, `Directory.Build.props`, `Directory.Packages.props`, central package management).
- C# 13 / C# 14 language surface and target-framework policy (`net10.0`, LangVersion, file-scoped namespaces, primary constructors, collection expressions).
- Nullable reference types as a hard default and the migration discipline around `#nullable enable`.
- `async`/`await` fundamentals: `ConfigureAwait`, `ValueTask`, `CancellationToken` propagation, sync-over-async as an anti-pattern.
- Dependency injection lifetimes (`Singleton` / `Scoped` / `Transient`) and captive-dependency rules.
- Options pattern (`IOptions<T>`, `IOptionsSnapshot<T>`, `IOptionsMonitor<T>`) and validation (`ValidateDataAnnotations`, `ValidateOnStart`).
- Configuration providers, ordering, and `IConfiguration` binding.
- Logging primitives (`Microsoft.Extensions.Logging`, `ILogger<T>`, `LoggerMessage` source-generated logging, log levels).
- Source generators as a first-class compile-time tool (logging, JSON, regex, configuration binding).
- Runtime configuration knobs that belong to the process: GC mode (Server/Workstation, DATAS), tiered compilation, dynamic PGO.

### References (does not re-decide)

- DI usage inside endpoints and filters → [Chapter 02 — aspnetcore](#chapter-02--aspnetcore).
- EF Core `DbContext` lifetime and pooling → [Chapter 03 — data](#chapter-03--data).
- Hosted-service testing patterns → [Chapter 04 — testing](#chapter-04--testing).
- NativeAOT, GC tuning specifics, container env-vars → [Chapter 05 — performance](#chapter-05--performance).
- OTLP exporters and `ServiceDefaults` wiring → [Chapter 06 — cloud-native](#chapter-06--cloud-native).
- Blazor / MAUI client lifetimes and render modes → [Chapter 07 — client](#chapter-07--client).

### Cross-links

- Logging primitives owned here ↔ telemetry export owned in [06](#chapter-06--cloud-native) and meter design in [05](#chapter-05--performance).
- DI lifetimes owned here ↔ endpoint DI surface in [02](#chapter-02--aspnetcore) ↔ `DbContext` scoping in [03](#chapter-03--data).
- Options pattern owned here ↔ secrets/config in cloud in [06](#chapter-06--cloud-native).

---

## Chapter 02 — aspnetcore

Source: [`docs/02-aspnetcore.md`](./docs/02-aspnetcore.md).

### Owns

- Minimal APIs vs MVC controllers — when each is correct, route-group composition, endpoint filters.
- `ProblemDetails` per RFC 9457 as the single error contract (`AddProblemDetails`, `IProblemDetailsService`, exception handler middleware).
- API versioning policy (`Asp.Versioning.Http`, URL/segment vs header) and deprecation headers.
- OpenAPI generation via `Microsoft.AspNetCore.OpenApi` (transformer pipeline, schema customization, replacing Swashbuckle).
- AuthN/AuthZ on the request pipeline (§10): JWT bearer with Microsoft Entra, validation of `iss`/`aud`/`scp`/`roles`/`azp`, **separate delegated (`scp`) and app-only (`roles` + `azp` allow-list) policies** (no single OR-claims policy), CAE / claims-challenge wiring.
- Antiforgery / CSRF handling for cookie-auth surfaces and Blazor server interactions on the API side (§14 — Security).
- OpenTelemetry HTTP wiring on the request pipeline (§6 — `AddAspNetCoreInstrumentation`, `AddHttpClientInstrumentation`, propagation).
- **HTTP resilience for outbound calls** (§7): `IHttpClientFactory` + `AddStandardResilienceHandler` as the single owner of retry / timeout / circuit-breaker / hedging defaults. Chapter 06 references this section, does not re-decide.
- **`HttpClient` factory and typed clients** (§11): `AddHttpClient<T>`, named vs typed, `SocketsHttpHandler` defaults, `PooledConnectionLifetime`.
- **OutputCache** (§9): HTTP-response caching middleware (`AddOutputCache`, policies, tag invalidation). Chapter 03's caching matrix links here for the OutputCache row.
- **Background work in the request host** (§12): `IHostedService` / `BackgroundService` patterns, `Channel<T>` pipelines, graceful drain of in-flight work — composition lifetime owned here; the cluster-side drain contract is owned by [06 §11](#chapter-06--cloud-native).
- **Security middleware** (§14): HTTPS redirection, HSTS, security headers, antiforgery, request-size limits, forwarded headers (paired with §15).
- **gRPC surface** (§16): `MapGrpcService`, gRPC-Web, code-first vs proto-first, interceptors, deadlines.
- **SignalR surface** (§17): hub authorization, backplane choice, scaling rules, sticky sessions.
- **Health-check endpoint mapping** (§18): the in-process `MapHealthChecks` helpers and how the request host exposes them — but the **probe contract** (endpoint names, semantics, what a check may do) is owned by [06 §10](#chapter-06--cloud-native).
- Rate limiting middleware (§8), request size limits, and Kestrel surface defaults relevant to APIs.

### References (does not re-decide)

- Logging primitives and source-generated loggers → [Chapter 01 — foundations](#chapter-01--foundations).
- Validation libraries and model-binding edge cases → [`docs/01-foundations.md`](./docs/01-foundations.md) and `validation.md` referenced from chapter 02.
- EF Core `DbContext` registration and async query patterns → [Chapter 03 — data](#chapter-03--data).
- `WebApplicationFactory` integration tests → [Chapter 04 — testing](#chapter-04--testing).
- OTel meters and application metrics → [Chapter 05 — performance](#chapter-05--performance).
- OTLP exporters, `ServiceDefaults`, K8s probes → [Chapter 06 — cloud-native](#chapter-06--cloud-native).

### Cross-links

- ProblemDetails owned here ↔ EF Core exception translation in [03](#chapter-03--data).
- JWT bearer + Entra owned here ↔ Blazor auth-state serialization in [07](#chapter-07--client).
- OTel HTTP wiring owned here ↔ OTel application metrics in [05](#chapter-05--performance) ↔ OTLP exporters in [06](#chapter-06--cloud-native).

---

## Chapter 03 — data

Source: [`docs/03-data.md`](./docs/03-data.md).

### Owns

- EF Core 10 modeling: keys, owned types, value converters, `[Owned]` vs JSON columns.
- Query patterns: `AsNoTracking`, projections to DTOs, compiled queries, `AsSplitQuery` vs single-query, N+1 avoidance.
- Bulk mutations with `ExecuteUpdateAsync` / `ExecuteDeleteAsync` instead of change-tracker round-trips.
- Migrations, migration bundles (`dotnet ef migrations bundle`), and zero-downtime expand/contract schema changes.
- Cosmos DB usage with the EF Core Cosmos provider and SDK direct mode — partition key design, RU budgeting.
- Transaction scope, `IDbContextFactory<T>`, and the **transactional outbox** pattern at the data tier.
- Caching matrix: in-memory (`IMemoryCache`), distributed (`IDistributedCache`, Redis), `HybridCache`, and EF second-level cache anti-pattern.
- `DbContext` lifetime, pooling (`AddDbContextPool`), and threading rules.

### References (does not re-decide)

- DI lifetimes underlying `DbContext` registration → [Chapter 01 — foundations](#chapter-01--foundations).
- Returning `ProblemDetails` for data-layer failures → [Chapter 02 — aspnetcore](#chapter-02--aspnetcore).
- Respawn-based test data reset and Testcontainers for SQL/Cosmos → [Chapter 04 — testing](#chapter-04--testing).
- Span/Memory/Pipelines for high-throughput data paths → [Chapter 05 — performance](#chapter-05--performance).
- Outbox dispatcher hosted in the cluster, message broker integrations → [Chapter 06 — cloud-native](#chapter-06--cloud-native).

### Cross-links

- Outbox owned here ↔ broker/dispatcher topology in [06](#chapter-06--cloud-native).
- Caching matrix owned here ↔ Data Protection key ring (separate concern) in [06](#chapter-06--cloud-native).
- Migration bundles owned here ↔ K8s init-container / job invocation in [06](#chapter-06--cloud-native).

---

## Chapter 04 — testing

Source: [`docs/04-testing.md`](./docs/04-testing.md).

### Owns

- xUnit v3 GA as the default test framework and `Microsoft.Testing.Platform` (MTP) as the runner.
- `WebApplicationFactory<T>` for in-process integration tests against ASP.NET Core hosts.
- Testcontainers for SQL Server, PostgreSQL, Redis, Cosmos emulator, and broker fixtures.
- Aspire `DistributedApplicationTestingBuilder` for end-to-end app-graph tests.
- Respawn for inter-test database reset, and the rules for when to truncate vs migrate.
- Assertion-library migration: AwesomeAssertions / Shouldly in place of FluentAssertions (post-license-change).
- Hosted-service / `BackgroundService` testing patterns, time abstraction (`TimeProvider`, `FakeTimeProvider`).
- Test taxonomy (unit / integration / e2e) and parallelization rules under MTP.

### References (does not re-decide)

- Code under test conventions (NRT, async, DI lifetimes) → [Chapter 01 — foundations](#chapter-01--foundations).
- Endpoint construction and ProblemDetails contracts being asserted → [Chapter 02 — aspnetcore](#chapter-02--aspnetcore).
- EF Core fixtures, migrations, and Cosmos provider behavior → [Chapter 03 — data](#chapter-03--data).
- BenchmarkDotNet and perf assertions → [Chapter 05 — performance](#chapter-05--performance).
- Aspire AppHost wiring under test → [Chapter 06 — cloud-native](#chapter-06--cloud-native).
- Blazor `bUnit` and MAUI device-test specifics → [Chapter 07 — client](#chapter-07--client).

### Cross-links

- `DistributedApplicationTestingBuilder` owned here ↔ Aspire AppHost in [06](#chapter-06--cloud-native).
- `WebApplicationFactory` owned here ↔ JWT bearer + ProblemDetails in [02](#chapter-02--aspnetcore).
- Respawn owned here ↔ EF migrations in [03](#chapter-03--data).

---

## Chapter 05 — performance

Source: [`docs/05-performance.md`](./docs/05-performance.md).

### Owns

- BenchmarkDotNet usage: project layout, diagnosers (`MemoryDiagnoser`, ETW/EventPipe), regression gates.
- Allocation discipline and pooling (`ArrayPool<T>`, `ObjectPool<T>`, `RecyclableMemoryStream`).
- `Span<T>` / `Memory<T>` / `System.IO.Pipelines` for parsing and high-throughput I/O.
- NativeAOT: project-level enablement, trimming annotations, source-generator-friendly libraries, unsupported APIs.
- GC tuning: Server vs Workstation, DATAS (`DOTNET_GCDynamicAdaptationMode`), `GCHeapCount`, LOH/POH considerations.
- Container Dockerfile env-vars for runtime tuning: `DOTNET_TieredPGO`, `DOTNET_GCServer`, `DOTNET_GCDynamicAdaptationMode`, `DOTNET_gcHeapHardLimit`.
- OpenTelemetry **application metrics**: `Meter`, instrument types, naming, low-cardinality tag rules.

### References (does not re-decide)

- `async`/`await`, `ValueTask`, source generators as the language baseline → [Chapter 01 — foundations](#chapter-01--foundations).
- HTTP-pipeline OTel instrumentation that produces transport metrics → [Chapter 02 — aspnetcore](#chapter-02--aspnetcore).
- EF Core query-shape choices that dominate allocations → [Chapter 03 — data](#chapter-03--data).
- Microbenchmark fixtures vs integration perf tests → [Chapter 04 — testing](#chapter-04--testing).
- OTLP exporters, collector, and K8s CPU/QoS that the metrics are read from → [Chapter 06 — cloud-native](#chapter-06--cloud-native).

### Cross-links

- App metrics meters owned here ↔ OTLP exporters in [06](#chapter-06--cloud-native).
- Container env-vars owned here ↔ Dockerfile baseline in [06](#chapter-06--cloud-native).
- NativeAOT owned here ↔ source generators in [01](#chapter-01--foundations).

---

## Chapter 06 — cloud-native

Source: [`docs/06-cloud-native.md`](./docs/06-cloud-native.md).

### Owns

- .NET Aspire AppHost composition and client integrations — **Aspire 9.x** for `net8.0` / `net9.0` services, **Aspire 13** for `net10.0` services (`Aspire.Hosting.*`, `Aspire.*`).
- `ServiceDefaults` shared project (§1): OTel wiring, default health checks, resilience handlers, service discovery defaults.
- **Configuration in cluster** (§4): ConfigMap + CSI Key Vault driver as the configuration substrate; `appsettings.Production.json` rejected. Foundations §6 owns the in-process options pattern; this chapter owns where the bytes come from in the cluster.
- **Resilience pipeline defaults at the platform layer** (§6): Polly v8 standard-pipeline composition for non-HTTP resources (queues, blobs, repositories) and the rationale for keeping HTTP resilience in [02 §7](#chapter-02--aspnetcore). HTTP retry verb policy is owned by ch02; this chapter does not re-decide it.
- **Health-check probe contract** (§10): `/health/live`, `/health/ready`, `/health/startup` endpoint names, semantics, what each probe may do, and the K8s probe wiring. Aspire `ServiceDefaults` ships **only** `/health` + `/alive` and **only in Development** — production code must explicitly `MapHealthChecks` for `/health/live`, `/health/ready`, `/health/startup` (see [ch06 §10](#chapter-06--cloud-native) and the [aspire.dev ServiceDefaults reference](https://aspire.dev/fundamentals/service-defaults/)). Chapter 02 §18 only owns the in-process `MapHealthChecks` plumbing and links here for the contract.
- Kubernetes runtime contract: liveness / readiness / startup probes, QoS class, the CPU-limits-considered-harmful stance, requests sizing.
- Service discovery via `Microsoft.Extensions.ServiceDiscovery` and `HttpClient` integration (§7).
- **Networking** (§13): HTTP/2 + HTTP/3 enablement, forwarded headers in cluster, mTLS via service mesh, TLS termination boundary.
- OTLP exporters (traces, metrics, logs) and collector topology assumptions (§5).
- **Secrets & identity** (§8): Workload Identity + Federated Identity Credentials only — no client secrets, no pod-identity v1, no service-principal passwords on disk.
- ASP.NET Core Data Protection key-ring storage (Azure Blob + Key Vault, or equivalent) for multi-replica deployments (§9).
- **Graceful shutdown** (§11): `IHostApplicationLifetime`, `preStop` hook, drain order, K8s `terminationGracePeriodSeconds`. Chapter 02's background-work section composes against this contract.
- Outbox **dispatcher** topology (hosted service / sidecar / worker pool) — the pattern itself is owned by [03](#chapter-03--data).
- **CI/CD** (§12): reproducible image build, Sigstore/cosign signing, SBOM, scanning gate. Source-build hygiene is owned by [01 §11](#chapter-01--foundations); this chapter owns the cluster-deployment pipeline.
- **Multi-tenancy at runtime** (§14): tenant-scoped DI, per-tenant configuration / connection-string resolution, isolation expectations at the cluster boundary.
- **Cost & efficiency** (§15): right-sizing requests, scale-to-zero suitability, image size baseline, the cluster-level levers a service team owns.
- Dockerfile baseline for .NET 10 images, including the runtime env-vars selected in [05](#chapter-05--performance).

### References (does not re-decide)

- Logging primitives feeding the OTLP log exporter → [Chapter 01 — foundations](#chapter-01--foundations).
- Endpoint surface and OTel HTTP instrumentation → [Chapter 02 — aspnetcore](#chapter-02--aspnetcore).
- EF migration bundles invoked as init-containers / Jobs → [Chapter 03 — data](#chapter-03--data).
- `DistributedApplicationTestingBuilder` and integration-test wiring → [Chapter 04 — testing](#chapter-04--testing).
- GC and runtime env-vars copied into the Dockerfile → [Chapter 05 — performance](#chapter-05--performance).

### Cross-links

- Aspire `ServiceDefaults` owned here ↔ OTel HTTP wiring in [02](#chapter-02--aspnetcore) ↔ app metrics in [05](#chapter-05--performance).
- K8s probes owned here ↔ `IHostApplicationLifetime` / health checks contract in [01](#chapter-01--foundations).
- Data Protection key ring owned here ↔ cookie/antiforgery surfaces in [02](#chapter-02--aspnetcore) and [07](#chapter-07--client).

---

## Chapter 07 — client

Source: [`docs/07-client.md`](./docs/07-client.md).

### Owns

- Blazor Web App unified model: render modes (`Static`, `Server`, `WebAssembly`, `Auto`) and the decision table for each.
- Authentication-state serialization across render-mode boundaries (`PersistentAuthenticationStateProvider`, `AddAuthenticationStateSerialization`).
- Forms in Blazor: `EditForm`, `[SupplyParameterFromForm]`, mandatory `FormName`, antiforgery integration on the client surface.
- `[PersistentState]` (.NET 10) for prerender → interactive state hand-off.
- .NET MAUI 10 `Window` lifecycle, single-window vs multi-window apps, lifecycle events.
- MAUI navigation: `Shell` vs `FlyoutPage` vs `NavigationPage`, with a decision table.
- Client-side DI scoping rules where they diverge from server defaults (Blazor circuits, MAUI app lifetime).

### References (does not re-decide)

- DI lifetimes, NRT, async fundamentals as language baseline → [Chapter 01 — foundations](#chapter-01--foundations).
- JWT bearer + Entra token shapes the client consumes → [Chapter 02 — aspnetcore](#chapter-02--aspnetcore).
- `bUnit` / device-test plumbing → [Chapter 04 — testing](#chapter-04--testing).
- Data Protection key ring backing Blazor auth cookies in multi-replica → [Chapter 06 — cloud-native](#chapter-06--cloud-native).

### Cross-links

- Auth-state serialization owned here ↔ JWT/Entra validation in [02](#chapter-02--aspnetcore).
- Antiforgery on `EditForm` owned here ↔ antiforgery middleware in [02](#chapter-02--aspnetcore).
- Render-mode decision table owned here ↔ Data Protection key ring in [06](#chapter-06--cloud-native).

---

## Supporting volume — patterns/

Sources: [`patterns/anti-patterns.md`](./patterns/anti-patterns.md), [`patterns/patterns.md`](./patterns/patterns.md), [`patterns/monorepo.md`](./patterns/monorepo.md), [`patterns/references.md`](./patterns/references.md).

### Owns

- Cross-cutting **anti-patterns** rejected in code review, with the named replacement for each.
- Cross-cutting **positive patterns** (incl. GoF refresher in .NET 10 idiom) that replace those anti-patterns.
- .NET **monorepo** layout, central package management, build graph, and CI fan-out strategy.
- Curated **primary-source references** shared across the patterns pages.

### References (does not re-decide)

- Subsystem-specific anti-patterns stay in their owning chapter (e.g., EF tracking misuse stays in [03](#chapter-03--data); captured-async pitfalls stay in [01](#chapter-01--foundations)).
- Per-claim citations for chapter-level rules stay inline in the owning chapter, not in `references.md`.

### Cross-links

- `patterns/patterns.md` ↔ chapters 01–07 (each named pattern points back to its owning chapter for the binding rule).
- `patterns/monorepo.md` ↔ [04 — testing](#chapter-04--testing) for fan-out test selection and [06 — cloud-native](#chapter-06--cloud-native) for image-build topology.

---

## Operational reference — checklist.md

Source: [`checklist.md`](./checklist.md).

### Owns

- The first-90-days, copy-pasteable PR checklist of `do` / `don't` pairs.
- The single page a reviewer pulls up during code review.

### References (does not re-decide)

- The card itself does not encode rationale; each line restates a rule whose canonical wording lives in the owning chapter listed in the checklist's footer chapter index.
- `should` / `prefer` / `avoid` nuance lives in chapters, not on the card.

### Cross-links

- `checklist.md` ↔ all chapters (one-way, checklist → chapter).
