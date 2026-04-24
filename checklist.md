# .NET 10 Backend PR Checklist

One page. Pull this up during code review. Pairs are ✅ do / ❌ don't. If a rule surprises you, jump to the linked deep doc at the bottom.

## 1. Project & build

- ✅ `<Nullable>enable</Nullable>` and `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` in `Directory.Build.props`.
- ✅ Central Package Management (`Directory.Packages.props`, `ManagePackageVersionsCentrally=true`).
- ✅ `Microsoft.CodeAnalysis.NetAnalyzers` + `EnforceCodeStyleInBuild=true`; `.editorconfig` checked in.
- ✅ `<Deterministic>true</Deterministic>` and `ContinuousIntegrationBuild=true` in CI.
- ✅ `bin/`, `obj/`, `*.user`, `.vs/` in `.gitignore`.
- ❌ Per-project `<PackageReference Version="...">` overrides without a CPM exception comment.
- ❌ `#pragma warning disable` without a justification comment + scope.
- ❌ Committing build artifacts or local `launchSettings.json` secrets.

## 2. C# style

- ✅ File-scoped namespaces (`namespace Foo;`).
- ✅ `record` / `record struct` for DTOs and value-like types.
- ✅ `sealed` by default for internal/implementation classes.
- ✅ `var` only when the RHS makes the type obvious; otherwise spell it out.
- ❌ Primary constructors on non-trivial types where parameters get captured into multiple methods (mutability + capture surprises).
- ❌ Null-forgiving `!` to silence the compiler — fix the nullability instead, or `ArgumentNullException.ThrowIfNull`.
- ❌ Public mutable fields. Public setters on DTOs that should be `init`.
- ❌ `#region` to hide complexity.

## 3. Async

- ✅ `await` everywhere; propagate `CancellationToken` through every async signature.
- ✅ `ConfigureAwait(false)` in **library** code; not required in ASP.NET Core app code.
- ✅ `await using` for `IAsyncDisposable`.
- ✅ `Task.WhenAll` for independent work; respect cancellation across the set.
- ❌ `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` in request paths. Ever.
- ❌ `async void` outside event handlers.
- ❌ `ValueTask` "for perf" without a benchmark — it has sharp edges (single-await, no double-consume).
- ❌ Swallowing `OperationCanceledException` as if it were a failure.

## 4. Dependency injection

- ✅ Lifetime hygiene: scoped consumed only inside a scope; resolve via `IServiceScopeFactory.CreateScope()` from singletons / hosted services.
- ✅ `IHttpClientFactory` (typed clients) for all outbound HTTP.
- ✅ Pick the right options accessor: `IOptions<T>` for singletons, `IOptionsSnapshot<T>` for scoped/per-request, `IOptionsMonitor<T>` for change notifications in singletons.
- ✅ Constructor injection; keyed services where you actually have variants.
- ❌ Captive dependencies (singleton holding a scoped/transient).
- ❌ `new HttpClient()` in product code.
- ❌ Service Locator (`IServiceProvider.GetService<T>()`) sprinkled through business logic.
- ❌ Static mutable state masquerading as DI.

## 5. Configuration & options

- ✅ Bind sections to typed options; validate with `IValidateOptions<T>` and `.ValidateOnStart()`.
- ✅ `[Required]`, `[Range]`, `[Url]` data annotations + `ValidateDataAnnotations()` where they suffice.
- ✅ Secrets from Key Vault / managed identity / env; `dotnet user-secrets` locally.
- ✅ One config tree per ring (dev/test/canary/prod) with explicit env overlays.
- ❌ Secrets, connection strings, or tokens in `appsettings*.json` committed to the repo.
- ❌ Reading `IConfiguration` directly inside business logic.
- ❌ "Optional" config that silently no-ops in prod.

## 6. Logging

- ✅ Source-generated `LoggerMessage` (`[LoggerMessage(...)]` partial methods) on hot paths.
- ✅ Structured args: `_log.OrderRejected(orderId, reason)` — never `$"order {id}"` into the message.
- ✅ Correlation/trace IDs via `Activity.Current` / `W3CTraceContext`.
- ✅ Log levels mean what they say: `Error` = actionable; `Warning` = degraded; `Information` = lifecycle, not chatter.
- ❌ PII (email, name, IP without policy, government IDs) in logs.
- ❌ Access tokens, refresh tokens, client secrets, cookies — even truncated — in logs.
- ❌ `catch { _log.LogError(ex, "boom"); throw; }` — log **or** throw, not both at every layer.

## 7. HTTP (outbound)

- ✅ Typed `HttpClient` registered via `AddHttpClient<TClient>()`.
- ✅ Standard resilience handler (`AddStandardResilienceHandler()`): retry, circuit breaker, timeout per attempt **and** per request.
- ✅ Explicit `Timeout` and `CancellationToken` on every call.
- ✅ Read responses with `EnsureSuccessStatusCode` only when you've considered the body for diagnostics first.
- ❌ `catch (Exception)` around network calls — catch `HttpRequestException`, `TaskCanceledException`, `IOException` deliberately.
- ❌ Retrying non-idempotent verbs without an idempotency key.
- ❌ Disposing the `HttpClient` you got from the factory.

## 8. ASP.NET Core APIs

- ✅ `AddProblemDetails()` + `IExceptionHandler` for uniform error shape (RFC 9457).
- ✅ OpenAPI via `Microsoft.AspNetCore.OpenApi`; contract reviewed in PR.
- ✅ `Asp.Versioning` configured; version in URL or header — pick one and stick to it.
- ✅ `AddRateLimiter` on public/anonymous endpoints; `AddOutputCache` for cacheable GETs.
- ✅ Minimal API groups with shared filters for auth/validation; or controllers with `[ApiController]`.
- ❌ Returning raw exceptions / stack traces to clients.
- ❌ `app.UseDeveloperExceptionPage()` outside Development.
- ❌ Custom error JSON shapes per endpoint.
- ❌ Binding directly to EF entities from the request body.

## 9. AuthN / AuthZ

See [`docs/best-practices.md`](../best-practices.md) and [`docs/validation.md`](../validation.md) for the why; this is the what.

- ✅ `[Authorize]` (or `.RequireAuthorization()`) explicit on every protected endpoint; fallback policy denies by default.
- ✅ Delegated calls: check **`scp`** (space-separated scopes) against required scope.
- ✅ App-only calls: check **`roles`** (app roles) **and** enforce an **`azp`/`appid` allow-list**.
- ✅ Validate `iss`, `aud`, signing key, `exp`, `nbf`; pin tenant where applicable.
- ✅ Use `RequiredScope` / policy-based authorization, not ad-hoc claim sniffing in handlers.
- ❌ Trusting `aud` alone for authorization decisions.
- ❌ Mixing `scp` and `roles` checks ("either is fine") — they mean different callers.
- ❌ Accepting tokens with `ver=1.0` when you expect `2.0` (or vice versa) without explicit handling.
- ❌ Disabling token validation in any environment that touches real data.

## 10. EF Core / data

- ✅ `AsNoTracking()` on read queries; project to DTOs with `Select(...)`.
- ✅ `IDbContextFactory<T>` in background services, gRPC streaming, and parallel work.
- ✅ `EnableRetryOnFailure()` on SQL/Cosmos providers; surface non-retriable errors.
- ✅ Explicit `Include` only when needed; verify with logged SQL — no surprise N+1s.
- ✅ Migrations are idempotent and reviewed; destructive changes gated.
- ❌ Returning `IQueryable<Entity>` from services / across layers.
- ❌ `ToList()` then `.Where(...)` (client-side filtering) on anything non-trivial.
- ❌ Long-lived `DbContext` instances; sharing one across threads.
- ❌ Lazy loading enabled in web request paths.

## 11. Background work

- ✅ `BackgroundService` / `IHostedService`; honor the `stoppingToken` in every loop.
- ✅ Graceful shutdown: drain in-flight work within `HostOptions.ShutdownTimeout`.
- ✅ Idempotency keys on every externally-visible side effect; safe to replay.
- ✅ Bound concurrency with `Channel<T>`, `Parallel.ForEachAsync`, or a semaphore — not unbounded `Task.Run`.
- ❌ `while (true)` without cancellation checks.
- ❌ `Task.Run` fire-and-forget without `await` and without exception handling.
- ❌ Catching `OperationCanceledException` and continuing the loop on shutdown.

## 12. Testing

- ✅ xUnit v3; `Microsoft.Testing.Platform` runner.
- ✅ `WebApplicationFactory<TEntryPoint>` for API integration tests.
- ✅ Testcontainers (Postgres/SQL/Redis/etc.) over EF Core InMemory provider.
- ✅ `TimeProvider` (and `FakeTimeProvider`) for anything time-dependent — no `DateTime.UtcNow` in product code.
- ✅ Deterministic seeds for randomness; no `Thread.Sleep` in tests.
- ❌ EF Core InMemory provider for relational behavior (it lies about transactions, constraints, concurrency).
- ❌ Tests that hit real cloud resources by default.
- ❌ Snapshot tests over volatile fields (timestamps, GUIDs) without redaction.

## 13. Performance

- ✅ Measure first: BenchmarkDotNet or production traces. Claims about allocations need numbers.
- ✅ Pre-size collections (`new List<T>(capacity)`, `StringBuilder(capacity)`) on hot paths.
- ✅ Pooled `HttpClient` (factory), `ArrayPool<T>`, `RecyclableMemoryStream` where it matters.
- ✅ Source-generated `System.Text.Json` (`JsonSerializerContext`) for hot serialization paths.
- ✅ Prefer `Span<T>`/`ReadOnlySpan<T>` for parsing on hot paths — with a benchmark.
- ❌ "Perf" rewrites with no baseline benchmark.
- ❌ `string.Format` / interpolation in tight loops where a writer / `Utf8Formatter` exists.
- ❌ LINQ on hot paths "because it's nicer" without checking allocations.

## 14. Cloud-native

- ✅ OpenTelemetry: traces + metrics + logs exported (OTLP). ASP.NET Core, HttpClient, EF Core instrumentation enabled.
- ✅ Health checks split: `/health/live` (process up) and `/health/ready` (deps reachable); separate ports/tags from app traffic.
- ✅ `terminationGracePeriodSeconds` ≥ `HostOptions.ShutdownTimeout` + drain time; `preStop` hook if your platform needs it.
- ✅ Managed Identity / Workload Identity / Federated Identity Credentials over client secrets.
- ✅ Data Protection keys persisted to a shared store (Blob + Key Vault) for any multi-instance app issuing cookies/tokens.
- ❌ Liveness probes that hit the database.
- ❌ Client secrets in pipelines when FIC works.
- ❌ Default in-memory Data Protection keys behind a load balancer.

## 15. Security

- ✅ Secret scanning + push protection enabled on the repo.
- ✅ HSTS in production; HTTPS redirection on; secure cookies (`Secure`, `HttpOnly`, `SameSite`).
- ✅ CORS: explicit origins, methods, headers — no `AllowAnyOrigin()` with credentials.
- ✅ Antiforgery for cookie-auth browser endpoints; not needed for pure bearer APIs.
- ✅ SBOM generated and dependency audit (`dotnet list package --vulnerable --include-transitive`) gating CI.
- ❌ Secrets in repo, in container images, or in environment variables baked into images.
- ❌ Writing tokens or keys to local disk; use `DataProtection`, Key Vault, or memory only.
- ❌ Disabling certificate validation ("just for staging").

## 16. Dependencies

- ✅ Central Package Management; one version per package across the solution.
- ✅ `dotnet list package --vulnerable --include-transitive` runs in CI and fails on High/Critical.
- ✅ NuGet lock files (`packages.lock.json`) for deployable apps; `RestoreLockedMode=true` in CI.
- ✅ Prefer `Microsoft.Extensions.*` (DI, Logging, Configuration, Resilience, Caching) when at parity with third-party.
- ❌ Floating versions (`*`, `1.*`) in production projects.
- ❌ Pre-release packages in `main` without an explicit owner + removal date.
- ❌ Adding a dependency for a one-liner you can write yourself.

---

If you're not sure why a rule is here, see the linked deep doc in `docs/dotnet/`:
[foundations](./foundations.md) ·
[aspnetcore](./aspnetcore.md) ·
[data](./data.md) ·
[testing](./testing.md) ·
[performance](./performance.md) ·
[cloud-native](./cloud-native.md) ·
[client](./client.md)
