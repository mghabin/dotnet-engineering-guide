# Patterns to follow (and a GoF refresher in .NET 10 idiom)

Positive patterns that should replace the anti-patterns, plus a short
Gang-of-Four refresher framed for idiomatic .NET 10.

> Companion pages: [`monorepo.md`](./monorepo.md),
> [`anti-patterns.md`](./anti-patterns.md), shared [`references.md`](./references.md).

---


### Composition root + DI graph testing

Compose at the host (`Program.cs`); test the graph:

```csharp
[Fact]
public void Container_resolves_all_services()
{
    using var app = new WebApplicationFactory<Program>();
    using var scope = app.Services.CreateScope();
    foreach (var sd in app.Services.GetRequiredService<IServiceCollection>()
                            .Where(s => s.ServiceType.IsPublic && !s.ServiceType.IsGenericTypeDefinition))
    {
        _ = scope.ServiceProvider.GetRequiredService(sd.ServiceType);
    }
}
```

(In practice you snapshot the registered `IServiceCollection` at host build
time via a test seam.)

### Options pattern, validated, fail-fast

```csharp
services.AddOptions<GraphOptions>()
        .Bind(config.GetSection("Graph"))
        .ValidateDataAnnotations()
        .Validate(o => o.TenantId != Guid.Empty, "TenantId required")
        .ValidateOnStart();
```

Combine with the [`[OptionsValidator]` source
generator](https://learn.microsoft.com/dotnet/core/extensions/options-library-authors#validateoptionssourcegen)
to avoid reflection at startup.

### Source generators for hot paths

```csharp
[LoggerMessage(Level = LogLevel.Information, Message = "Token issued for {ClientId}")]
public static partial void TokenIssued(this ILogger logger, string clientId);

[JsonSerializable(typeof(TokenResponse))]
internal partial class AuthJsonContext : JsonSerializerContext;

[GeneratedRegex("^[A-Z]{2}[0-9]{6}$", RegexOptions.CultureInvariant)]
private static partial Regex AccountIdRegex();
```

ASP.NET Core also ships a [Configuration binder source
generator](https://learn.microsoft.com/dotnet/core/extensions/configuration-generator)
and an OpenAPI source generator (`Microsoft.AspNetCore.OpenApi` in .NET 9+) —
all eliminate reflection at startup, all are AOT-safe.

### `TimeProvider` for testable time

```csharp
public sealed class Clock(TimeProvider time)
{
    public DateTimeOffset Now => time.GetUtcNow();
}
```

Test with `FakeTimeProvider`:

```csharp
var time = new FakeTimeProvider(DateTimeOffset.Parse("2025-01-01Z"));
time.Advance(TimeSpan.FromHours(2));
```

### Outbox + idempotent handlers

For reliable messaging, never publish from inside `SaveChangesAsync` — write
to an outbox table in the same transaction, dispatch from a relay.
[Jimmy Bogard, *Reliable messaging without distributed
transactions*](https://www.jimmybogard.com/refactoring-towards-resilience-evaluating-coupling/).
Wolverine and MassTransit both ship transactional-outbox EF Core integrations
out of the box ([Wolverine
docs](https://wolverinefx.net/guide/durability/),
[MassTransit
docs](https://masstransit.io/documentation/configuration/middleware/outbox)).

Idempotency keys on every external command: hash `(message-id, handler)` and
short-circuit re-deliveries — exactly-once is a fairy tale, idempotent
at-least-once is the achievable goal.

### Vertical slice vs Clean Architecture

- **Vertical slice** ([Jimmy
  Bogard](https://www.jimmybogard.com/vertical-slice-architecture/)): organize
  by feature folder, each slice has its own request/handler/validator/dto;
  shared abstractions only when *real* duplication appears. Best for systems
  where features evolve independently.
- **Clean Architecture** ([Steve Smith's
  template](https://github.com/ardalis/CleanArchitecture)): organize by layer
  (Domain / Application / Infrastructure / Web). Best for stable, complex
  domains with many cross-cutting policies.

Hybrid is fine: Clean at the assembly level, vertical-slice within
`Application`. Pick the *primary* axis.

### CQRS only where asymmetry exists

If your reads and writes have the same shape, MediatR + handlers *is* CQRS
theatre — Khorikov's
[*CQRS without 
DTOs*](https://enterprisecraftsmanship.com/posts/types-of-cqrs/). Adopt
read-model projections only when the asymmetry is real: different scaling,
different storage, different consistency.

### Domain events vs integration events

Domain events fire inside the aggregate boundary, dispatched in-process after
`SaveChanges`. Integration events leave the process via the outbox. Don't
conflate; the failure modes differ.

### Result types for expected failures

```csharp
public Result<Order, OrderError> Place(Cart cart) =>
    cart.Items.Count == 0
        ? OrderError.Empty
        : new Order(cart);
```

Libraries: [`OneOf`](https://github.com/mcintyre321/OneOf),
[`ErrorOr`](https://github.com/amantinband/error-or),
[`FluentResults`](https://github.com/altmann/FluentResults). Use exceptions
for *exceptional* (programmer error, infrastructure failure), results for
*expected* (validation, business rule). Khorikov,
[*Functional C#: Handling failures, input
errors*](https://enterprisecraftsmanship.com/posts/functional-c-handling-failures-input-errors/).

### Strongly-typed IDs

```csharp
[ValueObject<Guid>] public partial struct OrderId;
```

[Vogen](https://github.com/SteveDunn/Vogen) and
[StronglyTypedId](https://github.com/andrewlock/StronglyTypedId) both source-
generate the boilerplate. Eliminates `new OrderRepository().Get(customerId)`-style
bugs.

### Value objects + `record struct`

For tiny immutable values that get passed around hot loops, `readonly record
struct Money(decimal Amount, string Currency)` gives you value equality with
no heap allocation.

### Specification pattern

[Ardalis.Specification](https://github.com/ardalis/Specification) for
reusable EF query criteria; only adopt when you have ≥3 places asking the
same question. One-off queries belong inline.

### Repository ONLY for aggregate roots

Per Eric Evans' DDD book and Vaughn Vernon's IDDD: one repository per
*aggregate root*, not per table. `IOrderRepository`, not `IOrderLineRepository`.

### Unit of Work via EF Core

`DbContext` is the unit of work. Don't wrap it. If you need cross-repo
transactional behaviour, share the `DbContext` (it's already scoped). For
domain events, use [SaveChanges
interceptors](https://learn.microsoft.com/ef/core/logging-events-diagnostics/interceptors#savechanges-interceptor)
to dispatch after a successful commit.

### `BackgroundService` + `Channel<T>`

```csharp
public sealed class EmailQueue
{
    private readonly Channel<EmailJob> _ch = Channel.CreateBounded<EmailJob>(
        new BoundedChannelOptions(1024) { FullMode = BoundedChannelFullMode.Wait });
    public ValueTask EnqueueAsync(EmailJob j, CancellationToken ct) => _ch.Writer.WriteAsync(j, ct);
    public IAsyncEnumerable<EmailJob> ReadAllAsync(CancellationToken ct) => _ch.Reader.ReadAllAsync(ct);
}

public sealed class EmailWorker(EmailQueue queue, IEmailSender sender) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stop)
    {
        await foreach (var j in queue.ReadAllAsync(stop))
            await sender.SendAsync(j, stop);
    }
}
```

`BoundedChannelOptions.FullMode = Wait` gives you backpressure for free.

### Graceful shutdown

`IHostApplicationLifetime.ApplicationStopping` fires before `StopAsync`;
register cleanup there. Configure `HostOptions.ShutdownTimeout` to a value
larger than your longest in-flight request — default is 30s.

### `ProblemDetails` + `IExceptionHandler`

```csharp
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<DomainExceptionHandler>();
app.UseExceptionHandler();
```

[`IExceptionHandler` (NET 8+)](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling#iexceptionhandler)
is the new chain — register multiple, return `true` to mark handled.

### `WebApplicationFactory` integration tests

Use a `TestAuthHandler` that issues a synthetic claims principal so
`[Authorize]` routes are exercised; replace `IExternalApi` with a
`WireMock.Net` fixture. See
[Integration tests in ASP.NET
Core](https://learn.microsoft.com/aspnet/core/test/integration-tests).

### Testcontainers

[`Testcontainers for .NET`](https://dotnet.testcontainers.org/) — real
Postgres/SQL Server/Redis in Docker per test class via `[CollectionFixture]`.
Faster and more correct than in-memory providers.

### Snapshot testing

```csharp
await Verify(response).ScrubLinesContaining("traceparent");
```

Verify integrates with xUnit/NUnit/MSTest; commit the `*.verified.txt`.

### Architecture tests

```csharp
var result = Types.InAssembly(typeof(OrderHandler).Assembly)
    .That().ResideInNamespace("Contoso.Domain")
    .Should().NotHaveDependencyOn("Microsoft.EntityFrameworkCore")
    .GetResult();
Assert.True(result.IsSuccessful, string.Join(",", result.FailingTypeNames ?? []));
```

### Public-API tracking

Per packable project: `PublicAPI.Shipped.txt` + `PublicAPI.Unshipped.txt`.
Combined with `EnablePackageValidation`, accidental breaks fail the build.

### `IAsyncDisposable` discipline

```csharp
await using var conn = await _factory.CreateAsync(ct);
```

Both `using` and `await using` on the same scope are fine; don't mix synchronous
disposal with async resources or you block on shutdown.

### Telemetry-by-construction

Every cross-boundary type takes `ILogger<T>`, owns an `ActivitySource`, and
emits a `Meter`. Bake it in at construction so you don't retrofit it after
incidents:

```csharp
public sealed class GraphClient(HttpClient http, ILogger<GraphClient> log)
{
    private static readonly ActivitySource Activity = new("Contoso.Graph");
    private static readonly Meter Meter = new("Contoso.Graph");
    private static readonly Counter<long> CallCount = Meter.CreateCounter<long>("graph.calls");
    // ...
}
```

OpenTelemetry [docs for
.NET](https://opentelemetry.io/docs/instrumentation/net/).

---

## GoF + .NET-idiomatic patterns

For each: when to use → 5–15 line C# → "in .NET, prefer X built-in over hand-roll".

### Strategy
Pick an algorithm at runtime. Resolve via DI; don't build your own factory.
```csharp
public interface IHasher { string Hash(string s); }
public sealed class Sha256Hasher : IHasher { /* ... */ }
public sealed class Argon2Hasher : IHasher { /* ... */ }

services.AddKeyedSingleton<IHasher, Sha256Hasher>("sha256");
services.AddKeyedSingleton<IHasher, Argon2Hasher>("argon2");

public sealed class Login([FromKeyedServices("argon2")] IHasher hasher);
```
Prefer `IServiceProvider.GetRequiredKeyedService<T>` over `Func<string, IHasher>` factories.

### Decorator
Wrap an interface to add cross-cutting behaviour.
```csharp
services.AddScoped<IOrderService, OrderService>();
services.Decorate<IOrderService, LoggingOrderDecorator>();   // Scrutor
services.Decorate<IOrderService, CachingOrderDecorator>();
```
Prefer [Scrutor](https://github.com/khellang/Scrutor) `Decorate` over hand-rolled wrappers; for AOT-safe alternatives use a source generator (e.g. [DispatchProxy](https://learn.microsoft.com/dotnet/api/system.reflection.dispatchproxy) is not AOT).

### Adapter
Translate one interface to another at a boundary.
```csharp
public sealed class GraphUserAdapter(GraphClient g) : IUserDirectory
{
    public async Task<User?> FindAsync(string upn, CancellationToken ct)
        => (await g.GetUserAsync(upn, ct))?.ToUser();
}
```
Prefer hand-written adapters over reflection mappers at boundaries — they're the documentation of the seam.

### Factory / Abstract Factory
Defer construction.
```csharp
services.AddScoped<Func<TenantId, ITenantContext>>(sp =>
    id => ActivatorUtilities.CreateInstance<TenantContext>(sp, id));
```
For most cases, `IServiceProvider.GetRequiredKeyedService<T>(key)` replaces the factory entirely.

### Builder
Step-wise construction of complex objects.
```csharp
var req = new HttpRequestMessage(HttpMethod.Post, url);
req.Headers.Authorization = new("Bearer", token);
req.Content = JsonContent.Create(payload, options: AuthJsonContext.Default.Options);
```
The BCL already gives you fluent builders for the common cases (`HttpRequestMessage`, `WebApplicationBuilder`, `HostBuilder`, `BlobClientOptionsBuilder`); only hand-roll a builder for *your* domain-specific construction.

### Singleton
A single shared instance.
```csharp
services.AddSingleton<IClock, SystemClock>();
```
Always via the DI container, never `static readonly Foo Instance = new();` — the latter is untestable and forces lifetime decisions on consumers.

### Observer
Push notifications to subscribers.
```csharp
var ch = Channel.CreateUnbounded<OrderPlaced>();
_ = Task.Run(async () => { await foreach (var e in ch.Reader.ReadAllAsync()) await Handle(e); });
await ch.Writer.WriteAsync(new OrderPlaced(id));
```
For in-proc events use `Channel<T>` (backpressure!) or MediatR notifications. Skip `IObservable<T>` unless you genuinely need Rx operators.

### Mediator
Decouple senders from handlers.
```csharp
public record PlaceOrder(CartId Cart) : IRequest<OrderId>;
public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrder, OrderId> { /* ... */ }
var orderId = await mediator.Send(new PlaceOrder(cart));
```
[MediatR](https://github.com/jbogard/MediatR) for in-proc; [Wolverine](https://wolverinefx.net/) when you also want messaging + outbox in the same primitive. Don't use mediator as a hand-rolled service-locator.

### Command
Encapsulate a request as an object — the C in CQRS.
Same shape as Mediator above; commands are the *write* side, return `OrderId` or `Result<…>`, never query data back.

### Chain of Responsibility
Pass a request through a chain.
```csharp
app.UseExceptionHandler();
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
```
ASP.NET Core middleware *is* CoR. For exceptions specifically, register multiple `IExceptionHandler`s — the framework chains them.

### Template Method
Algorithm skeleton with overridable steps.
```csharp
public abstract class HostedJob : BackgroundService
{
    protected sealed override async Task ExecuteAsync(CancellationToken stop)
    {
        await OnStartAsync(stop);
        while (!stop.IsCancellationRequested) await TickAsync(stop);
    }
    protected abstract Task TickAsync(CancellationToken ct);
    protected virtual Task OnStartAsync(CancellationToken ct) => Task.CompletedTask;
}
```
`sealed override` on the framework method, `abstract` on the customisation point — communicates the contract.

### Visitor
Operate on a closed hierarchy.
```csharp
decimal Total(LineItem item) => item switch
{
    Product p   => p.Price * p.Qty,
    Discount d  => -d.Amount,
    Shipping s  => s.Cost,
    _ => throw new UnreachableException()
};
```
C# pattern matching on sealed hierarchies is the idiomatic Visitor. Mark the base type `abstract` and *all* derivations `sealed`; the compiler then warns on missing arms.

### Specification
Encapsulate a query predicate.
```csharp
public sealed class ActiveOrdersOver(decimal min) : Specification<Order>
{
    public ActiveOrdersOver() { Query.Where(o => o.IsActive && o.Total >= min); }
}
```
Use [Ardalis.Specification](https://github.com/ardalis/Specification); plays nicely with EF Core projections.

### Null Object
Avoid null branches with a do-nothing implementation.
```csharp
public static class NullClock { public static readonly IClock Instance = new Impl(); /* ... */ }
```
The BCL ships [`NullLogger<T>.Instance`](https://learn.microsoft.com/dotnet/api/microsoft.extensions.logging.abstractions.nulllogger-1) as the canonical example.

### Memento
Capture state for undo.
```csharp
var snapshot = doc.SaveSnapshot();
doc.Mutate();
if (cancel) doc.Restore(snapshot);
```
For larger graphs: serialise via STJ source-gen so the snapshot is a `byte[]`.

### Composite
Treat one and many uniformly.
```csharp
public sealed class CompositeHealthCheck(IEnumerable<IHealthCheck> inner) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(...)
        => /* aggregate */;
}
```
Most BCL composites are already provided: `IConfiguration` chains providers, `LoggerProvider` chains writers, `IFileProvider` composes via `CompositeFileProvider`.

### Proxy
Stand-in object controlling access.
For cross-cutting at runtime, [Castle DynamicProxy](https://github.com/castleproject/Core) is fine on JIT but not AOT. AOT-safe alternative: a Roslyn source generator that produces a sealed wrapper class. Decorator via Scrutor covers most "I want logging around this interface" cases without proxies at all.

### Flyweight
Share immutable state to save memory.
```csharp
ReadOnlySpan<byte> data = ...;
var rented = ArrayPool<byte>.Shared.Rent(8192);
try { /* use rented */ } finally { ArrayPool<byte>.Shared.Return(rented); }
```
BCL: `ArrayPool<T>`, `MemoryPool<T>`, `string.Intern`/`StringPool` (CommunityToolkit). Hand-roll only after a profiler shows allocation pressure.

### Bridge
Separate abstraction from implementation so each varies independently.
`ILogger` (abstraction) ↔ `ILoggerProvider` (impl) is the canonical .NET example: Serilog, Console, EventSource all plug in without consumers changing.

### Iterator
Lazy traversal.
```csharp
public async IAsyncEnumerable<Page> PagesAsync([EnumeratorCancellation] CancellationToken ct)
{
    string? next = null;
    do { var p = await Fetch(next, ct); yield return p; next = p.NextLink; } while (next is not null);
}
```
`yield return` for sync, `IAsyncEnumerable<T>` + `[EnumeratorCancellation]` for async. Don't materialise to `List<T>` unless the caller needs random access.

### State
Behaviour varies with state.
For non-trivial workflows use [Stateless](https://github.com/dotnet-state-machine/stateless); for simple toggles, an `enum` + `switch` is enough — don't introduce a state-machine library for two states.

### Interpreter
Evaluate sentences in a language.
You almost never write this by hand in .NET — Roslyn (C#), `System.Linq.Expressions`, EF Core's LINQ provider, and [Sprache](https://github.com/sprache/Sprache) cover the realistic cases. If you find yourself building one, you probably want a config DSL — consider YAML/JSON + a typed binding instead.

---


---

See [`references.md`](./references.md) for the curated source list.
