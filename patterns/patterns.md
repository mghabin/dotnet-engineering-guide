# Patterns index

Thin index of named patterns used across this guide.
Each entry is a one-paragraph summary with a link to the chapter section that owns the decision.
Decision-bearing prose lives in the chapters; this page only points there.

> Companion pages: [`monorepo.md`](./monorepo.md), [`anti-patterns.md`](./anti-patterns.md), shared [`references.md`](./references.md).

---

## Foundations

- **DI & lifetimes, keyed services** — Compose at the host, resolve via the container, prefer `GetRequiredKeyedService<T>` over hand-rolled factories.
  See [docs/01-foundations.md §5 DI & lifetimes](../docs/01-foundations.md#5-di--lifetimes).
- **Options pattern, validated, fail-fast** — Bind config sections to typed options, validate on start, prefer the `[OptionsValidator]` source generator.
  See [docs/01-foundations.md §6 Configuration & options pattern](../docs/01-foundations.md#6-configuration--options-pattern).
- **Source generators for hot paths** — `LoggerMessage`, `JsonSerializable`, `GeneratedRegex`, configuration binder, OpenAPI — all eliminate startup reflection and are AOT-safe.
  See [docs/01-foundations.md §10 Source generators](../docs/01-foundations.md#10-source-generators).
- **`IAsyncDisposable` discipline** — Use `await using` for async resources; never mix sync `Dispose` with async resources or shutdown blocks.
  See [docs/01-foundations.md §7 Disposal](../docs/01-foundations.md#7-disposal).
- **Result types for expected failures** — Exceptions for *exceptional* (programmer error, infra failure), `Result<T,E>` for *expected* (validation, business rules).
  See [docs/01-foundations.md §8 Exceptions](../docs/01-foundations.md#8-exceptions).
- **Immutability & DTO design, value objects, strongly-typed IDs** — `record` for DTOs, `readonly record struct` for hot-path values, source-generated wrappers (Vogen / StronglyTypedId) for primitive obsession.
  See [docs/01-foundations.md §9 Immutability & DTO design](../docs/01-foundations.md#9-immutability--dto-design).
- **`TimeProvider` in production code** — Inject `TimeProvider` (never `DateTime.UtcNow` directly) so handlers, schedulers, and token validation are testable.
  See [docs/01-foundations.md §5 DI & lifetimes](../docs/01-foundations.md#5-di--lifetimes) and the testing counterpart [docs/04-testing.md §14 Time](../docs/04-testing.md#time--use-timeprovider-net-8).

## ASP.NET Core

- **`ProblemDetails` + `IExceptionHandler`** — Standardise error responses via `AddProblemDetails`; chain multiple `IExceptionHandler`s instead of a single catch-all middleware.
  See [docs/02-aspnetcore.md §3 ProblemDetails & Error Handling](../docs/02-aspnetcore.md#3-problemdetails--error-handling).
- **Resilience pipeline (Polly v8)** — Use the standard resilience handler on `HttpClient`; do not hand-roll retry/circuit-breaker loops.
  See [docs/02-aspnetcore.md §7 Resilience](../docs/02-aspnetcore.md#7-resilience) and [docs/06-cloud-native.md §6 Resilience](../docs/06-cloud-native.md#6-resilience--polly-v8-standard-pipelines-not-hand-rolled-retries).
- **Output caching** — Prefer the `OutputCache` middleware for response-shaped reuse; data-shaped reuse belongs in the data caching matrix below.
  See [docs/02-aspnetcore.md §9 Output Caching](../docs/02-aspnetcore.md#9-output-caching).
- **`BackgroundService` + `Channel<T>`** — Long-running in-proc work runs as `BackgroundService`; use bounded `Channel<T>` (FullMode = Wait) for backpressure between producer and worker.
  See [docs/02-aspnetcore.md §12 Background Work](../docs/02-aspnetcore.md#12-background-work).
- **Chain of Responsibility = ASP.NET Core middleware** — `app.UseX()` is the canonical CoR pipeline; for exceptions specifically, register multiple `IExceptionHandler`s.
  See [docs/02-aspnetcore.md §3 ProblemDetails & Error Handling](../docs/02-aspnetcore.md#3-problemdetails--error-handling).

## Data

- **Repository — only for aggregate roots** — One repository per aggregate root (Evans / Vernon DDD), not per table; `IOrderRepository`, never `IOrderLineRepository`.
  See [docs/03-data.md §11 Repository / Specification](../docs/03-data.md#11-repository--specification).
- **Unit of Work = `DbContext`** — EF Core's `ChangeTracker` *is* the Unit of Work; do not wrap it in a custom `IUnitOfWork`. Share the scoped `DbContext` for cross-repo transactional behaviour.
  See [docs/03-data.md §6 Transactions & Unit of Work](../docs/03-data.md#6-transactions--unit-of-work).
- **Specification pattern** — Reusable EF query criteria via Ardalis.Specification; only adopt when ≥3 places ask the same question.
  See [docs/03-data.md §11 Repository / Specification](../docs/03-data.md#11-repository--specification).
- **CQRS only where asymmetry exists** — Adopt read-model projections only when reads and writes differ in scaling, storage, or consistency; otherwise it is theatre.
  See [docs/03-data.md §12 CQRS-lite](../docs/03-data.md#12-cqrs-lite).
- **Outbox + idempotent handlers** — Never publish from inside `SaveChangesAsync`; write to an outbox in the same transaction and dispatch from a relay. Idempotency keys on every external command.
  See [docs/03-data.md §6 Transactions & Unit of Work — Cross-system consistency → Outbox](../docs/03-data.md#cross-system-consistency--outbox).
- **Caching matrix (data side)** — Pick the right cache (in-memory, hybrid, distributed) per access pattern; data caches are not the same decision as `OutputCache`.
  See [docs/03-data.md §14 Caching](../docs/03-data.md#14-caching) and the response-shape counterpart [docs/02-aspnetcore.md §9 Output Caching](../docs/02-aspnetcore.md#9-output-caching).

## Testing

- **`WebApplicationFactory<TProgram>` integration tests** — One spine per app, override `IExternalApi` with WireMock, swap auth via a `TestAuthHandler`.
  See [docs/04-testing.md §5 WebApplicationFactory for ASP.NET Core](../docs/04-testing.md#5-webapplicationfactory-for-aspnet-core).
- **Testcontainers** — Real Postgres / SQL Server / Redis in Docker per fixture; faster and more correct than in-memory providers.
  See [docs/04-testing.md §6 Testcontainers](../docs/04-testing.md#6-testcontainers).
- **`TimeProvider` / `FakeTimeProvider` in tests** — Drive time deterministically with `FakeTimeProvider.Advance`; never `Thread.Sleep`.
  See [docs/04-testing.md §14 Time](../docs/04-testing.md#time--use-timeprovider-net-8).
- **Snapshot / approval testing** — Use Verify for response-shape and DI-graph snapshots; commit `*.verified.txt`.
  See [docs/04-testing.md §7 Snapshot / approval testing with Verify](../docs/04-testing.md#7-snapshot--approval-testing-with-verify).
- **Architecture tests** — NetArchTest / ArchUnitNET to enforce layer boundaries (e.g. Domain has no EF Core dependency).
  See [docs/04-testing.md §9 Architecture tests](../docs/04-testing.md#9-architecture-tests).
- **Composition root + DI graph test** — Resolve every registered service through the test host to fail fast on missing or cyclical registrations.
  See [docs/04-testing.md §5 WebApplicationFactory for ASP.NET Core](../docs/04-testing.md#5-webapplicationfactory-for-aspnet-core).

## Cloud-native

- **Aspire `ServiceDefaults`** — Shared OTel, health-check, resilience, and discovery wire-up; every service references the same `ServiceDefaults` project.
  See [docs/06-cloud-native.md §1 .NET Aspire](../docs/06-cloud-native.md#1-net-aspire--what-it-is-what-it-isnt).
- **Health checks — three endpoints** — `/health/startup`, `/health/live`, `/health/ready` mapped to K8s probes; what `MapDefaultEndpoints` ships is *not* enough.
  See [docs/06-cloud-native.md §10 Health checks](../docs/06-cloud-native.md#10-health-checks--three-endpoints-for-k8s-not-what-servicedefaults-gives-you).
- **Graceful shutdown** — Set `HostOptions.ShutdownTimeout` larger than the longest in-flight request; drain on `ApplicationStopping`, not on `StopAsync`.
  See [docs/06-cloud-native.md §11 Graceful shutdown](../docs/06-cloud-native.md#11-graceful-shutdown--drain-dont-drop).
- **Telemetry-by-construction** — Every cross-boundary type takes `ILogger<T>`, owns an `ActivitySource`, emits a `Meter`; bake it in, do not retrofit after incidents.
  See [docs/06-cloud-native.md §5 Observability](../docs/06-cloud-native.md#5-observability--opentelemetry-one-sdk-three-signals).

---

## Cross-cutting (no single chapter owner)

These do not map cleanly onto one chapter and stay here.

### Vertical slice vs Clean Architecture

- **Vertical slice** ([Jimmy Bogard](https://www.jimmybogard.com/vertical-slice-architecture/)): organise by feature folder; each slice owns its request, handler, validator, DTO. Best where features evolve independently.
- **Clean Architecture** ([Steve Smith template](https://github.com/ardalis/CleanArchitecture)): organise by layer (Domain / Application / Infrastructure / Web). Best for stable, complex domains with cross-cutting policies.
- Hybrid is fine: Clean at the assembly level, vertical-slice within `Application`. Pick a *primary* axis.
- **Don't** mix the two axes at the same level — it produces a folder tree nobody can navigate.

### Bounded contexts & monorepo strategy

- One bounded context per deployable; one solution per bounded context inside the monorepo.
- Cross-context calls go over the wire (HTTP / messaging) — never via a shared `Domain` project.
- See [`monorepo.md`](./monorepo.md) for the repository layout rules.

### Domain events vs integration events

- Domain events fire inside the aggregate boundary and dispatch in-process after `SaveChanges` (see [docs/03-data.md §6](../docs/03-data.md#6-transactions--unit-of-work)).
- Integration events leave the process via the outbox.
- **Don't** conflate the two — the failure modes differ and one becomes the other only by crossing the outbox.

---

## GoF refresher (.NET-idiomatic)

A short pointer list — each pattern's idiomatic .NET-10 form, and which BCL primitive replaces the hand-roll.
Use this as a vocabulary cheat-sheet, not as a design checklist.

- **Strategy** — Inject the algorithm via DI; pick at runtime with `IServiceProvider.GetRequiredKeyedService<T>` or `[FromKeyedServices("name")]`. See [docs/01-foundations.md §5](../docs/01-foundations.md#5-di--lifetimes).
- **Decorator** — Wrap an interface with [Scrutor](https://github.com/khellang/Scrutor) `Decorate`; AOT-safe alternative is a source generator. **Don't** use `DispatchProxy` under AOT.
- **Adapter** — Hand-write the adapter at the boundary; it documents the seam better than reflection mappers.
- **Factory / Abstract Factory** — `IServiceProvider.GetRequiredKeyedService<T>(key)` replaces most factory interfaces; reach for `Func<T, U>` only when the key is computed.
- **Builder** — The BCL already gives you `HttpRequestMessage`, `WebApplicationBuilder`, `HostBuilder`. Hand-roll a builder only for *your* domain construction.
- **Singleton** — Always via the DI container (`AddSingleton`); never `static readonly Foo Instance = new()` — the latter is untestable.
- **Observer** — `Channel<T>` (with backpressure) or MediatR notifications for in-proc fan-out; skip `IObservable<T>` unless you need Rx operators.
- **Mediator** — [MediatR](https://github.com/jbogard/MediatR) for in-proc dispatch; [Wolverine](https://wolverinefx.net/) when you also want messaging + outbox in the same primitive. Not a service locator.
- **Command** — Same shape as Mediator's request — the *write* side returns an ID or `Result<…>`, never query data back. See CQRS index entry.
- **Chain of Responsibility** — ASP.NET Core middleware *is* CoR; for exceptions, register multiple `IExceptionHandler`s. See [docs/02-aspnetcore.md §3](../docs/02-aspnetcore.md#3-problemdetails--error-handling).
- **Template Method** — `sealed override` on the framework hook + `abstract` on the customisation point. `BackgroundService.ExecuteAsync` is the canonical example.
- **Visitor** — C# pattern matching on a sealed hierarchy; mark the base `abstract` and all derivations `sealed` so the compiler warns on missing arms.
- **Specification** — See the data-chapter index entry above.
- **Null Object** — `NullLogger<T>.Instance` is the canonical BCL example; use it for tests and degenerate paths.
- **Memento** — Snapshot via `System.Text.Json` source-gen so the snapshot is a `byte[]`.
- **Composite** — The BCL already composes most things you need (`IConfiguration`, `LoggerProvider`, `CompositeFileProvider`).
- **Proxy** — Castle DynamicProxy on JIT only; AOT-safe alternative is a Roslyn source generator. Decorator-via-Scrutor covers most "logging around an interface" cases without proxies.
- **Flyweight** — `ArrayPool<T>`, `MemoryPool<T>`, `string.Intern` / `StringPool`. Hand-roll only after a profiler shows allocation pressure. See [docs/05-performance.md §2](../docs/05-performance.md#2-allocations--the-silent-latency-tax).
- **Bridge** — `ILogger` ↔ `ILoggerProvider` is the canonical .NET example; the abstraction and the implementation vary independently.
- **Iterator** — `yield return` for sync, `IAsyncEnumerable<T>` + `[EnumeratorCancellation]` for async. **Don't** materialise to `List<T>` unless the caller needs random access.
- **State** — [Stateless](https://github.com/dotnet-state-machine/stateless) for non-trivial workflows; an `enum` + `switch` is enough for two states — don't pull in a state-machine library.
- **Interpreter** — Roslyn, `System.Linq.Expressions`, EF Core's LINQ provider, and [Sprache](https://github.com/sprache/Sprache) cover the realistic cases. If you find yourself writing one, you probably want a typed YAML/JSON config instead.

---

## Anti-patterns

The negative counterpart to this index lives in [`anti-patterns.md`](./anti-patterns.md).

See [`references.md`](./references.md) for the curated source list.
