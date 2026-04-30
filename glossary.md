# Glossary

This glossary defines the terms used across the .NET Engineering Guide. Definitions are opinionated: where the .NET ecosystem uses a term inconsistently, the entry calls out the disambiguation and pins this guide to a specific primary source. Cross-referenced glossary terms are *italicized*. Citations point at primary sources (Microsoft Learn, .NET team blog, RFCs, Aspire docs, xunit.net, aspire.dev, kubernetes.io, opentelemetry.io, benchmarkdotnet.org) — not at blog summaries.

**Terms with strong opinions in this guide** (read these entries carefully): *AOT* vs *NativeAOT*, *Database.Migrate* vs *Migration Bundle*, *Workstation GC* vs *Server GC*, *AsNoTracking*, *ConfigureAwait*, *IHttpClientFactory*, *Interactive Auto*, *Minimal API* vs Controllers, *PGO* (Static vs Dynamic), *ProblemDetails*, *RFC 9457*.

**Deprecated or legacy** (do not use in new work): hand-rolled `HttpClient` `new`s (use *IHttpClientFactory*), `Newtonsoft.Json` for new APIs (use `System.Text.Json`), `lock(object)` over arbitrary references (use *System.Threading.Lock*), `ConcurrentDictionary` as a cache (use *HybridCache* / *IMemoryCache*), VSTest runner for new test projects (use *MTP*), `WebHostBuilder` / `Startup.cs` (use minimal hosting), `MemoryStream` per-request (use *RecyclableMemoryStream*), `Task.Run` to "make sync async", `Database.EnsureCreated` outside of tests, `Newtonsoft`-based MVC formatters in greenfield Minimal APIs.

## A

### AddAuthenticationStateDeserialization
Blazor WebAssembly extension method (`Microsoft.AspNetCore.Components.WebAssembly.Authentication`) that configures the WASM host to consume the `AuthenticationState` serialized by *AddAuthenticationStateSerialization* on the server, so the client renders with the same identity the server saw without an extra round-trip.
- **Used in**: ch07 — Part 1 — Blazor
- **Sources**: ASP.NET Core Blazor authentication and authorization — *Server-side render mode* — https://learn.microsoft.com/aspnet/core/blazor/security/server/

### AddAuthenticationStateSerialization
ASP.NET Core extension method that registers a `PersistentComponentState` provider serializing the user's `AuthenticationState` into the prerendered HTML so that an *Interactive WebAssembly* or *Interactive Auto* client receives the identity without a second authentication round-trip.
- **Used in**: ch07 — Part 1 — Blazor
- **Sources**: ASP.NET Core Blazor authentication — *Manage authentication state in Blazor Web Apps* — https://learn.microsoft.com/aspnet/core/blazor/security/server/additional-scenarios

### AddStandardResilienceHandler
`Microsoft.Extensions.Http.Resilience` extension that adds the opinionated Polly v8 resilience pipeline (rate limiter → total timeout → retry → circuit breaker → attempt timeout) to a typed/named `HttpClient`. Replaces hand-rolled retry/circuit policies.
- **Used in**: ch02 §7, ch06 §6
- **Sources**: Build resilient HTTP apps — `AddStandardResilienceHandler` — https://learn.microsoft.com/dotnet/core/resilience/http-resilience ; Polly v8 — https://www.pollydocs.org/

### Antiforgery
ASP.NET Core middleware and `IAntiforgery` service that mints and validates a synchronizer token to prevent cross-site request forgery on cookie-authenticated POST/PUT/DELETE endpoints. Required for Blazor SSR enhanced forms.
- **Used in**: ch02 §14, ch07 — Part 1
- **Sources**: Prevent Cross-Site Request Forgery (XSRF/CSRF) attacks in ASP.NET Core — https://learn.microsoft.com/aspnet/core/security/anti-request-forgery ; `AddAntiforgery` — https://learn.microsoft.com/dotnet/api/microsoft.extensions.dependencyinjection.antiforgeryservicecollectionextensions.addantiforgery

### AOT (Ahead-of-Time compilation)
Compilation strategy that produces machine code at build time rather than at *JIT* time. In .NET this umbrella covers *NativeAOT* (full native binary, no JIT, trimming), *R2R* / ReadyToRun (precompiled IL+native, JIT still present), and Mono AOT for mobile/wasm.
- **Used in**: ch01 §11, ch05 §8
- **Sources**: .NET runtime — *Ahead-of-time compilation* — https://learn.microsoft.com/dotnet/core/deploying/ ; What is Native AOT — https://learn.microsoft.com/dotnet/core/deploying/native-aot/

### AppHost (.NET Aspire)
The orchestrator project (`*.AppHost.csproj`) that declares your distributed app's resources (projects, containers, executables) and their connections. Runs locally via `dotnet run`; produces a manifest consumed by deployment tools (Azure Developer CLI, `aspirate`).
- **Used in**: ch06 §1
- **Sources**: .NET Aspire AppHost overview — https://learn.microsoft.com/dotnet/aspire/fundamentals/app-host-overview ; Aspire docs — https://aspire.dev/

### ArrayPool<T>
Shared/configurable pool of reusable `T[]` buffers in `System.Buffers`. Reduces *GC* pressure on hot paths by renting + returning arrays instead of `new T[n]`.
- **Used in**: ch05 §2
- **Sources**: `System.Buffers.ArrayPool<T>` — https://learn.microsoft.com/dotnet/api/system.buffers.arraypool-1

### AsNoTracking
EF Core query option (`IQueryable<T>.AsNoTracking()`) that skips inserting returned entities into the *ChangeTracker*. Default for read-only queries; required for projection paths and any query whose results are not saved back.
- **Used in**: ch03 §2, ch03 §7
- **Sources**: EF Core — *Tracking vs No-Tracking Queries* — https://learn.microsoft.com/ef/core/querying/tracking

### AsSingleQuery / AsSplitQuery
EF Core query operators that pin the cardinality strategy for queries containing collection navigations: `AsSingleQuery` issues one SQL `JOIN` (Cartesian-explosion risk), `AsSplitQuery` issues one query per collection. Set the default with `UseQuerySplittingBehavior`.
- **Used in**: ch03 §7
- **Sources**: EF Core — *Single vs. Split Queries* — https://learn.microsoft.com/ef/core/querying/single-split-queries

### AVX-512
Intel/AMD x86 SIMD instruction set with 512-bit registers, surfaced in .NET via `System.Runtime.Intrinsics.X86.Avx512*` and the higher-level `Vector512<T>`. Available in .NET 8+.
- **Used in**: ch05 §13
- **Sources**: .NET 8 — *AVX-512 support* — https://devblogs.microsoft.com/dotnet/dotnet-8-hardware-intrinsics/ ; `Vector512<T>` — https://learn.microsoft.com/dotnet/api/system.runtime.intrinsics.vector512-1

### azp / appid (claim)
JWT claims emitted by Microsoft Entra ID identifying the **client application** that obtained the token. `azp` is the standard OIDC "authorized party" claim (v2.0 tokens); `appid` is the Entra v1.0 equivalent. Use these — not *scp* or *roles* — to gate per-client policy.
- **Used in**: ch02 §10
- **Sources**: Microsoft identity platform — *Access tokens — payload claims* — https://learn.microsoft.com/entra/identity-platform/access-token-claims-reference ; OpenID Connect Core 1.0 §2 (`azp`) — https://openid.net/specs/openid-connect-core-1_0.html

## B

### BackgroundService
Abstract base class in `Microsoft.Extensions.Hosting` that implements *IHostedService* with a single `ExecuteAsync(CancellationToken)` method. The default for long-running queue/worker loops in ASP.NET Core and Worker Service apps.
- **Used in**: ch02 §12, ch06 §11
- **Sources**: Worker Services in .NET — https://learn.microsoft.com/dotnet/core/extensions/workers ; `BackgroundService` — https://learn.microsoft.com/dotnet/api/microsoft.extensions.hosting.backgroundservice

### BenchmarkDotNet
.NET's de-facto micro-benchmarking framework. Handles warmup, statistical analysis, multiple runtimes, exporters; pairs with *MemoryDiagnoser*. The only acceptable tool for "is this faster?" questions in this guide.
- **Used in**: ch04 §11, ch05 §1, ch05 §15
- **Sources**: BenchmarkDotNet docs — https://benchmarkdotnet.org/ ; *Good practices* — https://benchmarkdotnet.org/articles/guides/good-practices.html

### Blazor SSR (Static Server Rendering)
Default Blazor render mode in .NET 8+: components render to HTML on the server per request, no SignalR circuit, no WASM payload. Equivalent in scope to a Razor Pages response. Combine with *enhanced navigation* / *enhanced form* for SPA-like UX.
- **Used in**: ch07 — Part 1
- **Sources**: ASP.NET Core Blazor render modes — https://learn.microsoft.com/aspnet/core/blazor/components/render-modes

## C

### CAE (Continuous Access Evaluation)
Microsoft Entra capability allowing resource servers to revoke an in-flight token (group change, account disable, IP change) within minutes instead of waiting for token expiry. Clients must be CAE-aware and handle a *claims-challenge* on 401.
- **Used in**: ch02 §10
- **Sources**: Microsoft Entra — *Continuous access evaluation* — https://learn.microsoft.com/entra/identity/conditional-access/concept-continuous-access-evaluation ; *Claims challenges, claims requests, and client capabilities* — https://learn.microsoft.com/entra/identity-platform/claims-challenge

### ChangeTracker
EF Core component (`DbContext.ChangeTracker`) that snapshots loaded entities and detects modifications on `SaveChanges`. The cost you pay for tracked queries; bypass via *AsNoTracking* or projections when not needed.
- **Used in**: ch03 §2, ch03 §3
- **Sources**: EF Core — *Change Tracking* — https://learn.microsoft.com/ef/core/change-tracking/

### Chiseled image
Ubuntu-published distroless-style container image (`mcr.microsoft.com/dotnet/runtime-deps:*-jammy-chiseled`, `…-noble-chiseled`) containing only the files needed by .NET — no shell, no package manager, smaller surface for CVEs. Default base image for production .NET in this guide.
- **Used in**: ch06 §2
- **Sources**: Announcing .NET Chiseled Containers — https://devblogs.microsoft.com/dotnet/announcing-dotnet-chiseled-containers/ ; Ubuntu Chiseled — https://ubuntu.com/blog/install-dotnet-on-ubuntu

### Claims-challenge
WWW-Authenticate response from a resource server (Microsoft identity platform pattern) instructing the client to re-acquire a token presenting additional claims (MFA, device compliance, *CAE* invalidation). Encoded in the `claims` parameter of the next token request.
- **Used in**: ch02 §10
- **Sources**: Microsoft identity platform — *Claims challenges, claims requests, and client capabilities* — https://learn.microsoft.com/entra/identity-platform/claims-challenge

### Collection expressions
C# 12 syntax `[1, 2, 3]` / `[..a, ..b]` that constructs a collection whose target type is inferred. Lowers to optimal allocation for `Span<T>`, arrays, `List<T>`, and types decorated with `[CollectionBuilder]`.
- **Used in**: ch01 §2
- **Sources**: C# 12 — *Collection expressions* — https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions

### Compiled query
EF Core mechanism (`EF.CompileQuery` / `EF.CompileAsyncQuery`) that pre-compiles a LINQ query tree to a delegate, avoiding repeated translation overhead. Reach for it on hot paths after measurement.
- **Used in**: ch03 §7
- **Sources**: EF Core — *Compiled queries* — https://learn.microsoft.com/ef/core/performance/advanced-performance-topics#compiled-queries

### Complex type (EF Core 8+)
Value-typed grouping of properties mapped into the owner's table without an identity, navigation, or independent existence. The successor to *owned entity* for value-object scenarios.
- **Used in**: ch03 §2
- **Sources**: EF Core 8 — *Complex types* — https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#value-objects-using-complex-types

### ConfigureAwait
`Task`/`ValueTask` extension that controls whether the continuation captures the current `SynchronizationContext`. **In ASP.NET Core (no SynchronizationContext) it is a no-op** — do not add it ritualistically. Library code that may run under WPF/WinForms/MAUI should still use `ConfigureAwait(false)`.
- **Used in**: ch01 §4, ch05 §7
- **Sources**: Stephen Toub — *ConfigureAwait FAQ* — https://devblogs.microsoft.com/dotnet/configureawait-faq/

### CPM (Central Package Management)
NuGet feature: a repo-root `Directory.Packages.props` declares `<PackageVersion>` for every dependency; csproj files only list `<PackageReference Include="…" />` (no version). Eliminates version drift across solutions.
- **Used in**: ch01 §1
- **Sources**: NuGet — *Central Package Management* — https://learn.microsoft.com/nuget/consume-packages/central-package-management

## D

### Database.Migrate
`DbContext.Database.Migrate()` / `MigrateAsync()` applies pending EF Core migrations from the running app at startup. **Anti-pattern in production** in this guide — use a *migration bundle* run as a deployment step, or generated SQL scripts. See ch03 §4 (canonical).
- **Used in**: ch03 §4
- **Sources**: EF Core — *Applying migrations* — https://learn.microsoft.com/ef/core/managing-schemas/migrations/applying

### Data Protection key ring
ASP.NET Core's symmetric key set used by `IDataProtector` for cookies, antiforgery, identity tokens. Must be **persisted to shared storage** (Azure Blob, Redis, file share) and **encrypted at rest** for any multi-instance deployment, with `SetApplicationName` set so all replicas decrypt each other's payloads.
- **Used in**: ch06 §9
- **Sources**: ASP.NET Core Data Protection — *Configuration* — https://learn.microsoft.com/aspnet/core/security/data-protection/configuration/overview ; *Key storage providers* — https://learn.microsoft.com/aspnet/core/security/data-protection/implementation/key-storage-providers

### DATAS (Dynamic Adaptation To Application Sizes)
.NET 8+ *Server GC* mode that auto-tunes heap size to live-data set rather than to per-core constants, materially reducing memory in containers. Enable with `DOTNET_GCDynamicAdaptationMode=1` (default in .NET 9+ Server GC).
- **Used in**: ch05 §9, ch06 §3
- **Sources**: Performance Improvements in .NET 8 — *DATAS* — https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/ ; GC config — `GCDynamicAdaptationMode` — https://learn.microsoft.com/dotnet/core/runtime-config/garbage-collector

## E

### Enhanced form (Blazor)
Blazor SSR feature: forms decorated with `Enhance="true"` POST via `fetch`, the response HTML is diffed into the page, preserving scroll/focus and avoiding a full reload. Requires *antiforgery* and a *FormName*.
- **Used in**: ch07 — Part 1
- **Sources**: ASP.NET Core Blazor forms — *Enhanced form handling* — https://learn.microsoft.com/aspnet/core/blazor/forms/

### Enhanced navigation (Blazor)
Blazor SSR feature that intercepts internal anchor clicks, fetches the destination as a partial, and patches the DOM — giving SSR an SPA feel without WASM or a circuit.
- **Used in**: ch07 — Part 1
- **Sources**: ASP.NET Core Blazor routing — *Enhanced navigation* — https://learn.microsoft.com/aspnet/core/blazor/fundamentals/routing

### EventCounters
.NET runtime's lightweight always-on metric primitive (`System.Diagnostics.Tracing.EventCounter`). Largely superseded by *Meter* / `System.Diagnostics.Metrics` for new code, but remains the source for built-in runtime counters (`System.Runtime`, `Microsoft.AspNetCore.Hosting`).
- **Used in**: ch05 §1, ch06 §5
- **Sources**: .NET — *EventCounters* — https://learn.microsoft.com/dotnet/core/diagnostics/event-counters

### EventSource
Strongly-typed ETW/perf-trace event source (`System.Diagnostics.Tracing.EventSource`). The substrate for *EventCounters* and for high-throughput tracing consumed by `dotnet-trace`, PerfView, EventPipe.
- **Used in**: ch05 §1, ch06 §5
- **Sources**: .NET — *EventSource* — https://learn.microsoft.com/dotnet/core/diagnostics/eventsource

### ExecuteDeleteAsync
EF Core 7+ bulk delete operator: translates `IQueryable.ExecuteDeleteAsync()` to a single `DELETE … WHERE …` SQL statement without loading entities. Bypasses the *ChangeTracker* and `SaveChanges` interceptors — pair with manual outbox/audit if needed.
- **Used in**: ch03 §2
- **Sources**: EF Core 7 — *ExecuteUpdate and ExecuteDelete* — https://learn.microsoft.com/ef/core/what-is-new/ef-core-7.0/whatsnew#executeupdate-and-executedelete-bulk-updates

### ExecuteUpdateAsync
EF Core 7+ bulk update operator: translates `IQueryable.ExecuteUpdateAsync(setters)` to a single `UPDATE … SET … WHERE …`. Same caveats as *ExecuteDeleteAsync*.
- **Used in**: ch03 §2
- **Sources**: EF Core 7 — *ExecuteUpdate and ExecuteDelete* — https://learn.microsoft.com/ef/core/what-is-new/ef-core-7.0/whatsnew#executeupdate-and-executedelete-bulk-updates

### Expand-then-contract
Schema-evolution discipline for zero-downtime EF Core migrations: deploy additive schema changes (new column, nullable), backfill, deploy app reading both old + new, then ship a follow-up migration removing the old column. Required for any rolling deploy.
- **Used in**: ch03 §4
- **Sources**: Martin Fowler / Pramod Sadalage — *Evolutionary Database Design* — https://martinfowler.com/articles/evodb.html ; EF Core — *Migrations in team environments* — https://learn.microsoft.com/ef/core/managing-schemas/migrations/teams

## F

### FakeTimeProvider
`Microsoft.Extensions.TimeProvider.Testing` test double that lets tests advance virtual time deterministically (`Advance(TimeSpan)`). Pair with code that depends on `TimeProvider` instead of `DateTime.UtcNow`.
- **Used in**: ch04 §14
- **Sources**: `Microsoft.Extensions.TimeProvider.Testing` — https://learn.microsoft.com/dotnet/core/extensions/timeprovider ; NuGet — https://www.nuget.org/packages/Microsoft.Extensions.TimeProvider.Testing

### FluentAssertions / AwesomeAssertions / Shouldly / TUnit
Assertion-library landscape after Fluent Assertions changed its license in 2024. **AwesomeAssertions** is the OSS hard-fork (Apache-2.0); **Shouldly** is the long-standing alternative (BSD-3); **TUnit** ships its own assertion model. New projects in this guide pick AwesomeAssertions or Shouldly; existing FluentAssertions code stays put unless commercial-use thresholds bite.
- **Used in**: ch04 §3
- **Sources**: AwesomeAssertions — https://awesomeassertions.org/ ; Shouldly — https://docs.shouldly.org/ ; TUnit — https://tunit.dev/

### FlyoutPage (.NET MAUI)
`Microsoft.Maui.Controls.FlyoutPage` — two-pane navigation page exposing a flyout (master) and detail. Default chassis for tablet/desktop MAUI apps that don't fit *Shell*'s tab/flyout patterns.
- **Used in**: ch07 — Part 2
- **Sources**: .NET MAUI — *FlyoutPage* — https://learn.microsoft.com/dotnet/maui/user-interface/pages/flyoutpage

### FormName
`[SupplyParameterFromForm(FormName = "…")]` discriminator (and `<EditForm FormName="…">`) used by Blazor SSR to route a POST back to the correct component when several forms render on a page.
- **Used in**: ch07 — Part 1
- **Sources**: ASP.NET Core Blazor forms — *Handle form submission with EditForm* — https://learn.microsoft.com/aspnet/core/blazor/forms/

### [FromKeyedServices]
ASP.NET Core / .NET 8+ parameter attribute that resolves a *keyed service* by key into a controller action, minimal endpoint handler, or constructor.
- **Used in**: ch02 §10, ch01 §5
- **Sources**: ASP.NET Core 8 — *Keyed services* — https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection#keyed-services ; `FromKeyedServicesAttribute` — https://learn.microsoft.com/dotnet/api/microsoft.extensions.dependencyinjection.fromkeyedservicesattribute

### FsCheck
.NET property-based testing library (port of QuickCheck). Generates randomised inputs, shrinks failures to minimal counter-examples; integrates with xUnit/NUnit/MSTest.
- **Used in**: ch04 §10
- **Sources**: FsCheck docs — https://fscheck.github.io/FsCheck/

## G

### GCLatencyMode.SustainedLowLatency
`System.Runtime.GCLatencyMode` setting that asks the GC to avoid generation-2 collections "indefinitely" while still doing gen-0/1. Trades memory headroom for predictable latency in trading/RT-style workloads.
- **Used in**: ch05 §9
- **Sources**: `GCLatencyMode` — https://learn.microsoft.com/dotnet/api/system.runtime.gclatencymode ; *Latency modes* — https://learn.microsoft.com/dotnet/standard/garbage-collection/latency

## H

### HybridCache
`Microsoft.Extensions.Caching.Hybrid` two-tier cache (in-process L1 + *IDistributedCache* L2) shipped with .NET 9. Adds stampede protection, tagging, and source-generated serializers; supersedes hand-rolled `IMemoryCache` + `IDistributedCache` combos for new code.
- **Used in**: ch02 §9, ch03 §14
- **Sources**: ASP.NET Core — *HybridCache library* — https://learn.microsoft.com/aspnet/core/performance/caching/hybrid

## I

### IAsyncEnumerable<T>
`System.Collections.Generic.IAsyncEnumerable<T>` — async streaming sequence consumed with `await foreach`. Default return type for streaming endpoints (Minimal API auto-streams JSON), EF Core `AsAsyncEnumerable`, gRPC server-streaming bridges.
- **Used in**: ch01 §4, ch02 §1, ch03 §9
- **Sources**: *Iterating with Async Enumerables in C# 8* — https://learn.microsoft.com/archive/msdn-magazine/2019/november/csharp-iterating-with-async-enumerables-in-csharp-8 ; `IAsyncEnumerable<T>` — https://learn.microsoft.com/dotnet/api/system.collections.generic.iasyncenumerable-1

### IAuthorizationMiddlewareResultHandler
ASP.NET Core extension point invoked by the authorization middleware after policy evaluation. Replace it to customise 401/403 responses (e.g., emit *ProblemDetails*, return a *claims-challenge*).
- **Used in**: ch02 §10
- **Sources**: ASP.NET Core — *Custom Authorization Middleware Result Handler* — https://learn.microsoft.com/aspnet/core/security/authorization/authorizationmiddlewareresulthandler

### IClassFixture<T>
xUnit per-test-class shared-state hook: T is constructed once per test class, injected into the constructor, disposed after the last test. Use for an in-process fixture too expensive for per-test setup but not safe across classes.
- **Used in**: ch04 §5
- **Sources**: xUnit.net — *Shared Context: Class Fixtures* — https://xunit.net/docs/shared-context#class-fixture

### ICollectionFixture<T>
xUnit per-test-collection shared-state hook: T is constructed once for all classes in `[Collection]`-tagged test classes. Use for an expensive shared resource (Testcontainer, *WebApplicationFactory*) that must be reused across classes.
- **Used in**: ch04 §5, ch04 §6
- **Sources**: xUnit.net — *Shared Context: Collection Fixtures* — https://xunit.net/docs/shared-context#collection-fixture

### IDistributedCache
ASP.NET Core abstraction for an out-of-process key/value cache (Redis, SQL Server, NCache). The L2 backing for *HybridCache*; use directly only when you specifically need cross-instance invalidation without an L1.
- **Used in**: ch02 §9
- **Sources**: ASP.NET Core — *Distributed caching* — https://learn.microsoft.com/aspnet/core/performance/caching/distributed

### IExceptionHandler
ASP.NET Core 8+ pipeline contract (`UseExceptionHandler` registers implementations) that lets you map exception types to responses without writing custom middleware. Default mechanism for emitting *ProblemDetails* on uncaught exceptions.
- **Used in**: ch02 §3
- **Sources**: ASP.NET Core — *Handle errors with `IExceptionHandler`* — https://learn.microsoft.com/aspnet/core/fundamentals/error-handling#iexceptionhandler

### IHealthCheckPublisher
Periodic publisher contract for `Microsoft.Extensions.Diagnostics.HealthChecks` — push the latest health snapshot to a sink (logs, metrics, control plane). Independent of the HTTP `MapHealthChecks` endpoint.
- **Used in**: ch06 §10
- **Sources**: ASP.NET Core — *Health checks: publishers* — https://learn.microsoft.com/aspnet/core/host-and-deploy/health-checks#health-check-publisher

### IHostedService
`Microsoft.Extensions.Hosting.IHostedService` — `StartAsync` / `StopAsync` contract that the generic host runs alongside the app. Implement directly only for short-lived startup/shutdown work; use *BackgroundService* for long-running loops.
- **Used in**: ch02 §12, ch06 §11
- **Sources**: .NET — *Background tasks with hosted services* — https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services

### IHttpClientFactory
`Microsoft.Extensions.Http` factory that owns the lifetime of pooled `HttpMessageHandler`s, fixing socket-exhaustion and stale-DNS bugs that arise from `new HttpClient()` per call. Pair with *PooledConnectionLifetime* on the underlying *SocketsHttpHandler*. Mandatory for any outbound HTTP in this guide.
- **Used in**: ch02 §11, ch05 §10, ch06 §6
- **Sources**: ASP.NET Core — *Make HTTP requests using IHttpClientFactory* — https://learn.microsoft.com/aspnet/core/fundamentals/http-requests ; *Guidelines for using HttpClient* — https://learn.microsoft.com/dotnet/fundamentals/networking/http/httpclient-guidelines

### IMemoryOwner<T>
`System.Buffers.IMemoryOwner<T>` — RAII-style owner of a rented `Memory<T>` from a `MemoryPool<T>`. Disposing returns the buffer; pairs with `Memory<T>` consumers that need lifetime ownership.
- **Used in**: ch05 §2
- **Sources**: `System.Buffers.IMemoryOwner<T>` — https://learn.microsoft.com/dotnet/api/system.buffers.imemoryowner-1

### Interactive Auto (Blazor)
Render mode that starts a component on the *Interactive Server* circuit for fast first render, then transparently transfers to *Interactive WebAssembly* once the WASM payload is downloaded and warm. The "best of both" mode for public web with an authenticated user.
- **Used in**: ch07 — Part 1
- **Sources**: ASP.NET Core Blazor render modes — *Auto* — https://learn.microsoft.com/aspnet/core/blazor/components/render-modes#auto-render-mode

### Interactive Server (Blazor)
Render mode where component logic runs on the server and DOM diffs flow over a SignalR circuit (Blazor Server). Lowest startup latency, requires sticky sessions and per-user server memory.
- **Used in**: ch07 — Part 1
- **Sources**: ASP.NET Core Blazor render modes — *Interactive Server* — https://learn.microsoft.com/aspnet/core/blazor/components/render-modes

### Interactive WebAssembly (Blazor)
Render mode where component logic runs in the browser via WebAssembly. Highest first-load cost, zero per-user server memory, works offline; pair with *AddAuthenticationStateSerialization* to share identity from the prerender.
- **Used in**: ch07 — Part 1
- **Sources**: ASP.NET Core Blazor render modes — *Interactive WebAssembly* — https://learn.microsoft.com/aspnet/core/blazor/components/render-modes

### Idempotency key
Client-supplied identifier (typically `Idempotency-Key` header) that lets a server safely de-duplicate retries of a non-idempotent operation. Persisted alongside the request hash and result for a TTL.
- **Used in**: ch02 §7
- **Sources**: IETF draft — *The Idempotency-Key HTTP Header Field* — https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/ ; Stripe API — *Idempotent Requests* — https://docs.stripe.com/api/idempotent_requests

### ITestOutputHelper
xUnit per-test logger injected into the test class constructor; writes only show up if the test fails (or `[assembly: CaptureConsole]` is on). Replaces `Console.WriteLine` in xUnit tests.
- **Used in**: ch04 §16
- **Sources**: xUnit.net — *Capturing Output* — https://xunit.net/docs/capturing-output ; xUnit v3 — *CaptureConsoleAttribute* — https://xunit.net/docs/getting-started/v3/whats-new

## J

### JIT (Just-in-Time compilation)
RyuJIT — the .NET runtime's IL→native compiler invoked the first time a method runs. Interacts with *Tiered Compilation*, *PGO*, and *R2R*; absent under *NativeAOT*.
- **Used in**: ch05 §8
- **Sources**: .NET — *RyuJIT* — https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/jit/ryujit-overview.md ; .NET runtime overview — https://learn.microsoft.com/dotnet/standard/clr

### JWT bearer
RFC 7519 JSON Web Token presented per *RFC 6750* in an `Authorization: Bearer <token>` header. ASP.NET Core authenticates via `AddJwtBearer`; the token's *scp*, *roles*, *azp/appid*, *ver* claims drive authorization policies.
- **Used in**: ch02 §10
- **Sources**: RFC 7519 — https://www.rfc-editor.org/rfc/rfc7519 ; RFC 6750 — https://www.rfc-editor.org/rfc/rfc6750 ; ASP.NET Core JWT bearer — https://learn.microsoft.com/aspnet/core/security/authentication/configure-jwt-bearer-authentication

## K

### Keyed services
.NET 8+ DI feature: register multiple implementations of the same service type under different keys (`AddKeyedSingleton<T>(key, …)`), resolve with `IServiceProvider.GetRequiredKeyedService<T>(key)` or *[FromKeyedServices]*.
- **Used in**: ch01 §5, ch02 §10
- **Sources**: ASP.NET Core 8 — *Keyed services* — https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection#keyed-services

### K8s QoS class (Guaranteed / Burstable / BestEffort)
Kubernetes scheduling/eviction tier derived from a Pod's resource requests/limits: Guaranteed = requests == limits on every container for cpu+memory; Burstable = at least one request set; BestEffort = none set. The default for production .NET in this guide is Guaranteed (predictable GC sizing under DATAS/CFS).
- **Used in**: ch06 §3
- **Sources**: Kubernetes — *Configure Quality of Service for Pods* — https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/ ; *Pod Quality of Service Classes* — https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/

### CFS quota (cgroup CPU bandwidth)
Linux Completely Fair Scheduler bandwidth-control mechanism: cgroup gets `cpu.cfs_quota_us` of CPU time per `cpu.cfs_period_us` window. Kubernetes CPU limits map to CFS quota; aggressive limits cause throttling spikes that show up as p99 latency on managed runtimes.
- **Used in**: ch06 §3
- **Sources**: Linux kernel — *CFS Bandwidth Control* — https://www.kernel.org/doc/html/latest/scheduler/sched-bwc.html ; Kubernetes — *CPU limits and CFS quota* — https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

## L

### [LoggerMessage]
Source-generator attribute (`Microsoft.Extensions.Logging`) that emits zero-allocation, strongly-typed logging methods at compile time. Replaces `_logger.LogInformation("…{X}…", x)` on hot paths.
- **Used in**: ch01 §10, ch02 §6, ch05 §12
- **Sources**: .NET — *Compile-time logging source generation* — https://learn.microsoft.com/dotnet/core/extensions/logger-message-generator

## M

### MAUI Shell
.NET MAUI navigation chassis (`Shell.xaml`) that declares routes, flyout menu, and tab structure declaratively; supports URI-based navigation (`Shell.Current.GoToAsync("//route?p=v")`).
- **Used in**: ch07 — Part 2
- **Sources**: .NET MAUI — *Shell overview* — https://learn.microsoft.com/dotnet/maui/fundamentals/shell/

### MAUI Window lifecycle
`Microsoft.Maui.Controls.Window` events (`Created`, `Activated`, `Deactivated`, `Stopped`, `Resumed`, `Destroying`, `Backgrounding`) and `MauiAppBuilder.ConfigureLifecycleEvents` per-platform hooks that drive state save/restore on mobile and desktop.
- **Used in**: ch07 — Part 2
- **Sources**: .NET MAUI — *App lifecycle* — https://learn.microsoft.com/dotnet/maui/fundamentals/app-lifecycle ; *Windows* — https://learn.microsoft.com/dotnet/maui/fundamentals/windows

### Memory<T>
Heap-storable counterpart of *Span<T>* (`System.Memory<T>`) that can survive `await` boundaries and be stored as a field. Pairs with *IMemoryOwner<T>* / `MemoryPool<T>` for pooled allocation.
- **Used in**: ch05 §2
- **Sources**: *Memory<T> and Span<T> usage guidelines* — https://learn.microsoft.com/dotnet/standard/memory-and-spans/memory-t-usage-guidelines

### MemoryDiagnoser
*BenchmarkDotNet* diagnoser that reports allocations and GC counts per benchmark. Required column for any benchmark in this guide that touches reference types.
- **Used in**: ch05 §1
- **Sources**: BenchmarkDotNet — *MemoryDiagnoser* — https://benchmarkdotnet.org/articles/configs/diagnosers.html

### Meter (System.Diagnostics.Metrics)
`System.Diagnostics.Metrics.Meter` — the first-class .NET metrics API, OpenTelemetry-shaped (counters, histograms, observable gauges, up-down counters). Default for new instrumentation; preferred over *EventCounters*.
- **Used in**: ch05 §1, ch06 §5
- **Sources**: .NET — *Metrics API* — https://learn.microsoft.com/dotnet/core/diagnostics/metrics ; OpenTelemetry .NET — Metrics API — https://opentelemetry.io/docs/languages/net/

### Migration bundle
EF Core 7+ `dotnet ef migrations bundle` output: a self-contained executable (`efbundle.exe`) embedding a migrator + the migrations. Run as a deploy step; no `dotnet ef` SDK needed at runtime, no *Database.Migrate* on app start.
- **Used in**: ch03 §4
- **Sources**: EF Core 7 — *Migration bundles* — https://learn.microsoft.com/ef/core/managing-schemas/migrations/applying#bundles

### Minimal API
ASP.NET Core endpoint style: `app.MapGet("/x", handler)` with parameter-binding inferred from delegate signature. Default for new microservices in this guide; controllers remain valid for large MVC surfaces, gRPC stays separate.
- **Used in**: ch02 §1
- **Sources**: ASP.NET Core — *Minimal APIs overview* — https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview

### MTP (Microsoft.Testing.Platform)
Replacement test-host platform for .NET test projects: tests compile to a self-executable, no `vstest.console`, native to `dotnet test`'s newer `--` arg passthrough, faster startup, native AOT-friendly. Default for xUnit v3, MSTest 3.6+, NUnit 4+, TUnit.
- **Used in**: ch04 §2
- **Sources**: .NET — *Microsoft.Testing.Platform* — https://learn.microsoft.com/dotnet/core/testing/microsoft-testing-platform-intro

## N

### NativeAOT
.NET publish mode (`<PublishAot>true</PublishAot>`) that compiles the app to a single native binary with no *JIT* and aggressive trimming. Smallest startup, smallest memory; restricted reflection, no dynamic codegen, no `Assembly.LoadFrom`. Use for CLI, serverless, sidecars; not the default for ASP.NET services.
- **Used in**: ch01 §11, ch05 §8, ch06 §2
- **Sources**: .NET — *Native AOT deployment* — https://learn.microsoft.com/dotnet/core/deploying/native-aot/ ; Limitations — https://learn.microsoft.com/dotnet/core/deploying/native-aot/limitations

### NoGCRegion
`GC.TryStartNoGCRegion(totalSize)` API that asks the GC to defer all collections until a budget is exhausted. Reserved for tight latency windows (matching engine, RT loop); failure to fit budget falls back to normal GC.
- **Used in**: ch05 §9
- **Sources**: `GC.TryStartNoGCRegion` — https://learn.microsoft.com/dotnet/api/system.gc.trystartnogcregion

### NRT (Nullable Reference Types)
C# 8+ feature, on by default in templates, that promotes `T` (non-nullable) and `T?` (nullable) into the type system enforced by the compiler. `<Nullable>enable</Nullable>` is mandatory in this guide.
- **Used in**: ch01 §3
- **Sources**: C# — *Nullable reference types* — https://learn.microsoft.com/dotnet/csharp/nullable-references

## O

### OpenAPI
HTTP API description spec (formerly Swagger). ASP.NET Core 9+ ships `Microsoft.AspNetCore.OpenApi` (`AddOpenApi` / `MapOpenApi`) generating OpenAPI 3.x from minimal endpoints/controllers — replaces Swashbuckle for new projects.
- **Used in**: ch02 §5
- **Sources**: ASP.NET Core — *OpenAPI support* — https://learn.microsoft.com/aspnet/core/fundamentals/openapi/overview ; OpenAPI Specification — https://spec.openapis.org/oas/latest

### OTel SDK (.NET)
`OpenTelemetry` NuGet packages implementing the OpenTelemetry spec for traces, metrics, logs in .NET. The single observability SDK in this guide; exports via OTLP — never multiple exporters per signal.
- **Used in**: ch05 §1, ch06 §5
- **Sources**: OpenTelemetry .NET — https://opentelemetry.io/docs/languages/net/ ; .NET observability with OpenTelemetry — https://learn.microsoft.com/dotnet/core/diagnostics/observability-with-otel

### OTEL_EXPORTER_OTLP_ENDPOINT
Standard OpenTelemetry environment variable selecting the OTLP exporter target. Aspire's *ServiceDefaults* reads it; production K8s sets it to the in-cluster collector. Per-signal overrides exist (`…_TRACES_ENDPOINT`, `…_METRICS_ENDPOINT`, `…_LOGS_ENDPOINT`).
- **Used in**: ch06 §5
- **Sources**: OpenTelemetry — *Environment Variable Specification* — https://opentelemetry.io/docs/specs/otel/protocol/exporter/ ; *OTLP Exporter* — https://opentelemetry.io/docs/specs/otlp/

### OTLP exporter
OpenTelemetry Protocol exporter (gRPC or HTTP/protobuf) — the only wire protocol promoted to first-class by the OpenTelemetry project. Send everything to a collector, fan out from there.
- **Used in**: ch06 §5
- **Sources**: OpenTelemetry — *OTLP Specification* — https://opentelemetry.io/docs/specs/otlp/

### Outbox (Transactional Outbox)
Reliable-messaging pattern: an event row is `INSERT`ed into an `outbox` table inside the same DB transaction as the business write; a relay publishes pending rows to the broker and marks them sent. Eliminates dual-write inconsistency without 2PC.
- **Used in**: ch03 §6, ch06 §7
- **Sources**: microservices.io — *Transactional outbox* — https://microservices.io/patterns/data/transactional-outbox.html ; Chris Richardson, *Microservices Patterns* (Manning)

### OutputCache
ASP.NET Core 7+ middleware (`AddOutputCache` / `CacheOutput`) that caches the full response of an endpoint keyed by route/headers/query, with policy-based variation, eviction tags, and an `IOutputCacheStore` abstraction.
- **Used in**: ch02 §9
- **Sources**: ASP.NET Core — *Output caching middleware* — https://learn.microsoft.com/aspnet/core/performance/caching/output

### Owned entity
EF Core mapping where one entity type's lifetime is bound to its owner; rows live in the owner's table (default) or a side table. Largely replaced by *complex type* in EF 8+ for value-object scenarios.
- **Used in**: ch03 §2
- **Sources**: EF Core — *Owned Entity Types* — https://learn.microsoft.com/ef/core/modeling/owned-entities

## P

### params collections (C# 13)
C# 13 extension to the `params` modifier letting it bind to any collection type (`Span<T>`, `IEnumerable<T>`, `ReadOnlySpan<T>`, custom builders) in addition to `T[]`. Enables zero-alloc varargs.
- **Used in**: ch01 §2
- **Sources**: C# 13 — *params collections* — https://learn.microsoft.com/dotnet/csharp/language-reference/proposals/csharp-13.0/params-collections

### [PersistentState]
Razor component property attribute (`Microsoft.AspNetCore.Components`) that declares the property's value should be serialized into the prerendered HTML and re-hydrated on the interactive client, removing the second fetch on transition from SSR to *Interactive Server* / *Interactive WebAssembly* / *Interactive Auto*.
- **Used in**: ch07 — Part 1
- **Sources**: ASP.NET Core Blazor — *Prerender and integrate ASP.NET Core Razor components* — https://learn.microsoft.com/aspnet/core/blazor/components/prerender

### PDB (PodDisruptionBudget)
Kubernetes object capping voluntary disruptions (drain, upgrade) so at least N pods of a workload remain available. Required for any production .NET deployment with > 1 replica behind an SLO.
- **Used in**: ch06 §3
- **Sources**: Kubernetes — *Specifying a Disruption Budget* — https://kubernetes.io/docs/tasks/run-application/configure-pdb/

### PGO (Profile-Guided Optimization)
.NET *JIT* feature that uses runtime profile data to drive devirtualization, inlining, and hot/cold splitting.
- **Static PGO** ships profile data with the app (collected via `dotnet pgo`/crossgen2).
- **Dynamic PGO** profiles tier-0 in-process and recompiles tier-1 with that profile; on by default in .NET 8+.
- **Used in**: ch05 §8
- **Sources**: Performance Improvements in .NET 8 — *Dynamic PGO* — https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/ ; .NET — *Profile-guided optimization* — https://learn.microsoft.com/dotnet/core/whats-new/dotnet-8/runtime#dynamic-pgo

### Pipelines (System.IO.Pipelines)
`PipeReader` / `PipeWriter` API for high-throughput byte-stream processing without `Stream` allocations. Powers Kestrel, SignalR, gRPC; reach for it when parsing protocols off raw sockets.
- **Used in**: ch05 §10
- **Sources**: .NET — *System.IO.Pipelines* — https://learn.microsoft.com/dotnet/standard/io/pipelines

### PooledConnectionLifetime
`SocketsHttpHandler.PooledConnectionLifetime` — TimeSpan after which a pooled HTTP connection is recycled. Set to a finite value (e.g., 2–5 min) so DNS changes propagate; *IHttpClientFactory* sets a default but typed clients should still tune it.
- **Used in**: ch02 §11, ch05 §10
- **Sources**: `SocketsHttpHandler.PooledConnectionLifetime` — https://learn.microsoft.com/dotnet/api/system.net.http.socketshttphandler.pooledconnectionlifetime ; *Guidelines for using HttpClient* — https://learn.microsoft.com/dotnet/fundamentals/networking/http/httpclient-guidelines

### preStop hook
Kubernetes container lifecycle hook executed before SIGTERM, used to drain (`/healthz/ready` → unhealthy, `sleep`) so the service mesh / load balancer stops sending new requests before the .NET host stops listening. Pair with `terminationGracePeriodSeconds` ≥ preStop + max in-flight request time.
- **Used in**: ch06 §11
- **Sources**: Kubernetes — *Container Lifecycle Hooks* — https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/ ; *Termination of Pods* — https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination

### Primary constructors (C#)
C# 12 syntax for declaring constructor parameters directly on a `class` / `struct` declaration; parameters are in scope for the body and initializers but are **not** auto-properties (use a `record` for that). Most useful for DI in ASP.NET Core service classes.
- **Used in**: ch01 §2
- **Sources**: C# 12 — *Primary constructors* — https://learn.microsoft.com/dotnet/csharp/whats-new/tutorials/primary-constructors

### PriorityClass
Kubernetes object assigning a priority value (and `preemptionPolicy`) to Pods. Higher-priority Pods can preempt lower-priority ones during scheduling pressure. Reserved for control-plane / latency-critical .NET workloads.
- **Used in**: ch06 §3
- **Sources**: Kubernetes — *Pod Priority and Preemption* — https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/

### ProblemDetails
RFC 9457 (obsoletes RFC 7807) JSON/XML media type (`application/problem+json`) for machine-readable error responses. ASP.NET Core's `AddProblemDetails` + *IExceptionHandler* are the default mechanism for HTTP errors in this guide.
- **Used in**: ch02 §3
- **Sources**: RFC 9457 — *Problem Details for HTTP APIs* — https://www.rfc-editor.org/rfc/rfc9457 ; ASP.NET Core — *Handle errors → ProblemDetails* — https://learn.microsoft.com/aspnet/core/fundamentals/error-handling#problem-details

## R

### R2R (ReadyToRun)
Ahead-of-time IL→native compilation that ships precompiled code alongside the IL; the JIT can still recompile at runtime (so PGO/tiering still apply). Reduces startup and first-request latency. Enable via `<PublishReadyToRun>true</PublishReadyToRun>`.
- **Used in**: ch05 §8
- **Sources**: .NET — *ReadyToRun* — https://learn.microsoft.com/dotnet/core/deploying/ready-to-run

### RecyclableMemoryStream
Microsoft library (`Microsoft.IO.RecyclableMemoryStream`) — pooled `MemoryStream` substitute that buffers across an *ArrayPool*. Replaces `new MemoryStream()` per request when sizes are large or rates are high.
- **Used in**: ch05 §2
- **Sources**: Microsoft.IO.RecyclableMemoryStream — https://github.com/microsoft/Microsoft.IO.RecyclableMemoryStream

### RetainVM
`GCSettings.LatencyMode` interaction: when `<GCRetainVM>` is set in runtimeconfig, the GC keeps virtual address segments after collection rather than returning them to the OS — trades RSS for fewer page-faults / mmap churn under spiky workloads.
- **Used in**: ch05 §9
- **Sources**: .NET — *Runtime configuration: GCRetainVM* — https://learn.microsoft.com/dotnet/core/runtime-config/garbage-collector#retain-vm

### RFC 6750
"The OAuth 2.0 Authorization Framework: Bearer Token Usage". Defines `Authorization: Bearer <token>` and the `WWW-Authenticate: Bearer error=…` response used by *JWT bearer* and *claims-challenge*.
- **Used in**: ch02 §10
- **Sources**: RFC 6750 — https://www.rfc-editor.org/rfc/rfc6750

### RFC 9457
"Problem Details for HTTP APIs". Replaces RFC 7807; format used by ASP.NET Core *ProblemDetails*. Cite this — not 7807 — in new docs.
- **Used in**: ch02 §3
- **Sources**: RFC 9457 — https://www.rfc-editor.org/rfc/rfc9457

### Roles claim
Microsoft Entra app-role membership claim (`roles`) emitted in tokens for users/apps assigned an app role you defined on the API's app registration. Use for **app-permission** authorization (`[Authorize(Roles="Admin")]`); contrast *scp* (delegated).
- **Used in**: ch02 §10
- **Sources**: Microsoft identity platform — *App roles* — https://learn.microsoft.com/entra/identity-platform/howto-add-app-roles-in-apps ; *Access token claims reference* — https://learn.microsoft.com/entra/identity-platform/access-token-claims-reference

## S

### scp (claim)
Delegated-permission scope claim (`scp`, space-separated) emitted by Microsoft Entra when the token represents a user-delegated grant. Use for **delegated** authorization; contrast *roles* (app-only).
- **Used in**: ch02 §10
- **Sources**: Microsoft identity platform — *Access token claims reference* — https://learn.microsoft.com/entra/identity-platform/access-token-claims-reference

### Server GC
.NET garbage collector mode optimized for throughput on multi-core servers: per-CPU heaps, parallel collection, larger gen-0. Default for ASP.NET Core; tuned further by *DATAS* in .NET 8+. Contrast *Workstation GC*.
- **Used in**: ch05 §9, ch06 §3
- **Sources**: .NET — *Workstation and server garbage collection* — https://learn.microsoft.com/dotnet/standard/garbage-collection/workstation-server-gc

### Service discovery (`https+http://`, `_endpoint`)
.NET Aspire / `Microsoft.Extensions.ServiceDiscovery` URL scheme: `https+http://catalog` resolves logical name `catalog` against configured providers (Aspire host, configuration, DNS SRV) and prefers `https`, falls back to `http`. `_endpoint` suffix selects a named endpoint (e.g., `catalog/_grpc`).
- **Used in**: ch06 §7
- **Sources**: .NET Aspire — *Service discovery* — https://learn.microsoft.com/dotnet/aspire/service-discovery/overview ; Aspire docs — Service discovery — https://aspire.dev/

### ServiceDefaults (.NET Aspire)
Shared library project (`*.ServiceDefaults`) referenced by every service in an Aspire solution; calls `AddServiceDefaults()` to wire OpenTelemetry, health checks, *service discovery*, default *AddStandardResilienceHandler*. Customise once, apply everywhere.
- **Used in**: ch06 §1
- **Sources**: .NET Aspire — *Service defaults* — https://learn.microsoft.com/dotnet/aspire/fundamentals/service-defaults

### SetApplicationName
`IDataProtectionBuilder.SetApplicationName(string)` — discriminator written into the *Data Protection key ring* envelope. Must be identical across replicas of one app and **distinct** across different apps that share a key ring; otherwise replicas can't decrypt each other's cookies/antiforgery tokens.
- **Used in**: ch06 §9
- **Sources**: ASP.NET Core Data Protection — *Configuration: SetApplicationName* — https://learn.microsoft.com/aspnet/core/security/data-protection/configuration/overview#setapplicationname

### SocketsHttpHandler
Cross-platform managed `HttpMessageHandler` (`System.Net.Http.SocketsHttpHandler`) — the substrate `HttpClient` uses by default since .NET Core 2.1. Owns the connection pool, HTTP/2/3 selection, *PooledConnectionLifetime*. Configure it through *IHttpClientFactory*, not directly.
- **Used in**: ch02 §11, ch05 §10
- **Sources**: `SocketsHttpHandler` — https://learn.microsoft.com/dotnet/api/system.net.http.socketshttphandler ; *Guidelines for using HttpClient* — https://learn.microsoft.com/dotnet/fundamentals/networking/http/httpclient-guidelines

### Source generators
Roslyn-driven compile-time code generation (`IIncrementalGenerator`). Powers *[LoggerMessage]*, `System.Text.Json` source-gen serializer, Regex source-gen, gRPC, *NativeAOT*-friendly DI. Default mechanism for replacing reflection on hot paths.
- **Used in**: ch01 §10, ch05 §6
- **Sources**: .NET — *Source generators overview* — https://learn.microsoft.com/dotnet/csharp/roslyn-sdk/source-generators-overview

### Span<T>
`System.Span<T>` — stack-only ref struct over contiguous memory (array, native, stack, slice). Cannot cross `await` boundaries or live on the heap; pair with *Memory<T>* for that.
- **Used in**: ch05 §2, ch05 §3
- **Sources**: *Memory<T> and Span<T> usage guidelines* — https://learn.microsoft.com/dotnet/standard/memory-and-spans/memory-t-usage-guidelines

### [SupplyParameterFromForm]
Razor component parameter attribute that binds a value from the posted form, named via *FormName*. The Blazor SSR equivalent of MVC model binding.
- **Used in**: ch07 — Part 1
- **Sources**: ASP.NET Core Blazor forms — *SupplyParameterFromForm* — https://learn.microsoft.com/aspnet/core/blazor/forms/

### System.Threading.Lock (.NET 9)
Dedicated mutual-exclusion type (`System.Threading.Lock`) that `lock(x)` uses an optimised path for. Replaces locking on `private readonly object _gate = new();` patterns — declare `private readonly Lock _gate = new();` instead.
- **Used in**: ch01 §2, ch05 §13
- **Sources**: .NET 9 — *System.Threading.Lock* — https://learn.microsoft.com/dotnet/api/system.threading.lock ; What's new in C# 13 — *new lock object* — https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-13#new-lock-object

## T

### Testcontainers
.NET library (`Testcontainers` — Atomic Jar) launching ephemeral Docker containers (Postgres, Redis, Kafka, etc.) inside integration tests via Docker / Podman / TestContainers Cloud. Default for "real dependency" integration tests in this guide.
- **Used in**: ch04 §6
- **Sources**: Testcontainers for .NET — https://dotnet.testcontainers.org/ ; Testcontainers — https://testcontainers.com/

### TFM (Target Framework Moniker)
`<TargetFramework>` value (`net10.0`, `net9.0-android`, `netstandard2.0`) selecting the BCL / runtime / OS-API surface a project compiles against. New code in this guide targets `net10.0` (or current LTS) and uses *CPM* to pin matching package versions.
- **Used in**: ch01 §1
- **Sources**: .NET — *Target frameworks in SDK-style projects* — https://learn.microsoft.com/dotnet/standard/frameworks

### Tiered Compilation
.NET *JIT* strategy that compiles methods quickly at tier-0, then recompiles hot methods at tier-1 with full optimisations and *Dynamic PGO*. On by default since .NET Core 3.0.
- **Used in**: ch05 §8
- **Sources**: .NET — *Tiered compilation* — https://learn.microsoft.com/dotnet/standard/runtime/tiered-compilation

## V

### ValueTask / ValueTask<T>
`System.Threading.Tasks.ValueTask` — value-typed task return that avoids the `Task` allocation when the operation completes synchronously or is pooled. Use only on hot async APIs that often complete synchronously; consume **once**, no `.Result`, no double-await.
- **Used in**: ch01 §4, ch05 §7
- **Sources**: Stephen Toub — *Understanding the Whys, Whats, and Whens of ValueTask* — https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/

### Vector512<T>
`System.Runtime.Intrinsics.Vector512<T>` — 512-bit SIMD vector type lowered to *AVX-512* (or the best available instruction set) by RyuJIT. Default abstraction for portable vectorised loops in .NET 8+.
- **Used in**: ch05 §13
- **Sources**: `Vector512<T>` — https://learn.microsoft.com/dotnet/api/system.runtime.intrinsics.vector512-1 ; .NET 8 hardware intrinsics — https://devblogs.microsoft.com/dotnet/dotnet-8-hardware-intrinsics/

### ver (claim)
Microsoft Entra token-version claim (`ver` = `1.0` or `2.0`) telling the resource server which token contract to expect (different claim shapes; `appid` vs `azp`, `oid` semantics, etc.). Validate it before parsing other claims.
- **Used in**: ch02 §10
- **Sources**: Microsoft identity platform — *Access tokens — payload claims* — https://learn.microsoft.com/entra/identity-platform/access-token-claims-reference ; *Token versions* — https://learn.microsoft.com/entra/identity-platform/access-tokens

### Verify
Snapshot/approval testing library (`Verify.Xunit`, `Verify.MSTest`, `Verify.NUnit`, `Verify.TUnit`). Compares serialized output of the system-under-test against committed `.verified.*` files; diff tools auto-launch on mismatch.
- **Used in**: ch04 §7
- **Sources**: Verify — https://github.com/VerifyTests/Verify

### VSTest
Legacy Visual Studio test runtime that hosts test adapters (`vstest.console`, `dotnet test`'s default before *MTP*). Still required for old projects and for some IDE integrations; new test projects should opt into *MTP* via `<EnableMSTestRunner>true</EnableMSTestRunner>` / xUnit v3 / TUnit defaults.
- **Used in**: ch04 §2
- **Sources**: VSTest — https://github.com/microsoft/vstest ; .NET — *Microsoft.Testing.Platform vs VSTest* — https://learn.microsoft.com/dotnet/core/testing/microsoft-testing-platform-vs-vstest

## W

### WebApplicationFactory<TEntryPoint>
ASP.NET Core test host (`Microsoft.AspNetCore.Mvc.Testing`) that boots the real `Program` in-process, returning a `HttpClient` bound to a `TestServer`. The integration-test entry point for any HTTP surface in this guide; combine with *Testcontainers* for real backing services and *ICollectionFixture* for sharing.
- **Used in**: ch04 §5
- **Sources**: ASP.NET Core — *Integration tests with WebApplicationFactory* — https://learn.microsoft.com/aspnet/core/test/integration-tests

### Workstation GC
.NET garbage-collector mode tuned for client/UI processes: single heap, lower memory ceiling, optimised for responsiveness over throughput. Default for desktop/MAUI; do **not** select for ASP.NET services — use *Server GC* + *DATAS*.
- **Used in**: ch05 §9
- **Sources**: .NET — *Workstation and server garbage collection* — https://learn.microsoft.com/dotnet/standard/garbage-collection/workstation-server-gc

## X

### xUnit v3
Major rewrite of the xUnit.net test framework: native *MTP* runner, new `Assert` improvements, project-as-executable model, `[assembly: CaptureConsole]`, async lifetime fixes. Default test framework in this guide.
- **Used in**: ch04 §2
- **Sources**: xUnit.net — *What's new in v3* — https://xunit.net/docs/getting-started/v3/whats-new ; xUnit.net — https://xunit.net/

### [assembly: CaptureConsole]
xUnit v3 assembly attribute that redirects `Console.Out` / `Console.Error` written during a test into the test's *ITestOutputHelper* sink, so third-party libraries that log to the console show up in failed-test output.
- **Used in**: ch04 §16
- **Sources**: xUnit v3 — *Capturing Output / CaptureConsole* — https://xunit.net/docs/capturing-output
