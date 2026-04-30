# Patterns index

This is an index. Each pattern is owned and detailed in the linked chapter section; this page only points there.

> Companion pages: [`monorepo.md`](./monorepo.md), [`anti-patterns.md`](./anti-patterns.md), shared [`references.md`](./references.md).

---

## Foundations

- **DI & lifetimes, keyed services** — see [chapter 01 §5](../docs/01-foundations.md#5-di--lifetimes).
- **Options pattern, validated, fail-fast** — see [chapter 01 §6](../docs/01-foundations.md#6-configuration--options-pattern).
- **Source generators for hot paths** — see [chapter 01 §10](../docs/01-foundations.md#10-source-generators).
- **`IAsyncDisposable` discipline** — see [chapter 01 §7](../docs/01-foundations.md#7-disposal).
- **Result types for expected failures** — see [chapter 01 §8](../docs/01-foundations.md#8-exceptions).
- **Immutability & DTO design, value objects, strongly-typed IDs** — see [chapter 01 §9](../docs/01-foundations.md#9-immutability--dto-design).
- **`TimeProvider` in production code** — see [chapter 01 §5](../docs/01-foundations.md#5-di--lifetimes); test-side counterpart [chapter 04 §14](../docs/04-testing.md#time--use-timeprovider-net-8).

## ASP.NET Core

- **`ProblemDetails` + `IExceptionHandler`** — see [chapter 02 §3](../docs/02-aspnetcore.md#3-problemdetails--error-handling).
- **Resilience pipeline (Polly v8)** — see [chapter 02 §7](../docs/02-aspnetcore.md#7-resilience) and [chapter 06 §6](../docs/06-cloud-native.md#6-resilience--see-ch02-7).
- **Output caching** — see [chapter 02 §9](../docs/02-aspnetcore.md#9-output-caching).
- **`BackgroundService` + `Channel<T>`** — see [chapter 02 §12](../docs/02-aspnetcore.md#12-background-work).
- **Chain of Responsibility = ASP.NET Core middleware** — see [chapter 02 §3](../docs/02-aspnetcore.md#3-problemdetails--error-handling).

## Data

- **Repository — only for aggregate roots** — see [chapter 03 §11](../docs/03-data.md#11-repository--specification).
- **Unit of Work = `DbContext`** — see [chapter 03 §6](../docs/03-data.md#6-transactions--unit-of-work).
- **Specification pattern** — see [chapter 03 §11](../docs/03-data.md#11-repository--specification).
- **CQRS-lite** — see [chapter 03 §12](../docs/03-data.md#12-cqrs-lite).
- **Outbox + idempotent handlers** — see [chapter 03 §6 → Cross-system consistency](../docs/03-data.md#cross-system-consistency--outbox).
- **Caching matrix (data side)** — see [chapter 03 §14](../docs/03-data.md#14-caching); response-shape counterpart [chapter 02 §9](../docs/02-aspnetcore.md#9-output-caching).

## Testing

- **`WebApplicationFactory<TProgram>` integration tests** — see [chapter 04 §5](../docs/04-testing.md#5-webapplicationfactory-for-aspnet-core).
- **Testcontainers** — see [chapter 04 §6](../docs/04-testing.md#6-testcontainers).
- **`TimeProvider` / `FakeTimeProvider` in tests** — see [chapter 04 §14](../docs/04-testing.md#time--use-timeprovider-net-8).
- **Snapshot / approval testing** — see [chapter 04 §7](../docs/04-testing.md#7-snapshot--approval-testing-with-verify).
- **Architecture tests** — see [chapter 04 §9](../docs/04-testing.md#9-architecture-tests).
- **Composition root + DI graph test** — see [chapter 04 §5](../docs/04-testing.md#5-webapplicationfactory-for-aspnet-core).

## Cloud-native

- **Aspire `ServiceDefaults`** — see [chapter 06 §1](../docs/06-cloud-native.md#1-net-aspire--what-it-is-what-it-isnt).
- **Health checks — three endpoints** — see [chapter 06 §10](../docs/06-cloud-native.md#10-health-checks--three-endpoints-for-k8s-not-what-servicedefaults-gives-you).
- **Graceful shutdown** — see [chapter 06 §11](../docs/06-cloud-native.md#11-graceful-shutdown--drain-dont-drop).
- **Telemetry-by-construction** — see [chapter 06 §5](../docs/06-cloud-native.md#5-observability--opentelemetry-one-sdk-three-signals).

---

## GoF refresher (.NET-idiomatic)

A short pointer list — each pattern's idiomatic .NET-10 form, and which BCL primitive replaces the hand-roll.
Vocabulary cheat-sheet, not a design checklist.

- **Strategy** — algorithm via DI; runtime pick with `IServiceProvider.GetRequiredKeyedService<T>` or `[FromKeyedServices("name")]`. See [chapter 01 §5](../docs/01-foundations.md#5-di--lifetimes).
- **Decorator** — wrap an interface with [Scrutor](https://github.com/khellang/Scrutor) `Decorate`; AOT-safe alternative is a source generator. `DispatchProxy` is JIT-only.
- **Adapter** — hand-written at the boundary; documents the seam better than reflection mappers.
- **Factory / Abstract Factory** — `IServiceProvider.GetRequiredKeyedService<T>(key)` replaces most factory interfaces; `Func<T, U>` when the key is computed.
- **Builder** — BCL ships `HttpRequestMessage`, `WebApplicationBuilder`, `HostBuilder`. Hand-roll for *your* domain construction.
- **Singleton** — DI container (`AddSingleton`); `static readonly Foo Instance = new()` is untestable.
- **Observer** — `Channel<T>` (with backpressure) or MediatR notifications for in-proc fan-out; `IObservable<T>` only when you need Rx operators.
- **Mediator** — [MediatR](https://github.com/jbogard/MediatR) for in-proc dispatch; [Wolverine](https://wolverinefx.net/) when you also want messaging + outbox in the same primitive. Not a service locator.
- **Command** — same shape as Mediator's request — the *write* side returns an ID or `Result<…>`, never query data back. See CQRS index entry.
- **Chain of Responsibility** — ASP.NET Core middleware *is* CoR; for exceptions, multiple `IExceptionHandler`s. See [chapter 02 §3](../docs/02-aspnetcore.md#3-problemdetails--error-handling).
- **Template Method** — `sealed override` on the framework hook + `abstract` on the customisation point. `BackgroundService.ExecuteAsync` is the canonical example.
- **Visitor** — C# pattern matching on a sealed hierarchy; mark the base `abstract` and all derivations `sealed` so the compiler warns on missing arms.
- **Specification** — see the data-chapter index entry above.
- **Null Object** — `NullLogger<T>.Instance` is the canonical BCL example.
- **Memento** — snapshot via `System.Text.Json` source-gen so the snapshot is a `byte[]`.
- **Composite** — BCL already composes most things you need (`IConfiguration`, `LoggerProvider`, `CompositeFileProvider`).
- **Proxy** — Castle DynamicProxy on JIT only; AOT-safe alternative is a Roslyn source generator. Decorator-via-Scrutor covers most "logging around an interface" cases.
- **Flyweight** — `ArrayPool<T>`, `MemoryPool<T>`, `string.Intern` / `StringPool`. See [chapter 05 §2](../docs/05-performance.md#2-allocations--the-silent-latency-tax).
- **Bridge** — `ILogger` ↔ `ILoggerProvider` is the canonical .NET example; abstraction and implementation vary independently.
- **Iterator** — `yield return` for sync, `IAsyncEnumerable<T>` + `[EnumeratorCancellation]` for async.
- **State** — [Stateless](https://github.com/dotnet-state-machine/stateless) for non-trivial workflows; an `enum` + `switch` is enough for two states.
- **Interpreter** — Roslyn, `System.Linq.Expressions`, EF Core's LINQ provider, and [Sprache](https://github.com/sprache/Sprache) cover the realistic cases.

---

## Anti-patterns

The negative counterpart to this index lives in [`anti-patterns.md`](./anti-patterns.md).

See [`references.md`](./references.md) for the curated source list.
