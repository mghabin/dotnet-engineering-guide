# Anti-patterns that create chaos in .NET codebases

The 38 patterns we reject in code review, with the failure mode each one
produces and the right replacement.

> Severity legend: 🔴 hard fail in review, 🟡 needs justification,
> 🟢 recommended.

> Companion pages: [`monorepo.md`](./monorepo.md), [`patterns.md`](./patterns.md),
> shared [`references.md`](./references.md).

---


Each entry: **smell** → **why it bites** → **fix**, with severity.

### 🔴 "God csproj" with 30+ `PackageReference` entries
**Smell.** A single project pulls in MVC, EF Core, gRPC, MassTransit, Polly,
Serilog, AutoMapper, MediatR, FluentValidation, … . **Why.** The project is
doing the job of three: hosting, application, and infrastructure. Build times
explode and the test surface is unclear. **Fix.** Split into `*.Host`,
`*.Application`, `*.Infrastructure` per the [Clean Architecture
template](https://github.com/ardalis/CleanArchitecture); host is the only
project that references infrastructure.

### 🔴 Per-project `<TargetFramework>` drift
**Smell.** Half the solution is `net8.0`, half `net10.0`. **Why.** You lose
language features unevenly, package resolution diverges, and library projects
silently downlevel. **Fix.** Set `TargetFramework` in `Directory.Build.props`;
override only with a code-reviewed `<TargetFrameworks>` for a libraries that
genuinely multi-target. A test exists in `tests/Architecture` that asserts no
csproj overrides `TargetFramework` outside an allow-list.

### 🔴 `Version=` on `PackageReference` while CPM is enabled
**Smell.** `<PackageReference Include="Foo" Version="1.2.3" />` in a CPM repo.
**Why.** NuGet now warns ([NU1010](https://learn.microsoft.com/nuget/reference/errors-and-warnings/nu1010))
or errors; either the override is silently ignored or it diverges from CPM.
**Fix.** Use `VersionOverride` if the override is intentional, otherwise delete
the attribute.

### 🔴 Catching `Exception` and `Console.WriteLine`-ing
**Smell.** `try { … } catch (Exception ex) { Console.WriteLine(ex); }`. **Why.**
Hides every failure mode, breaks observability, and `Console.WriteLine` is not
captured by `ILogger` providers. **Fix.** Catch what you can handle; let the
rest propagate; log via `ILogger.LogError(ex, "context {Order}", id)` so the
exception object is preserved and structured. See [exception
guidelines](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions).

### 🔴 `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` in request paths
**Smell.** Sync-over-async anywhere a request thread can block. **Why.**
[Stephen Cleary, *Don't Block on Async
Code*](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html) —
deadlocks under sync contexts (still a thing in WPF/WinForms), threadpool
starvation under load. **Fix.** `await` all the way; if you really must bridge
sync code, do it at process boundaries (`Main`) and document why.

### 🔴 `async void` outside event handlers
**Smell.** `public async void DoStuff()` on services. **Why.** Exceptions
escape onto the synchronization context and crash the process; callers cannot
`await`. **Fix.** Return `Task`. The only legitimate `async void` is a UI event
handler signature you don't control. Enable [VSTHRD100](https://github.com/microsoft/vs-threading/blob/main/doc/analyzers/VSTHRD100.md).

### 🟡 `Task.Run` over CPU-bound code in ASP.NET Core controllers
**Smell.** `await Task.Run(() => Compute(x))` inside an action method. **Why.**
You're moving work from one threadpool thread to another; net effect is
*negative*. ASP.NET Core has no synchronization context to free up.
[David Fowler, AspNetCoreDiagnosticScenarios](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/main/AsyncGuidance.md#warning-avoid-using-taskrun-for-long-running-work-that-blocks-the-thread).
**Fix.** Either inline (it's already on a worker thread) or move to a
`BackgroundService` + `Channel<T>` if the work is truly long-running.

### 🔴 Static `HttpClient` everywhere without `IHttpClientFactory`
**Smell.** `private static readonly HttpClient _http = new();`. **Why.** DNS
changes are cached forever; no resilience pipeline; no shared handler pooling.
**Fix.** Register a typed client and inject it.
[`IHttpClientFactory` docs](https://learn.microsoft.com/dotnet/core/extensions/httpclient-factory).
```csharp
builder.Services.AddHttpClient<GraphClient>(c => c.BaseAddress = new(url))
    .AddStandardResilienceHandler();
```

### 🔴 Hand-rolled retry/Polly policies where `AddStandardResilienceHandler` exists
**Smell.** Bespoke `Policy.Handle<HttpRequestException>().WaitAndRetryAsync(...)`
in 2025 code. **Why.** The
[`Microsoft.Extensions.Http.Resilience`](https://learn.microsoft.com/dotnet/core/resilience/http-resilience)
package gives you retry + circuit breaker + timeout + hedging with sensible
defaults, OpenTelemetry instrumentation, and configuration binding. **Fix.**
Use `AddStandardResilienceHandler()`; bind `HttpStandardResilienceOptions`
from configuration; only customise the deltas.

### 🔴 DI lifetime mistakes
**Smell.** Singleton holding a `DbContext`; `IServiceProvider.GetService<TScoped>()`
inside a singleton; `IHttpClientFactory` *not* used. **Why.** Captive
dependencies leak DB connections / break tenancy.
[DI guidelines](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines#captive-dependency).
**Fix.** Inject `IServiceScopeFactory`, create a scope per unit of work.
Enable scope validation always (`ValidateScopes=true` in dev *and* prod
hosts).

### 🔴 Service Locator
**Smell.** `serviceProvider.GetService<IFoo>()` sprinkled in domain/application
code. **Why.** Hidden dependencies, untestable, breaks scope analysis. Mark
Seemann has been writing about this for 15+ years; see
[*Service Locator is an anti-pattern*](https://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/).
**Fix.** Constructor inject. The only legitimate `GetRequiredService` calls
are in framework glue (middleware, hosted-service plumbing, factory delegates).

### 🟡 Anaemic domain model + transaction script masquerading as DDD
**Smell.** `OrderEntity` has only properties, all logic is in
`OrderService.PlaceOrder`. **Why.** You've adopted the layering and folder
names of DDD without the modelling — Mark Seemann calls this the
[anaemic data model](https://blog.ploeh.dk/2014/08/11/cqs-versus-server-generated-ids/).
**Fix.** Either commit to behaviour-rich aggregates (Vladimir Khorikov,
[*Domain Model Purity*](https://enterprisecraftsmanship.com/posts/domain-model-purity-completeness/))
*or* be honest and call it a transaction-script architecture; both can ship,
the half-and-half does not.

### 🔴 Repository on top of EF Core
**Smell.** `IGenericRepository<T>` wrapping `DbSet<T>`. **Why.** `DbSet<T>` is
already a repository and `DbContext` is already a unit of work
([Jimmy Bogard, *Favour query objects over
repositories*](https://www.jimmybogard.com/favor-query-objects-over-repositories/)).
You've added an abstraction that hides EF features (Include, AsNoTracking,
projections) and gives you nothing back. **Fix.** Inject `DbContext` directly,
or write specific repositories per aggregate root, or use [Ardalis.Specification](https://github.com/ardalis/Specification)
when query reuse is real.

### 🔴 `IRepository<T>` + Generic Service + AutoMapper everywhere
**Smell.** "The Onion of Grief" — `IRepository<T>`, `IService<T>`,
`AutoMapper` profile per pair, all configured by reflection at startup.
**Why.** Every layer adds zero domain insight; mapping bugs are silent;
startup eats reflection cost. **Fix.** Hand-write the mappings (it's ~5 lines
per type and trivially testable), or use a *source-generated* mapper like
[Mapperly](https://github.com/riok/mapperly) so it's compile-time-checked and
allocation-free.

### 🔴 Static mutable `DateTime.Now`
**Smell.** `if (order.DueDate < DateTime.UtcNow)`. **Why.** Untestable, time
zones leak, DST bugs. **Fix.** Inject [`TimeProvider`](https://learn.microsoft.com/dotnet/standard/datetime/timeprovider-overview)
(in-box since .NET 8) and use `provider.GetUtcNow()`. Test code uses
`FakeTimeProvider` from `Microsoft.Extensions.TimeProvider.Testing`.

### 🟡 `using Newtonsoft.Json` in greenfield .NET 10 code
**Smell.** New code reaching for Json.NET. **Why.** `System.Text.Json` is
faster, AOT-friendly, has a source generator, and is part of the BCL. Json.NET
has features (custom contract resolvers, `JObject` ergonomics) that justify it
in *legacy* paths but not greenfield. **Fix.** Default to `System.Text.Json`
with `[JsonSerializable]` source generation; pull in Json.NET only with a
documented reason.

### 🔴 Null-forgiving `!` to silence warnings
**Smell.** `.FirstOrDefault()!.Name`. **Why.** You suppressed the very
analyser that would have caught the next bug. **Fix.** `.First()` if it must
exist (and let it throw with context), or pattern-match `is { } x` and handle
the null branch. The `!` operator is acceptable in three places only — see
[`../docs/01-foundations.md` §3](../docs/01-foundations.md).

### 🔴 `CancellationToken` ignored
**Smell.** Method takes `CancellationToken ct` and never passes it to the
`HttpClient`/`DbContext`/`Stream`. **Why.** Client disconnects don't free
server work; shutdown hangs. **Fix.** Propagate to every `await`. Add the
[`CA2016`](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/ca2016)
analyser at error severity.

### 🟡 Unbounded `Parallel.ForEachAsync` / `Task.WhenAll` over network calls
**Smell.** `await Task.WhenAll(items.Select(GetAsync))` over 10 000 items.
**Why.** Saturates the threadpool and the downstream service simultaneously.
**Fix.** `Parallel.ForEachAsync(items, new ParallelOptions { MaxDegreeOfParallelism = 16 }, …)`
or a bounded `Channel<T>` with N consumer tasks.

### 🔴 Global mutable state via static fields
**Smell.** `public static List<User> Cache = new();`. **Why.** Concurrency
bugs, untestable, leaks across requests. **Fix.** Singleton service with a
`ConcurrentDictionary` or — better — `IMemoryCache` / `HybridCache`
([.NET 9+ docs](https://learn.microsoft.com/aspnet/core/performance/caching/hybrid)).

### 🟡 Reflection-heavy mapping in hot paths
**Smell.** AutoMapper / hand-rolled `Activator.CreateInstance` in a request
loop. **Why.** Per-call reflection allocates and is slow.
[Steve Gordon's `IHttpClientFactory` perf
posts](https://www.stevejgordon.co.uk/) consistently flag reflection as a top
allocator. **Fix.** Source generators — Mapperly for object-to-object, STJ
source-gen for JSON, `LoggerMessage` for logs.

### 🔴 `IConfiguration["X"]` strewn across the code
**Smell.** `_config["Foo:Bar"]` inside services. **Why.** Stringly-typed,
no validation, no change tracking, no IntelliSense. **Fix.** Bind to typed
options with `services.Configure<FooOptions>(config.GetSection("Foo"))` and
inject `IOptions<FooOptions>` (or `IOptionsSnapshot` / `IOptionsMonitor` per
lifetime).

### 🟡 `logger.LogError(ex.ToString())`
**Smell.** Exception passed as a message string. **Why.** You lose the
structured exception object; sinks (App Insights, Seq, ELK) can't render the
stack as exception data. **Fix.** `logger.LogError(ex, "Order {OrderId} failed", id)`.
Source-generate with `[LoggerMessage]` to remove the boxing of `id`.

### 🟡 Allocation-heavy LINQ in hot paths
**Smell.** `items.Where(...).Select(...).ToList()` inside a per-request
inner loop. **Why.** Allocates an iterator chain + a list per call. Fine on
warm code; not fine on the hottest 1 %. **Fix.** Profile first
([BenchmarkDotNet docs](https://benchmarkdotnet.org/)). When measured,
collapse to a `for` loop over an array/`Span<T>` and pre-size the destination.

### 🟡 Big "Common"/"Utils"/"Helpers" projects
**Smell.** A `Contoso.Common` referenced by *everything*. **Why.** Becomes a
cycle magnet, every change rebuilds the world, hides domain seams. **Fix.**
Push helpers into the project that owns the concept; if it really is shared,
name it for what it is (`Contoso.Time`, `Contoso.Json`) and version it
deliberately.

### 🔴 "Shared" entity assemblies leaked across bounded contexts
**Smell.** `Contoso.Entities` referenced by `Auth` *and* `Billing`. **Why.**
A schema change in one context cascades into the other; you've recreated the
shared-database antipattern in code form. **Fix.** Each bounded context owns
its own entity types; share *messages* (DTOs, integration events), not
entities. See [.NET Microservices
eBook §5](https://learn.microsoft.com/dotnet/architecture/microservices/multi-container-microservice-net-applications/microservice-application-design).

### 🟡 Hand-written DTO ↔ entity mapping with no tests
**Fix.** Either Mapperly with a `[MapperGenerator]`, or hand-write *and* add
a snapshot test (Verify) that fails when a property is added on either side.

### 🔴 Coupling controllers to EF DbContext directly
**Smell.** Controller injects `AppDbContext` and queries inline. **Why.** Mixes
HTTP, application logic, and persistence; integration tests must spin up EF.
**Fix.** Inject an application service / handler; the controller stays thin.
(Acceptable in *deliberately* CRUD-only minimal APIs — call it out.)

### 🔴 `Startup.cs` god-class
**Smell.** 600-line `ConfigureServices`. **Why.** Unreviewable, merge-conflict
magnet, no feature ownership. **Fix.** Feature-folder composition root —
`AddAuth(this IServiceCollection)`, `AddBilling(...)` — each living next to
the feature it composes; `Program.cs` only orchestrates.

### 🟡 Holding files open / DI'd `Stream`s
**Smell.** Singleton holding a `FileStream`. **Why.** Lifetime mismatch,
locks across reloads, no clean disposal. **Fix.** Open per-operation; use
`IFileProvider` for read-only assets; `await using` everywhere.

### 🔴 Magic strings for config/keys
**Smell.** `_config["ConnectionStrings:Default"]`. **Why.** Typo == prod
outage. **Fix.** `nameof`, constants in a `static class Keys`, or
`[StringSyntax("Json")]` / `[StringSyntax("Regex")]` on parameters so the IDE
gives you syntax checking.

### 🔴 Build-time secrets in `appsettings.Development.json` checked in
**Fix.** [`dotnet user-secrets`](https://learn.microsoft.com/aspnet/core/security/app-secrets)
locally; Key Vault + Managed Identity in deployed envs; CI uses
GitHub OIDC → Azure federated credentials, never long-lived secrets.

### 🟡 ASP.NET Core MVC + Minimal APIs mixed without a clear rule
**Smell.** Half the endpoints are controllers, half are `app.MapGet` lambdas.
**Why.** Two binding/validation/error pipelines. **Fix.** Pick one per
*assembly*. Acceptable mix: minimal APIs for `/health`, `/metrics`, internal
diagnostics; controllers for everything user-facing — or vice versa, but
write the rule down.

### 🟡 Misuse of `IAsyncEnumerable<T>`
**Smell.** `await foreach` then `ToListAsync()` immediately. **Why.** You
lost streaming benefit and added overhead. Worse: deferred enumeration that
disposes the `DbContext` before consumption. **Fix.** Stream end-to-end
(`return _db.Orders.AsAsyncEnumerable()` → `await foreach` in the controller),
or materialise and return `IReadOnlyList<T>` — never both.

### 🔴 `record` for domain entities with mutable EF Core navigation properties
**Smell.** `public record Order(Guid Id) { public List<OrderLine> Lines { get; set; } }`.
**Why.** Records have value equality; EF entities have *identity* equality.
Two `Order` instances with the same `Id` but different `Lines` are unequal,
which breaks change tracking and `HashSet` invariants.
[Khalid Abuhakmeh on records vs entities](https://khalidabuhakmeh.com/dotnet-records-and-entity-framework-core).
**Fix.** `class` for entities; `record` for DTOs/events/value objects.

### 🔴 Public `set;` everywhere on EF entities
**Smell.** `public string Status { get; set; }`. **Why.** Anyone can put the
aggregate in an invalid state. **Fix.** `private set` + behaviour methods
(`order.MarkPaid()`); EF reads private setters fine via the
[backing-field convention](https://learn.microsoft.com/ef/core/modeling/backing-field).

### 🟡 Big-bang upgrades across SDK majors
**Smell.** "We'll skip .NET 9 and go straight to .NET 11 next year." **Why.**
You accumulate two majors of breaking changes; debugging which one broke a
dependency is painful. **Fix.** Adopt every LTS at minimum; STS too if you
have a healthy CI pipeline. Use the [.NET upgrade
assistant](https://learn.microsoft.com/dotnet/core/porting/upgrade-assistant-overview)
on a branch, land in pieces.

### 🟡 Disabled analyzers project-by-project
**Smell.** `<NoWarn>CA1822;CA2007;…</NoWarn>` in 30 csproj files. **Why.**
Severity is a property of *what kind of code this is*, not which folder. **Fix.**
Tune analyzers in `.editorconfig` once, with a comment per disable; let the
analyzer tree decide based on `IsTestProject` etc.

---


---

See [`references.md`](./references.md) for the curated source list.
