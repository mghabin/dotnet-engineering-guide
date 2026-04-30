# .NET 10 / C# 13 — Foundations

Opinionated, cross-cutting defaults for a .NET 10 codebase. Everything here is
do-this-by-default; deviations should be justified in code review. ASP.NET Core,
EF Core, testing, perf, cloud-native, and client topics are covered in sibling
documents.

> Conventions: **Do** = required default. **Don't** = reject in review unless a
> written exception exists. **Prefer** = strong default; use the alternative
> only with measurement or a documented constraint.

---

## 1. Solution & projects

**Pin the SDK.** Every repo has a `global.json` at root with an explicit SDK
version and `rollForward: latestFeature`. This stops "works on my machine" from
SDK drift and makes CI reproducible.

```json
{ "sdk": { "version": "10.0.100", "rollForward": "latestFeature", "allowPrerelease": false } }
```

**Use `.slnx`.** The XML solution format (GA in VS 17.14 / .NET SDK 9.0.200+) is
the future of solution files; it is human-readable, diff-friendly, and
officially supported by `dotnet sln`, MSBuild, and Rider/VS. New repos should
not start with `.sln`. Migrate existing ones with `dotnet sln migrate`.

**Centralize MSBuild defaults in `Directory.Build.props`** at repo root:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <!-- LangVersion is intentionally omitted: the TFM picks the matching default
         (C# 13 for net9.0, C# 14 for net10.0). `latest` varies by installed compiler
         and can enable features unsupported by the selected TFM, which breaks reproducibility. -->
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <WarningsNotAsErrors></WarningsNotAsErrors> <!-- add specific IDs only with justification -->
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <AnalysisMode>All</AnalysisMode>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);CS1591</NoWarn> <!-- silence "missing XML doc" project-wide; opt back in per-lib -->
    <Deterministic>true</Deterministic>
    <ContinuousIntegrationBuild Condition="'$(CI)' == 'true'">true</ContinuousIntegrationBuild>
    <PublishRepositoryUrl>true</PublishRepositoryUrl>
    <EmbedUntrackedSources>true</EmbedUntrackedSources>
    <IncludeSymbols>true</IncludeSymbols>
    <SymbolPackageFormat>snupkg</SymbolPackageFormat>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.SourceLink.GitHub" PrivateAssets="All" />
  </ItemGroup>
</Project>
```

**Central Package Management.** Add `Directory.Packages.props` and set
`<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>`. Csproj
files reference packages without versions; the props file is the single source
of truth. Combine with `<CentralPackageTransitivePinningEnabled>true</...>` to
also pin transitives.

**`.editorconfig`** is mandatory and authoritative. Drive both IDE and
`dotnet format` from it. Don't duplicate analyzer severity in `.ruleset` files
— set `dotnet_diagnostic.<ID>.severity` in `.editorconfig`.

**Project layout.** One assembly per bounded responsibility; keep `src/` and
`tests/` siblings. Avoid the `Common`/`Utilities` dumping ground — it becomes a
cycle magnet. Internal types + `InternalsVisibleTo` for test seams; no
`public` for things consumers shouldn't depend on.

**Deterministic & reproducible builds.** `Deterministic=true` + Source Link +
`ContinuousIntegrationBuild=true` on CI gives byte-stable outputs and PDBs that
debuggers can resolve to GitHub source. Do not skip Source Link on internal
libs; you will want it the first time prod stack-traces hit a NuGet'd package.

**Sources:**

- [.NET SDK `global.json`](https://learn.microsoft.com/dotnet/core/tools/global-json) — SDK pinning and `rollForward` semantics.
- [Configure C# language version](https://learn.microsoft.com/dotnet/csharp/language-reference/configure-language-version) — why `latest` is hostile to reproducibility; let the TFM pick the default.
- [Solution file (`.slnx`) format](https://learn.microsoft.com/visualstudio/ide/solutions-files) — XML solution format and `dotnet sln migrate`.
- [`Directory.Build.props` & customizing builds](https://learn.microsoft.com/visualstudio/msbuild/customize-by-directory) — repo-wide MSBuild defaults.
- [Central Package Management](https://learn.microsoft.com/nuget/consume-packages/central-package-management) — `Directory.Packages.props`, transitive pinning.
- [Source Link](https://learn.microsoft.com/dotnet/standard/library-guidance/sourcelink) — debug into NuGet packages from prod stacks.
- [`ContinuousIntegrationBuild`](https://learn.microsoft.com/dotnet/core/project-sdk/msbuild-props#continuousintegrationbuild) — what the CI flag strips from PDBs.

---

## 2. Language (C# 13)

**File-scoped namespaces, always.** One namespace per file, declared with
`namespace Foo.Bar;`. Saves an indent level and is enforced via
`csharp_style_namespace_declarations = file_scoped:error`.

**`global using` directives** belong in a single `GlobalUsings.cs` per project,
not scattered. Don't `global using` your own domain types — only framework
namespaces (`System.Collections.Immutable`, `System.Threading.Channels`).

**Primary constructors — use them sparingly.** They are *not* records.
Captured parameters become hidden mutable instance storage with no `readonly`
guarantee — the parameter is in scope for the whole class body and remains
reassignable, which makes "is this value still what was passed in?" a real
question in code review. They can also shadow base members and complicate
debugging (no field to inspect by name). Use them for small DI-style holders
where every member just forwards a dependency:

```csharp
public sealed class OrderService(IOrderRepository repo, ILogger<OrderService> logger)
{
    public Task<Order?> GetAsync(OrderId id, CancellationToken ct) => repo.GetAsync(id, ct);
}
```

**Don't** mix primary-ctor parameters with additional fields/state when the
parameter is also captured (you end up with two storage models in one type).
Primary constructors are *fine* on `readonly struct` for simple immutable init
where the parameters flow straight into properties — the danger is capturing
mutable state, not the syntax itself. Promote to explicit `private readonly`
fields the moment a class grows non-trivial state, or whenever you need a true
`readonly` guarantee on the captured value.

**`System.Threading.Lock` (C# 13) for new intra-process mutexes.** The new
`Lock` type pairs with `lock (myLock) { ... }` and is allocation-free, with a
disposable `EnterScope()` for `using`-style acquisition. Prefer it in new code
over `lock (new object())`; the only reason to keep an `object` lock is interop
with code that already exposes one or callers that pass `object` through
reflection.

```csharp
private readonly Lock _gate = new();

public void Enqueue(Item item)
{
    lock (_gate) { _buffer.Add(item); }
}
```

**`required` members > constructor explosion.** For DTOs and config types,
prefer `required init` properties over telescoping constructors. Combine with
`SetsRequiredMembersAttribute` only when the type genuinely has a "build it
all in one shot" constructor.

**Records vs classes.**
- `record` (class): immutable data with value equality. DTOs, value objects,
  events, messages.
- `record struct` (readonly): tiny immutable value types (≤16 bytes-ish, no
  heap references) — IDs, coordinates, money.
- `class`: identity, behavior, mutability, or anything held by a DI container.
- `struct`: only with measurement; default to readonly.

Don't use records for entities with identity (EF entities) — equality semantics
will bite you.

**Init-only vs readonly.** `init` for properties on immutable types
(serializable, with-expression friendly). `readonly` for fields inside
mutable classes that must not change after construction.

**Collection expressions** (`[]`, `[..xs, y]`) are the default. They pick the
optimal target type (`List<T>`, `T[]`, `ImmutableArray<T>`, `Span<T>`, etc.) and
beat hand-rolled `new List<T> { ... }` for many cases. Combine with
`params ReadOnlySpan<T>` (C# 13) on hot APIs to avoid array allocation.

**`params` collections (C# 13).** Library APIs taking `params` should use
`params ReadOnlySpan<T>` when the values are consumed eagerly — callers using
collection expressions get zero-alloc dispatch.

**`ref readonly` vs `in`.** `in` is "I won't mutate it, copy is fine if I
need"; `ref readonly` is "I want a non-defensive-copy alias and I promise I
won't mutate." Use `ref readonly` parameters for large readonly structs you
pass through hot loops; `in` is fine for general "no copy please" intent. For
fields of large structs, declare `ref readonly` returns rather than copying.

**Pattern matching style.** Prefer switch expressions over if/else chains for
total mappings; use property and list patterns where they read naturally;
do not chain more than ~5 arms — extract to a method. `is { } x` for
"non-null and capture"; avoid `is not null` *plus* `!` later in the same scope
(the flow analysis already gives you non-null).

**`using` declarations** (no braces) are the default — fewer indents, same
deterministic disposal. Only use the block form when scope must end before the
method does.

**Sources:**

- [What's new in C# 13](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-13) — `params` collections, `ref readonly`, partial properties, new `Lock` type.
- [`System.Threading.Lock` (C# 13)](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-13#new-lock-object) — when the compiler rewrites `lock` to `EnterScope()`.
- [Primary constructors tutorial](https://learn.microsoft.com/dotnet/csharp/whats-new/tutorials/primary-constructors) — capture semantics, when to promote to fields, struct guidance.
- [Collection expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions) — target-typing rules.
- [`params` collections](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/params) — `params ReadOnlySpan<T>` and zero-alloc dispatch.
- [Records](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record) — value equality, `with`, when to choose record vs class.
- [Pattern matching](https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching) — switch expressions, property/list patterns.
- [`ref readonly` parameters](https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/ref#ref-readonly-parameters) — when to use over `in`.

---

## 3. Nullable reference types

NRT is non-negotiable. Every project sets `<Nullable>enable</Nullable>` at
the `Directory.Build.props` level. New code that introduces nullable warnings
fails the build (`TreatWarningsAsErrors=true`).

**The `!` operator is a code smell, not a tool.** It is acceptable in exactly
three places: (a) test code asserting a value the test just produced,
(b) immediately after a framework method whose nullability is wrong (and you
file the bug), (c) inside a generic helper where you've proven non-null but
the analyzer can't see it. Anywhere else, fix the contract.

**Use the flow attributes.** They are the public-API contract:

- `[NotNullWhen(true)] out T? value` — `Try*` patterns.
- `[MaybeNullWhen(false)] out T value` — same, inverted.
- `[NotNull] ref T? value` — "I will assign non-null."
- `[MemberNotNull(nameof(_field))]` on init helpers called from constructors.
- `[MemberNotNullWhen(true, nameof(_x))]` on `IsInitialized`-style flags.
- `[DoesNotReturn]` / `[DoesNotReturnIf(false)]` on guard helpers — flow
  analysis treats subsequent reads as non-null.

**Public API contracts.** Library `public`/`protected` surface must have
deliberate nullability. `string?` parameters mean "null is meaningful";
`string` means "I will throw on null." Throw with
`ArgumentNullException.ThrowIfNull(arg)` — it is annotated and feeds the
analyzer.

**Generics.** `T?` on an unconstrained generic means
`default(T)` — which is `null` for reference types and `0`/`default` for value
types, *not* `Nullable<T>`. If you need `Nullable<T>` semantics, constrain
with `where T : struct` and write `T?` explicitly, or use two overloads.
`where T : notnull` is the right constraint for "no nulls through here".

**Don't suppress at the project level.** `#nullable disable` is for
auto-generated files only. New code does not get a nullable annotation
amnesty.

**Sources:**

- [Nullable reference types](https://learn.microsoft.com/dotnet/csharp/nullable-references) — enabling, suppression rules.
- [Nullable static-analysis attributes](https://learn.microsoft.com/dotnet/csharp/language-reference/attributes/nullable-analysis) — `NotNullWhen`, `MemberNotNull`, `DoesNotReturn`, etc.
- [`ArgumentNullException.ThrowIfNull`](https://learn.microsoft.com/dotnet/api/system.argumentnullexception.throwifnull) — annotated guard; feeds flow analysis.

---

## 4. Async

The canon (Stephen Cleary): **async all the way, no sync-over-async, configure
await in libraries, cancellation everywhere**.

**Never sync-over-async.** `.Result`, `.Wait()`, `GetAwaiter().GetResult()` on
a `Task` you produced are deadlock and threadpool-starvation grenades. The
only legitimate use is at a true sync entry point you don't own (e.g.
`Main` in some hosts — and even then, prefer `await` with `async Main`).

**`ConfigureAwait(false)` in libraries**, never in app/UI/ASP.NET Core
endpoint code. ASP.NET Core has no sync context, so it's a no-op there, but
shared libraries don't know who calls them. Enforce with `CA2007`.

**`ValueTask<T>` only when measured.** Default to `Task<T>`. `ValueTask` exists
to avoid allocations on hot paths that are *usually synchronous* (cache hits,
buffered reads). Misusing it (awaiting twice, blocking on it, storing it) is a
correctness bug. Rule of thumb: if you can't point to a benchmark, use `Task`.

**`IAsyncEnumerable<T>`** for streamed pull-based pipelines (paged APIs, log
tails, EF Core `AsAsyncEnumerable`). Both sides must participate in
cancellation: producers accept `[EnumeratorCancellation] CancellationToken`,
and consumers attach a token via `.WithCancellation(ct)` (the token passed to
`GetAsyncEnumerator` is what `[EnumeratorCancellation]` actually wires up — a
plain `await foreach` does *not* propagate the ambient token):

```csharp
// Producer
public async IAsyncEnumerable<Page> ReadAsync(
    [EnumeratorCancellation] CancellationToken ct = default) { /* illustrative */ }

// Consumer
await foreach (var page in source.ReadAsync().WithCancellation(ct))
{
    // ...
}
```

For push-based fan-in/out, use `System.Threading.Channels` — bounded
`Channel<T>` is the right primitive for producer/consumer. Don't reach for
Rx/`IObservable<T>` unless you actually need temporal operators.

**Cancellation propagation is mandatory.** Every async method that does I/O
takes `CancellationToken ct` as the *last* parameter and forwards it.
Background services use `stoppingToken`; HTTP endpoints use
`HttpContext.RequestAborted` (already wired into minimal API parameter
binding). `Task.Delay` without `ct` is a bug.

**Parallel composition.**
- `Task.WhenAll` — fan-out a known small set of independent awaitables.
  Materialize with `await Task.WhenAll(tasks)`, then read `.Result` on each
  (safe, all completed). Aggregate exceptions: read every task or wrap in
  `try`/`catch` knowing only the first throws.
- `Parallel.ForEachAsync` — bounded async iteration over a collection with
  `MaxDegreeOfParallelism`. This is the right tool for "process N things, no
  more than K at a time, with cancellation."
- Avoid `Task.WhenAll(Enumerable.Select(...))` for unbounded collections —
  you'll DoS your dependencies.

**`await using`** for `IAsyncDisposable` (HTTP responses, DB connections,
streams in .NET). Don't mix `using`/`await using` on the same async-disposable
— the sync `Dispose` may block.

**Don't `async void`.** Except for event handlers. There is no `Task` to
observe, so the method is non-composable (you can't `await` it, you can't
`WhenAll` it, you can't unit-test it cleanly) and any thrown exception
propagates on the captured `SynchronizationContext` — or, with no context, on
the unhandled-exception path that tears down the process. `TaskScheduler.UnobservedTaskException`
is *not* the catch-net here (there is no Task).

**Don't capture `this` accidentally** in long-lived `async` lambdas held by a
singleton — the captured state is the lifetime of the singleton.

**Sources:**

- [Task-based Asynchronous Pattern (TAP)](https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap) — TAP guidance.
- [`ConfigureAwait` FAQ — Stephen Toub](https://devblogs.microsoft.com/dotnet/configureawait-faq/) — definitive treatment.
- [Understanding `ValueTask<T>` — Stephen Toub](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/) — when it pays off; correctness rules.
- [Generate and consume async streams](https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/generate-consume-asynchronous-stream) — `[EnumeratorCancellation]` + `WithCancellation` pairing.
- [`System.Threading.Channels`](https://learn.microsoft.com/dotnet/core/extensions/channels) — producer/consumer primitives.
- [`Parallel.ForEachAsync`](https://learn.microsoft.com/dotnet/api/system.threading.tasks.parallel.foreachasync) — bounded async iteration.
- [Stephen Cleary — *Async/Await Best Practices*](https://learn.microsoft.com/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming) — `async void` exception flow, sync-over-async.

---

## 5. DI & lifetimes

**Three lifetimes, one rule: never inject a shorter-lived service into a
longer-lived one.** That is the *captive dependency* trap. Singletons that
take an `IScoped` capture the first scope's instance for the process lifetime.
Set `ServiceProviderOptions.ValidateScopes = true` and `ValidateOnBuild = true`
in development; the host does this by default in Development environment, but
turn it on explicitly so it can't regress.

**Background work needs `IServiceScopeFactory`.** Inject the factory into your
`BackgroundService`/`IHostedService`, then `await using var scope =
_scopeFactory.CreateAsyncScope();` per unit of work. Resolving a scoped
`DbContext` directly into a hosted service is the canonical bug.

**Keyed services (.NET 8+).** Use them when you have multiple
implementations of the same abstraction selected by a stable key
(`AddKeyedSingleton<IHasher, Sha256Hasher>("sha256")`). Don't simulate this
with factories anymore. Don't use keyed services as a string-typed plugin
registry — that's a configuration smell.

**Options pattern surfaces.**
- `IOptions<T>` — singleton snapshot, value resolved once. Use for
  config that never changes after startup.
- `IOptionsSnapshot<T>` — scoped, re-bound per request. Use in request-scoped
  services that need reload-on-change semantics. **Cannot** be injected into
  singletons.
- `IOptionsMonitor<T>` — singleton, push-notified on change with
  `OnChange`. Use in singletons that must react to config reload.

If you find yourself injecting `IOptions<T>` into a class that already takes
the typed value — just inject `T` (registered with
`services.AddSingleton(sp => sp.GetRequiredService<IOptions<T>>().Value)`).

**`HttpClient`: pick one of two valid defaults — never `new HttpClient()` ad
hoc.** Both options solve socket exhaustion and DNS staleness; pick by whether
you need DI/policies or not.

- **`IHttpClientFactory` (default for app code).** Use when you want DI-resolved
  configuration, named/typed clients, and policy plug-in points
  (`Microsoft.Extensions.Http.Resilience`, delegating handlers, telemetry).
  Two flavors:
  - **Named clients**: `services.AddHttpClient("github", c => ...)`, resolved
    via `IHttpClientFactory.CreateClient("github")`. Good for a small number
    of ad-hoc calls.
  - **Typed clients**: `services.AddHttpClient<IGitHubClient, GitHubClient>(...)`.
    The class takes `HttpClient` in its constructor; lifetime is **transient**,
    but the underlying handler is pooled. Default for new code — testable seam,
    DI-resolved configuration.
- **Long-lived static/singleton `HttpClient` with `SocketsHttpHandler.PooledConnectionLifetime`.**
  Equally supported by Microsoft today. Appropriate for libraries or
  ahead-of-time scenarios where you don't want to take a dependency on
  `IHttpClientFactory`, and for hot paths where you measured the per-request
  factory overhead and care. Set `PooledConnectionLifetime` so connections
  recycle and DNS gets re-resolved:

  ```csharp
  // illustrative
  private static readonly HttpClient Shared = new(new SocketsHttpHandler
  {
      PooledConnectionLifetime = TimeSpan.FromMinutes(2),
  });
  ```

The anti-pattern is *neither* of these — it's `new HttpClient()` per call
(socket exhaustion) or a static `HttpClient` with the default handler (DNS
never re-resolves).

Add resilience via `Microsoft.Extensions.Http.Resilience` (Polly v8 under the
hood): `.AddStandardResilienceHandler()` gets you retry+circuit-breaker+timeout
defaults that are sensible.

**Sources:**

- [Dependency injection in .NET](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection) — lifetimes, validation.
- [DI guidelines / pitfalls](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines) — captive dependency, scope validation, disposal.
- [Keyed services in .NET 8](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#keyed-services) — official intro.
- [Options pattern](https://learn.microsoft.com/dotnet/core/extensions/options) — `IOptions`, `IOptionsSnapshot`, `IOptionsMonitor` semantics.
- [`HttpClient` guidelines for .NET](https://learn.microsoft.com/dotnet/fundamentals/networking/http/httpclient-guidelines) — when `IHttpClientFactory` vs long-lived static `HttpClient` + `PooledConnectionLifetime`.
- [`IHttpClientFactory`](https://learn.microsoft.com/dotnet/core/extensions/httpclient-factory) — named/typed clients, handler pooling.
- [`Microsoft.Extensions.Http.Resilience`](https://learn.microsoft.com/dotnet/core/resilience/http-resilience) — Polly v8 standard handlers.

---

## 6. Configuration & options pattern

**Bind sections to typed records, validate at startup, fail fast.**

```csharp
services.AddOptions<GitHubOptions>()
        .BindConfiguration("GitHub")
        .ValidateDataAnnotations()
        .ValidateOnStart();
```

`ValidateOnStart()` runs validators in the host's `StartAsync` and aborts the
process on failure — far better than discovering the misconfiguration on the
first request that touches it.

**Beyond DataAnnotations**, implement `IValidateOptions<T>` for cross-field
rules, or use the `[OptionsValidator]` source generator (.NET 8+) which
generates a zero-reflection validator at compile time:

```csharp
[OptionsValidator]
public partial class GitHubOptionsValidator : IValidateOptions<GitHubOptions> { }
```

**Layering.** Default order: `appsettings.json` →
`appsettings.{Environment}.json` → user-secrets (Development only) →
environment variables → command-line. Don't fight the precedence; document it.

**Secret hygiene.**
- Never put secrets in `appsettings*.json` (even Development).
- Don't set secrets in shell env vars on dev machines — they leak into every
  process and shell history. Use `dotnet user-secrets` for local dev.
- In every other environment, use Key Vault (or your cloud equivalent) and
  reference via the configuration provider; the provider handles rotation.
- `ILogger` redaction: ensure secrets are never tokens in structured log
  templates; wrap in a custom type that overrides `ToString()`.

**Don't read `IConfiguration` directly** in service code. Bind to options at
composition root; injecting `IConfiguration` into business logic is a code
smell that defeats validation, defeats reload, and defeats testability.

**Sources:**

- [Options pattern](https://learn.microsoft.com/dotnet/core/extensions/options) — semantics of `IOptions`, `IOptionsSnapshot`, `IOptionsMonitor`.
- [Options validation](https://learn.microsoft.com/dotnet/core/extensions/options#options-validation) — `ValidateOnStart`, `IValidateOptions<T>`, `[OptionsValidator]`.
- [Configuration providers & precedence](https://learn.microsoft.com/dotnet/core/extensions/configuration) — layering rules.
- [Safe storage of secrets in development](https://learn.microsoft.com/aspnet/core/security/app-secrets) — `dotnet user-secrets`.

---

## 7. Disposal

**`IDisposable` for unmanaged or scoped resources; `IAsyncDisposable` when
disposal does I/O.** A type that owns a `Stream`, `HttpResponseMessage`, DB
connection, or `CancellationTokenSource` must dispose it.

**`using` declarations** (`using var x = ...;`) are the default — they dispose
at end of scope without an extra indent.

**The full dispose pattern** (`protected virtual Dispose(bool)` + finalizer)
is for types that *directly* hold unmanaged resources. **Don't write
finalizers** otherwise — they make types more expensive (finalization queue),
delay GC, and almost always indicate the wrong design. If your type only owns
managed `IDisposable`s, implement `IDisposable` simply and forward; no
finalizer, no `GC.SuppressFinalize`.

For async-disposable resources, implement both interfaces if the type can be
used in sync contexts; otherwise just `IAsyncDisposable`. Don't call
`Dispose()` from `DisposeAsync()` if it would block.

**`CancellationTokenSource` is `IDisposable`.** Linked CTS instances
(`CreateLinkedTokenSource`) leak callback registrations until disposed —
always `using`.

**Sources:**

- [Implement `IDisposable` / dispose pattern](https://learn.microsoft.com/dotnet/standard/garbage-collection/implementing-dispose) — when finalizers, when not.
- [Implement `IAsyncDisposable`](https://learn.microsoft.com/dotnet/standard/garbage-collection/implementing-disposeasync) — `await using` rules.
- [`CancellationTokenSource`](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtokensource) — disposal of linked sources.

---

## 8. Exceptions

**Exceptions for exceptional.** Validation failures from user input are not
exceptional — return them as values (model-state errors, `ProblemDetails`,
`Result<T>`). Authorization failures, missing-but-expected resources,
optimistic concurrency, downstream timeouts — all *expected* on a healthy
system; design them as flow, not throws.

**Don't catch `Exception`.** Two exceptions: (1) a top-level boundary that
logs and converts to a transport error (ASP.NET exception handler middleware,
hosted service supervisor), (2) an explicit "best effort" with documented
rationale. Catch the narrowest type that captures intent.

**Don't swallow.** `catch { }` is a bug. `catch (Exception ex) { _logger.LogError(ex, "..."); }`
without a re-throw is also a bug unless this *is* the boundary.

**`Result<T>` / `OneOf<T1,T2>` — when warranted.** A `Result` discriminated
union is appropriate when the failure is part of the domain contract and
callers must handle each variant (e.g., `Created | DuplicateEmail | Invalid`).
Don't blanket-replace exceptions with `Result` — you lose stack traces, you
add ceremony to every call site, and you re-invent checked exceptions. Pick
per-API.

**`ExceptionDispatchInfo` for re-throw across async boundaries.** When you
captured an exception (e.g., from a fan-out) and want to rethrow it elsewhere
without losing the original stack:

```csharp
using System.Runtime.ExceptionServices;

ExceptionDispatchInfo.Capture(ex).Throw();
```

A bare `throw ex;` resets the stack — never do that. `throw;` (no operand) in
a catch block preserves it.

**Custom exception types** should be sparse and meaningful. If callers won't
catch it specifically, it shouldn't exist as its own type — a built-in
(`InvalidOperationException`, `ArgumentException`) is fine.

**Sources:**

- [Best practices for exceptions](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions) — design rules.
- [`ExceptionDispatchInfo`](https://learn.microsoft.com/dotnet/api/system.runtime.exceptionservices.exceptiondispatchinfo) — preserving stacks across boundaries.
- [`ProblemDetails` (RFC 7807)](https://datatracker.ietf.org/doc/html/rfc7807) — standard error envelope for HTTP APIs.

---

## 9. Immutability & DTO design

**Default to immutable.** `record` (or `record struct`) with `init`-only
members, then mutate via `with`-expressions. The compiler-generated
equality and `ToString()` are bonuses; the real win is reasoning about
state.

```csharp
public sealed record Order(OrderId Id, CustomerId Customer, ImmutableArray<LineItem> Lines)
{
    public Order AddLine(LineItem line) => this with { Lines = Lines.Add(line) };
}
```

**Use `ImmutableArray<T>`/`ImmutableList<T>`** in DTO surfaces; don't expose
`List<T>` you didn't mean to allow callers to mutate. `IReadOnlyList<T>` is a
view, not a guarantee — a hostile caller can downcast.

**When mutability is right.** Long-lived stateful services (caches, queues,
aggregates inside an aggregate root), perf-critical builders, anything where
the cost of allocation per change dominates. Be explicit: name the type
`MutableX` or document why.

**Equality.** Records get value equality by default. For value objects (money,
ids) that's exactly right. For entities with identity, override or write a
class — never let two `Customer` records with the same fields be `==`.

**Sources:**

- [Records](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record) — value equality, `with` expressions.
- [`System.Collections.Immutable`](https://learn.microsoft.com/dotnet/api/system.collections.immutable) — `ImmutableArray<T>`, `ImmutableList<T>`.
- [Choosing between class and struct](https://learn.microsoft.com/dotnet/standard/design-guidelines/choosing-between-class-and-struct) — when value semantics fit.

---

## 10. Source generators

Source generators have replaced reflection for the cases below. Use them by
default; fall back to reflection only when shape isn't statically knowable.

- **`LoggerMessage` source generator** (`[LoggerMessage(...)]`) for all hot-path
  logging. It eliminates boxing, string formatting, and the enabled-check
  ceremony for the *log call itself*. Unstructured `_logger.LogInformation($"...")`
  is a bug — it defeats structured logging and allocates even when disabled.
  When the *arguments* are expensive to compute (serialization, LINQ, allocations
  for diagnostic snapshots), still guard the call site manually with
  `if (logger.IsEnabled(LogLevel.Debug))` — the generator can't elide your
  argument evaluation.
- **`System.Text.Json` source generator** (`[JsonSerializable(typeof(T))]` on a
  partial `JsonSerializerContext`). Required for AOT/trimming, faster than
  reflection-based serialization, and removes a runtime startup cost. Pass
  the context to `JsonSerializer.Serialize(value, MyContext.Default.MyType)`.
- **`GeneratedRegex`** (`[GeneratedRegex(@"...")]` on a `partial` method).
  Compiles at build time, no `RegexOptions.Compiled` warmup, AOT-safe.
- **`OptionsValidator`** (`[OptionsValidator]`, see §6). Replaces
  reflection-based validation.
- **Minimal API `RequestDelegate` generator** (on by default in ASP.NET Core
  7+). Removes per-request reflection; you get it for free as long as your
  endpoint signatures are static. Don't dynamically build endpoints if you
  can express them statically.
- **`StringSyntax`** (`[StringSyntax(StringSyntaxAttribute.Json)]`,
  `Regex`, etc.) gives IDEs and analyzers the hint they need to colorize and
  validate string literal payloads. *Interceptors are an experimental
  preview-flagged language feature, not a current C# 13 default — treat them
  as ecosystem-specific and don't ship them in shared libraries.*

If you write your own generator, ship it as a separate analyzer project with
`<IsRoslynComponent>true</IsRoslynComponent>`, target
`netstandard2.0`, and **never** take third-party dependencies (the analyzer
runs in the IDE — every dep becomes a load-order hazard).

**Sources:**

- [`LoggerMessage` source generator](https://learn.microsoft.com/dotnet/core/extensions/logger-message-generator) — high-perf logging.
- [Logging guidance for library authors](https://learn.microsoft.com/dotnet/core/extensions/logging-library-authors) — when to call `IsEnabled` to elide argument evaluation.
- [`System.Text.Json` source generation](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation) — AOT, perf, startup.
- [`GeneratedRegex`](https://learn.microsoft.com/dotnet/standard/base-types/regular-expression-source-generators) — compile-time regex.
- [`StringSyntaxAttribute`](https://learn.microsoft.com/dotnet/api/system.diagnostics.codeanalysis.stringsyntaxattribute) — string-literal hints for analyzers/IDEs.
- [What's new in C# 13](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-13) — confirms interceptors are not a shipped C# 13 feature.
- [Source generators overview](https://learn.microsoft.com/dotnet/csharp/roslyn-sdk/source-generators-overview) — authoring guidance.

---

## 11. Build hygiene

**`dotnet format`** runs in CI on every PR (`--verify-no-changes`) and as a
pre-commit hook. Fights happen once, in `.editorconfig`, never in review.

**Analyzers — the baseline set:**
- **`Microsoft.CodeAnalysis.NetAnalyzers`** (built into the SDK) at
  `AnalysisMode=All`, `AnalysisLevel=latest-recommended`. This is the floor.
- **`Roslynator.Analyzers`** for code-style and refactoring suggestions
  beyond what the SDK ships.
- **`SonarAnalyzer.CSharp`** for bug-pattern and cognitive-complexity rules
  (the free Roslyn package, no SonarQube server needed).
- **`Meziantou.Analyzer`** — opinionated rules that catch real
  bugs (sync-over-async, `DateTime.Now` misuse, culture-sensitive string ops,
  task-related foot-guns). Treat as required.
- **`AsyncFixer`** if your team still ships sync-over-async by accident.

Wire them via `Directory.Packages.props` so versions are unified, and via
`Directory.Build.props` so every project gets them automatically.

**Pre-commit with Husky.NET.** `dotnet husky install` then a task runner
(`task-runner.json`) that runs `dotnet format`, `dotnet build --no-restore
-warnaserror`, and any project-specific lints. Keep it under ~10s — a slow
hook gets bypassed.

**NuGet hygiene.**
- `dotnet restore --locked-mode` in CI; commit `packages.lock.json` with
  `<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>`. Locked
  restore is your supply-chain trip-wire.
- `dotnet list package --vulnerable --include-transitive` in CI. Fail the
  build on a known CVE.
- `<NuGetAudit>true</NuGetAudit>` (default in modern SDKs) and
  `<NuGetAuditMode>all</NuGetAuditMode>` to include transitives.
  `NuGetAuditLevel=moderate` is a reasonable bar.
- Single feed in `nuget.config` with `<clear />` first, then your sources.
  Never let a second feed sneak a package in.

**Deterministic + reproducible builds.** Beyond `Deterministic=true`,
`ContinuousIntegrationBuild=true` strips local paths from PDBs so two
machines produce byte-identical artifacts. Combined with Source Link, you can
debug a NuGet'd library straight to the commit it was built from.

**Don't ship `Debug` binaries.** CI publishes `Release` with
`-p:DebugType=portable` and pushes `.snupkg` symbols separately to your symbol
server (or NuGet.org's).

**Sources:**

- [Code analysis in .NET](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/overview) — `AnalysisMode`, `AnalysisLevel`, `EnforceCodeStyleInBuild`.
- [`dotnet format`](https://learn.microsoft.com/dotnet/core/tools/dotnet-format) — formatting + style verification.
- [NuGet package lock files](https://learn.microsoft.com/nuget/consume-packages/package-references-in-project-files#locking-dependencies) — `--locked-mode`, `packages.lock.json`.
- [NuGet audit (`<NuGetAudit>`)](https://learn.microsoft.com/nuget/concepts/auditing-packages) — vulnerability scanning at restore.
- [Reproducible builds (.NET)](https://github.com/dotnet/reproducible-builds) — repo and guidance.

---

## 12. Runtime configuration

The defaults are good. Change them only with measurement, and pin the
overrides in source so they ride with the binary instead of living in a
deployment script.

**Server GC for server workloads.** ASP.NET Core and most service hosts
already opt into Server GC; verify on your project (it is the default for the
ASP.NET Core SDK and for `dotnet new web`/`webapi` templates). For console
apps and worker services, opt in explicitly:

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

Server GC uses one heap and one GC thread per logical core, which is the right
default for throughput-bound services. Workstation GC is the default for
console apps and is appropriate for short-lived CLIs and desktop processes.

**Tiered compilation and Dynamic PGO are on by default — leave them on.**
Tiered compilation lets methods start at quick-JIT and re-JIT to optimized
code based on observed call counts; Dynamic PGO (default since .NET 8)
collects type and branch profiles in the tier-0 code and feeds them to the
tier-1 JIT. Disabling either is a measurable regression on virtually every
real workload — do not disable as a "just in case" tweak. If you suspect a
specific issue, isolate it via `DOTNET_TieredCompilation=0` /
`DOTNET_TieredPGO=0` *in a benchmark*, never in production by default.

**Thread-pool tuning needs a benchmark.** `ThreadPool.SetMinThreads` and the
`System.Threading.ThreadPool.MinThreads` runtime config exist for the
specific case where bursts of synchronous-blocking work starve the pool faster
than the hill-climbing heuristic recovers. Do not raise minimums "for safety"
— you mask design bugs (sync-over-async, blocking I/O) that will surface
elsewhere. Fix the blocking call first.

**Pin runtime knobs via `RuntimeHostConfigurationOption`, not env vars.**
For app-scoped behavior (HTTP/2, socket handler defaults, JSON reflection,
etc.), bake the switch into the project so every deployment carries the same
behavior:

```xml
<ItemGroup>
  <RuntimeHostConfigurationOption
      Include="System.Net.SocketsHttpHandler.Http2UnencryptedSupport"
      Value="true" />
</ItemGroup>
```

This emits the switch into `runtimeconfig.json`, which the runtime reads at
startup and which `dotnet publish` carries into containers untouched.
Environment variables are fine for one-off diagnostics but should never be
how a release knows to behave correctly.

**`runtimeconfig.template.json`** is the override hatch for properties without
a first-class MSBuild surface (e.g. `System.GC.HeapHardLimit`). Commit it
next to the csproj; do not let ops set GC policy in the deploy pipeline.

**Sources:**

- [Runtime configuration options](https://learn.microsoft.com/dotnet/core/runtime-config/) — index of every supported knob.
- [GC configuration options](https://learn.microsoft.com/dotnet/core/runtime-config/garbage-collector) — Server vs Workstation, concurrent, heap limits.
- [Tiered compilation](https://learn.microsoft.com/dotnet/core/runtime-config/compilation) — how tiers and quick-JIT interact.
- [Dynamic PGO (.NET 8 blog)](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/#dynamic-pgo) — what it measures and why it's on by default.
- [Thread-pool runtime config](https://learn.microsoft.com/dotnet/core/runtime-config/threading) — `MinThreads`, hill-climbing semantics.
- [`RuntimeHostConfigurationOption` MSBuild item](https://learn.microsoft.com/dotnet/core/runtime-config/#runtimeconfigjson) — how project switches reach `runtimeconfig.json`.

---

## Further reading

Community sources: secondary to the per-section Microsoft Learn links above,
useful for depth, war stories, and benchmarks.

- [Stephen Cleary's blog](https://blog.stephencleary.com/) — deep dives on cancellation, sync contexts, `ValueTask`.
- [Stephen Cleary — *There Is No Thread*](https://blog.stephencleary.com/2013/11/there-is-no-thread.html) — mental model for async I/O.
- [David Fowler — *AspNetCoreDiagnosticScenarios*](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/main/AsyncGuidance.md) — async + DI gotchas, the de-facto field guide.
- [Andrew Lock — *.NET Escapades*](https://andrewlock.net/) — series on options pattern, configuration, hosting, source generators.
- [Andrew Lock — *Adding validation to strongly-typed configuration objects*](https://andrewlock.net/adding-validation-to-strongly-typed-configuration-objects-in-asp-net-core/) — validation patterns.
- [Steve Gordon's blog](https://www.stevejgordon.co.uk/) — `IHttpClientFactory` internals, channels, performance.
- [Meziantou's blog](https://www.meziantou.net/) — analyzer rationale and a stream of "this is a bug" patterns; companion to `Meziantou.Analyzer`.
- [Roslynator](https://github.com/dotnet/roslynator) — analyzer/refactoring catalog.
- [Husky.NET](https://github.com/alirezanet/Husky.Net) — pre-commit hooks for .NET repos.
