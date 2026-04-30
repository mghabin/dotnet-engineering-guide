# .NET 10 Backend PR Checklist

One page. Pull this up during code review. Each section is a **Do** / **Don't** pair; if a rule surprises you, jump to the linked owner chapter section.

**Baseline:** .NET 10 / C# 14 / .NET Aspire — **Aspire 9.x** for `net8.0`/`net9.0` services, **Aspire 13** for `net10.0` services ([learn.microsoft.com/dotnet/aspire/whats-new/dotnet-aspire-9](https://learn.microsoft.com/dotnet/aspire/whats-new/dotnet-aspire-9), [aspire.dev](https://aspire.dev)). When in doubt where to start, open [`docs/decision-trees.md`](./docs/decision-trees.md) first, then [`SCOPE.md`](./SCOPE.md) and [`coverage-map.md`](./coverage-map.md).

## 1. Project & build

**Do:**

- `<Nullable>enable</Nullable>` and `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` in `Directory.Build.props`.
- Central Package Management (`Directory.Packages.props`, `ManagePackageVersionsCentrally=true`) ([learn.microsoft.com/nuget/consume-packages/central-package-management](https://learn.microsoft.com/nuget/consume-packages/central-package-management)).
- `Microsoft.CodeAnalysis.NetAnalyzers` + `EnforceCodeStyleInBuild=true`; `.editorconfig` checked in.
- `<Deterministic>true</Deterministic>` and `ContinuousIntegrationBuild=true` in CI.
- `bin/`, `obj/`, `*.user`, `.vs/` in `.gitignore`.

**Don't:**

- Per-project `<PackageReference Version="...">` overrides without a CPM exception comment.
- `#pragma warning disable` without a justification comment + scope.
- Commit build artifacts or local `launchSettings.json` secrets.

See: [`docs/01-foundations.md`](./docs/01-foundations.md), [`patterns/monorepo.md`](./patterns/monorepo.md).

## 2. C# style

**Do:**

- File-scoped namespaces (`namespace Foo;`) and `record` / `record struct` for DTOs and value-like types ([learn.microsoft.com/dotnet/csharp/whats-new/csharp-14](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-14)).
- `sealed` by default for internal/implementation classes.
- `var` only when the RHS makes the type obvious; otherwise spell it out.

**Don't:**

- Primary constructors on non-trivial types where parameters get captured into multiple methods (mutability + capture surprises).
- Null-forgiving `!` to silence the compiler — fix the nullability instead, or `ArgumentNullException.ThrowIfNull`.
- Public mutable fields. Public setters on DTOs that should be `init`.
- `#region` to hide complexity.

See: [`docs/01-foundations.md`](./docs/01-foundations.md).

## 3. Async

**Do:**

- `await` everywhere; propagate `CancellationToken` through every async signature ([learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/consuming-the-task-based-asynchronous-pattern](https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/consuming-the-task-based-asynchronous-pattern)).
- `ConfigureAwait(false)` in **library** code; not required in ASP.NET Core app code ([devblogs.microsoft.com/dotnet/configureawait-faq](https://devblogs.microsoft.com/dotnet/configureawait-faq)).
- `await using` for `IAsyncDisposable`.
- `Task.WhenAll` for independent work; respect cancellation across the set.

**Don't:**

- `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` in request paths. Ever.
- `async void` outside event handlers.
- `ValueTask` "for perf" without a benchmark — it has sharp edges (single-await, no double-consume) ([learn.microsoft.com/dotnet/api/system.threading.tasks.valuetask-1](https://learn.microsoft.com/dotnet/api/system.threading.tasks.valuetask-1)).
- Swallow `OperationCanceledException` as if it were a failure.

See: [`docs/01-foundations.md`](./docs/01-foundations.md).

## 4. Dependency injection

**Do:**

- Lifetime hygiene: scoped consumed only inside a scope; resolve via `IServiceScopeFactory.CreateScope()` from singletons / hosted services ([learn.microsoft.com/dotnet/core/extensions/dependency-injection](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)).
- `IHttpClientFactory` (typed clients) for all outbound HTTP ([learn.microsoft.com/dotnet/core/extensions/httpclient-factory](https://learn.microsoft.com/dotnet/core/extensions/httpclient-factory)).
- Pick the right options accessor: `IOptions<T>` for singletons, `IOptionsSnapshot<T>` for scoped/per-request, `IOptionsMonitor<T>` for change notifications in singletons ([learn.microsoft.com/dotnet/core/extensions/options](https://learn.microsoft.com/dotnet/core/extensions/options)).
- Constructor injection; keyed services where you actually have variants.

**Don't:**

- Captive dependencies (singleton holding a scoped/transient).
- `new HttpClient()` in product code.
- Service Locator (`IServiceProvider.GetService<T>()`) sprinkled through business logic.
- Static mutable state masquerading as DI.

See: [`docs/01-foundations.md`](./docs/01-foundations.md), decision tree [11 — IHttpClientFactory](./docs/decision-trees.md#11-http-client-ihttpclientfactory-vs-long-lived-httpclient).

## 5. Configuration & options

**Do:**

- Bind sections to typed options; validate with `IValidateOptions<T>` and `.ValidateOnStart()` ([learn.microsoft.com/dotnet/core/extensions/options-validation](https://learn.microsoft.com/dotnet/core/extensions/options-validation)).
- `[Required]`, `[Range]`, `[Url]` data annotations + `ValidateDataAnnotations()` where they suffice.
- Secrets from Key Vault / Workload Identity / env; `dotnet user-secrets` locally ([learn.microsoft.com/aspnet/core/security/app-secrets](https://learn.microsoft.com/aspnet/core/security/app-secrets)).
- One config tree per ring (dev/test/canary/prod) with explicit env overlays.

**Don't:**

- Secrets, connection strings, or tokens in `appsettings*.json` committed to the repo.
- Read `IConfiguration` directly inside business logic.
- "Optional" config that silently no-ops in prod.

See: [`docs/01-foundations.md`](./docs/01-foundations.md), [`docs/06-cloud-native.md#4-configuration--configmap--csi-key-vault-not-appsettingsproductionjson`](./docs/06-cloud-native.md#4-configuration--configmap--csi-key-vault-not-appsettingsproductionjson).

## 6. Logging

**Do:**

- Source-generated `LoggerMessage` (`[LoggerMessage(...)]` partial methods) on hot paths ([learn.microsoft.com/dotnet/core/extensions/logger-message-generator](https://learn.microsoft.com/dotnet/core/extensions/logger-message-generator)).
- Structured args: `_log.OrderRejected(orderId, reason)` — never `$"order {id}"` into the message.
- Correlation/trace IDs via `Activity.Current` and W3C Trace Context ([w3.org/TR/trace-context](https://www.w3.org/TR/trace-context/)).
- Log levels mean what they say: `Error` = actionable; `Warning` = degraded; `Information` = lifecycle, not chatter ([learn.microsoft.com/dotnet/core/extensions/logging#log-level](https://learn.microsoft.com/dotnet/core/extensions/logging#log-level)).

**Don't:**

- PII (email, name, IP without policy, government IDs) in logs.
- Access tokens, refresh tokens, client secrets, cookies — even truncated — in logs.
- `catch { _log.LogError(ex, "boom"); throw; }` — log **or** throw, not both at every layer.

See: [`docs/01-foundations.md`](./docs/01-foundations.md) (primitives) and [`docs/06-cloud-native.md#5-observability--opentelemetry-one-sdk-three-signals`](./docs/06-cloud-native.md#5-observability--opentelemetry-one-sdk-three-signals) (OTLP exporter wiring).

## 7. HTTP (outbound) & resilience

**Do:**

- Typed `HttpClient` registered via `AddHttpClient<TClient>()` and apply `AddStandardResilienceHandler()` once via `services.ConfigureHttpClientDefaults(...)` ([learn.microsoft.com/dotnet/core/resilience/http-resilience](https://learn.microsoft.com/dotnet/core/resilience/http-resilience)).
- Call **`o.Retry.DisableForUnsafeHttpMethods()`** unless every mutating endpoint you call accepts an `Idempotency-Key` header and dedups server-side — the standard handler retries **all** HTTP methods by default, including `POST`/`PATCH`/`PUT`/`DELETE`.
- Set a **total request timeout** *and* a per-attempt timeout — without the total, retries stack unboundedly.
- Explicit `CancellationToken` on every call; read responses with `EnsureSuccessStatusCode` only after considering the body for diagnostics.
- To customize, call `RemoveAllResilienceHandlers()` first, then add a single configured pipeline — never stack handlers.

**Don't:**

- Assume the standard handler skips `POST`/`PATCH` — it does not. The previous "just don't retry non-idempotent verbs" rule was wrong about the default.
- Stack retries across layers (client + gateway + mesh) — pick one layer.
- `catch (Exception)` around network calls — catch `HttpRequestException`, `TaskCanceledException`, `IOException` deliberately.
- Dispose the `HttpClient` you got from the factory.

See: [`docs/02-aspnetcore.md#7-resilience`](./docs/02-aspnetcore.md#7-resilience) (single owner; ch06 §6 defers here), decision tree [15 — resilience](./docs/decision-trees.md#15-resilience-standard-handler-vs-custom-polly-vs-hedging).

## 8. ASP.NET Core APIs

**Do:**

- `AddProblemDetails()` + `IExceptionHandler` for uniform error shape ([RFC 9457](https://datatracker.ietf.org/doc/html/rfc9457)).
- OpenAPI via `Microsoft.AspNetCore.OpenApi`; contract reviewed in PR ([learn.microsoft.com/aspnet/core/fundamentals/openapi/aspnetcore-openapi](https://learn.microsoft.com/aspnet/core/fundamentals/openapi/aspnetcore-openapi)).
- `Asp.Versioning` configured; version in URL or header — pick one and stick to it.
- `AddRateLimiter` on public/anonymous endpoints, partitioned on authenticated identity (`oid`/`sub`), not IP alone ([learn.microsoft.com/aspnet/core/performance/rate-limit](https://learn.microsoft.com/aspnet/core/performance/rate-limit)).
- For cacheable GETs use `AddOutputCache` — see §10 below for the OutputCache rules and link to the owner.
- Minimal API groups with shared filters for auth/validation; or controllers with `[ApiController]`.

**Don't:**

- Return raw exceptions / stack traces to clients.
- `app.UseDeveloperExceptionPage()` outside Development.
- Custom error JSON shapes per endpoint.
- Bind directly to EF entities from the request body.

See: [`docs/02-aspnetcore.md`](./docs/02-aspnetcore.md) §§3, 4, 5, 8.

## 9. AuthN / AuthZ

**Do:**

- `[Authorize]` (or `.RequireAuthorization()`) explicit on every protected endpoint; `FallbackPolicy` denies by default ([learn.microsoft.com/aspnet/core/security/authorization/introduction](https://learn.microsoft.com/aspnet/core/security/authorization/introduction)).
- Keep `MapInboundClaims = false` so `scp` / `roles` / `azp` stay verbatim, and validate `iss`, `aud`, signing key, `exp`, `nbf`; pin tenant where applicable ([learn.microsoft.com/entra/identity-platform/access-token-claims-reference](https://learn.microsoft.com/entra/identity-platform/access-token-claims-reference), [RFC 9068](https://datatracker.ietf.org/doc/html/rfc9068)).
- **Two separate named policies** per capability: a delegated policy that requires the `scp` scope (and rejects tokens carrying `roles` without `scp`); an app-only policy that requires `roles` **and** an `azp`/`appid` allow-list **and** the absence of `scp`.
- For endpoints that legitimately accept both flows, list both policies on the endpoint (`RequireAuthorization("XDelegated", "XApp")`) so each policy still enforces its own invariants.
- Use `RequiredScope` / policy-based authorization, not ad-hoc claim sniffing in handlers.

**Don't:**

- Write a single OR-claims assertion (`HasScope("X") || HasAppRole("Y")`). It lets app tokens reach user-only endpoints (no `azp` allow-list, no `scp`) and lets user tokens satisfy app-only endpoints — both are real privilege-escalation bugs.
- Trust `aud` alone for authorization decisions.
- Accept tokens with `ver=1.0` when you expect `2.0` (or vice versa) without explicit handling.
- Disable `ValidateIssuer`, `ValidateAudience`, or `ValidateLifetime`. Ever.

See: [`docs/02-aspnetcore.md#10-authnauthz`](./docs/02-aspnetcore.md#10-authnauthz) (owner) and decision tree [12 — auth policy shape](./docs/decision-trees.md#12-auth-policy-shape-delegated-scp-vs-app-only-roles--azp). See also (non-normative, runnable sample): [`mghabin/entra-auth-patterns-dotnet`](https://github.com/mghabin/entra-auth-patterns-dotnet).

## 10. Caching

**Do:**

- `AddOutputCache` for cacheable HTTP responses; vary by auth-relevant dimensions (tenant, role) and use **tags** for fan-out invalidation ([learn.microsoft.com/aspnet/core/performance/caching/output](https://learn.microsoft.com/aspnet/core/performance/caching/output)).
- Multi-instance OutputCache: back it with a distributed `IOutputCacheStore` (e.g. `Aspire.StackExchange.Redis.OutputCaching`) — adding `Microsoft.Extensions.Caching.StackExchangeRedis` alone changes nothing about output caching.
- App-data caching: prefer `HybridCache` (L1 in-proc + L2 distributed + built-in stampede protection); fall back to `IDistributedCache` only for plain K/V ([learn.microsoft.com/aspnet/core/performance/caching/hybrid](https://learn.microsoft.com/aspnet/core/performance/caching/hybrid)).
- Cache DTOs / primitives only; invalidate explicitly at `ExecuteUpdate` / `ExecuteDelete` / raw SQL call sites.

**Don't:**

- Cache personalized responses under a shared key — audit `VaryByValue`.
- Put EF-tracked entities in cache (identity-map bombs).
- Reach for a third-party EF L2 cache interceptor — there is no official EF L2 cache; use cache-aside on the query you actually want to cache.

See: [`docs/02-aspnetcore.md#9-output-caching`](./docs/02-aspnetcore.md#9-output-caching) (OutputCache owner) and [`docs/03-data.md#14-caching`](./docs/03-data.md#14-caching) (HybridCache / `IDistributedCache` owner). Decision tree [13 — caching](./docs/decision-trees.md#13-caching-outputcache-vs-hybridcache-vs-idistributedcache).

## 11. EF Core / data

**Do:**

- `AsNoTracking()` on read queries; project to DTOs with `Select(...)` ([learn.microsoft.com/ef/core/querying/tracking](https://learn.microsoft.com/ef/core/querying/tracking)).
- `IDbContextFactory<T>` in background services, gRPC streaming, and parallel work ([learn.microsoft.com/ef/core/dbcontext-configuration](https://learn.microsoft.com/ef/core/dbcontext-configuration)).
- `EnableRetryOnFailure()` on SQL/Cosmos providers; surface non-retriable errors.
- Explicit `Include` only when needed; verify with logged SQL — no surprise N+1s.
- Migrations are idempotent and reviewed; destructive changes go through expand → migrate → contract.
- Cross-system writes ("save row + emit event") go through the **transactional outbox** — a relay publishes after `SaveChangesAsync` commits. Never `SaveChangesAsync()` then `bus.SendAsync()`.

**Don't:**

- Return `IQueryable<Entity>` from services / across layers.
- `ToList()` then `.Where(...)` (client-side filtering) on anything non-trivial.
- Long-lived `DbContext` instances; share one across threads.
- Lazy loading enabled in web request paths.
- Distributed (two-phase / MSDTC) transactions across DB + bus / DB + HTTP / two databases — there is no DTC on Linux .NET, and most managed services don't enlist.

See: [`docs/03-data.md#6-transactions--unit-of-work`](./docs/03-data.md#6-transactions--unit-of-work) (outbox is the single owner here), and decision tree [17 — Cosmos partition key](./docs/decision-trees.md#17-cosmos-db-partition-key) for the modelling rule.

## 12. Background work

**Do:**

- `BackgroundService` / `IHostedService`; honor the `stoppingToken` in every loop ([learn.microsoft.com/dotnet/core/extensions/workers](https://learn.microsoft.com/dotnet/core/extensions/workers)).
- Graceful shutdown: drain in-flight work within `HostOptions.ShutdownTimeout`; the cluster-side drain contract is owned by [`docs/06-cloud-native.md#11-graceful-shutdown--drain-dont-drop`](./docs/06-cloud-native.md#11-graceful-shutdown--drain-dont-drop).
- Idempotency keys on every externally-visible side effect; safe to replay (same `(key, tenant)` UNIQUE store the outbox/inbox uses — see §11).
- Bound concurrency with `Channel<T>`, `Parallel.ForEachAsync`, or a semaphore — not unbounded `Task.Run`.

**Don't:**

- `while (true)` without cancellation checks.
- `Task.Run` fire-and-forget without `await` and without exception handling.
- Catch `OperationCanceledException` and continue the loop on shutdown.

See: [`docs/02-aspnetcore.md#12-background-work`](./docs/02-aspnetcore.md#12-background-work).

## 13. Testing

**Do:**

- xUnit v3 on `Microsoft.Testing.Platform` ([learn.microsoft.com/dotnet/core/testing/microsoft-testing-platform-intro](https://learn.microsoft.com/dotnet/core/testing/microsoft-testing-platform-intro)).
- `WebApplicationFactory<TEntryPoint>` for in-process API integration tests ([learn.microsoft.com/aspnet/core/test/integration-tests](https://learn.microsoft.com/aspnet/core/test/integration-tests)).
- Testcontainers (Postgres / SQL / Redis / etc.) over the EF Core InMemory provider.
- `TimeProvider` (and `FakeTimeProvider`) for anything time-dependent — no `DateTime.UtcNow` in product code ([learn.microsoft.com/dotnet/api/system.timeprovider](https://learn.microsoft.com/dotnet/api/system.timeprovider)).
- Aspire `DistributedApplicationTestingBuilder` for end-to-end tests across the AppHost graph.

**Don't:**

- Use the EF Core InMemory provider for relational behavior — it lies about transactions, constraints, and concurrency ([learn.microsoft.com/ef/core/providers/in-memory](https://learn.microsoft.com/ef/core/providers/in-memory)).
- Hit real cloud resources by default in tests.
- Snapshot tests over volatile fields (timestamps, GUIDs) without redaction.

See: [`docs/04-testing.md`](./docs/04-testing.md), decision tree [7 — test type](./docs/decision-trees.md#7-test-type-unit-vs-webapplicationfactory-vs-testcontainers-vs-aspire).

## 14. Performance

**Do:**

- Measure first: BenchmarkDotNet or production traces. Claims about allocations need numbers ([benchmarkdotnet.org](https://benchmarkdotnet.org)).
- Pre-size collections (`new List<T>(capacity)`, `StringBuilder(capacity)`) on hot paths.
- Pooled `HttpClient` (factory), `ArrayPool<T>`, `RecyclableMemoryStream` where it matters ([learn.microsoft.com/dotnet/api/system.buffers.arraypool-1](https://learn.microsoft.com/dotnet/api/system.buffers.arraypool-1)).
- Source-generated `System.Text.Json` (`JsonSerializerContext`) for hot serialization paths ([learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation)).
- Prefer `Span<T>` / `ReadOnlySpan<T>` for parsing on hot paths — with a benchmark.

**Don't:**

- "Perf" rewrites with no baseline benchmark.
- `string.Format` / interpolation in tight loops where a writer / `Utf8Formatter` exists.
- LINQ on hot paths "because it's nicer" without checking allocations.

See: [`docs/05-performance.md`](./docs/05-performance.md), decision trees [9 — NativeAOT](./docs/decision-trees.md#9-nativeaot-vs-jit) and [10 — Server GC](./docs/decision-trees.md#10-server-gc-vs-workstation-gc).

## 15. Cloud-native (Aspire, K8s, observability, identity)

**Do:**

- **Aspire packages — split by responsibility.** `Aspire.Hosting.*` packages live **only** in the AppHost project (`*.AppHost.csproj`); `Aspire.<Vendor>.<Tech>` client integrations live in the **service** project that consumes the resource. Aspire 9.x for `net8.0` / `net9.0`, Aspire 13 for `net10.0` ([aspire.dev](https://aspire.dev)).
- **Health probes — three canonical endpoints, mapped explicitly.** `/health/live` (in-process only), `/health/ready` (critical deps reachable), `/health/startup` (readiness with a longer grace window). Expose them on a separate non-public port or gate with a network policy. `MapDefaultEndpoints()` from Aspire ServiceDefaults only maps `/health` + `/alive` and only in Development — it is **not** a substitute for the K8s probe contract ([kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/), [learn.microsoft.com/aspnet/core/host-and-deploy/health-checks](https://learn.microsoft.com/aspnet/core/host-and-deploy/health-checks)).
- OpenTelemetry: traces + metrics + logs exported via OTLP; ASP.NET Core, `HttpClient`, and EF Core instrumentation enabled; filter `/health/*` from server traces ([opentelemetry.io/docs/specs/otel](https://opentelemetry.io/docs/specs/otel/)).
- `terminationGracePeriodSeconds` ≥ `HostOptions.ShutdownTimeout` + drain time; `preStop` hook if your platform needs it.
- Managed Identity / Workload Identity / Federated Identity Credentials over client secrets ([learn.microsoft.com/entra/workload-id/workload-identity-federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation)).
- Data Protection keys persisted to a shared store (Blob + Key Vault) for any multi-instance app issuing cookies/tokens ([learn.microsoft.com/aspnet/core/security/data-protection](https://learn.microsoft.com/aspnet/core/security/data-protection/)).

**Don't:**

- Ship `Aspire.Hosting.*` packages in production service images — they are inner-loop-only.
- Wire `Aspire.<Vendor>.<Tech>` client integrations into the AppHost — the dashboard and resource model lose visibility.
- Rename the probe endpoints (`/healthz`, `/livez`, `/ping`) — the canonical contract is `/health/live`, `/health/ready`, `/health/startup`.
- Liveness probes that hit the database — that converts a dependency outage into a self-inflicted pod restart loop.
- Client secrets in pipelines when FIC works.
- Default in-memory Data Protection keys behind a load balancer.

See: [`docs/06-cloud-native.md#10-health-checks--three-endpoints-for-k8s-not-what-servicedefaults-gives-you`](./docs/06-cloud-native.md#10-health-checks--three-endpoints-for-k8s-not-what-servicedefaults-gives-you) (probe contract owner; ch02 §18 only owns the `MapHealthChecks` plumbing), [`docs/06-cloud-native.md#1-net-aspire--what-it-is-what-it-isnt`](./docs/06-cloud-native.md#1-net-aspire--what-it-is-what-it-isnt), decision tree [16 — Aspire scope](./docs/decision-trees.md#16-aspire-scope-apphost-resource-vs-in-service-client-integration).

## 16. Security

**Do:**

- Secret scanning + push protection enabled on the repo ([docs.github.com/code-security/secret-scanning](https://docs.github.com/code-security/secret-scanning)).
- HSTS in production; HTTPS redirection on; secure cookies (`Secure`, `HttpOnly`, `SameSite`) ([learn.microsoft.com/aspnet/core/security/enforcing-ssl](https://learn.microsoft.com/aspnet/core/security/enforcing-ssl)).
- CORS: explicit origins, methods, headers — no `AllowAnyOrigin()` with credentials ([learn.microsoft.com/aspnet/core/security/cors](https://learn.microsoft.com/aspnet/core/security/cors)).
- Antiforgery for cookie-auth browser endpoints; not needed for pure bearer APIs.
- SBOM generated and dependency audit (`dotnet list package --vulnerable --include-transitive`) gating CI ([learn.microsoft.com/dotnet/core/tools/dotnet-list-package](https://learn.microsoft.com/dotnet/core/tools/dotnet-list-package)).

**Don't:**

- Secrets in repo, in container images, or in environment variables baked into images.
- Write tokens or keys to local disk; use `DataProtection`, Key Vault, or memory only.
- Disable certificate validation ("just for staging").

See: [`docs/02-aspnetcore.md#14-security`](./docs/02-aspnetcore.md#14-security), [`docs/06-cloud-native.md#8-secrets--identity--workload-identity-only`](./docs/06-cloud-native.md#8-secrets--identity--workload-identity-only).

## 17. Dependencies

**Do:**

- Central Package Management; one version per package across the solution ([learn.microsoft.com/nuget/consume-packages/central-package-management](https://learn.microsoft.com/nuget/consume-packages/central-package-management)).
- `dotnet list package --vulnerable --include-transitive` runs in CI and fails on High/Critical.
- NuGet lock files (`packages.lock.json`) for deployable apps; `RestoreLockedMode=true` in CI ([learn.microsoft.com/nuget/consume-packages/package-references-in-project-files#locking-dependencies](https://learn.microsoft.com/nuget/consume-packages/package-references-in-project-files#locking-dependencies)).
- Prefer `Microsoft.Extensions.*` (DI, Logging, Configuration, Resilience, Caching) when at parity with third-party.

**Don't:**

- Floating versions (`*`, `1.*`) in production projects.
- Pre-release packages in `main` without an explicit owner + removal date.
- Add a dependency for a one-liner you can write yourself.

See: [`patterns/monorepo.md`](./patterns/monorepo.md), [`docs/01-foundations.md`](./docs/01-foundations.md).

---

If you're not sure why a rule is here, walk the source order:

1. [`docs/decision-trees.md`](./docs/decision-trees.md) — which decision tree the rule sits under.
2. [`SCOPE.md`](./SCOPE.md) — whether the default applies to your envelope.
3. [`coverage-map.md`](./coverage-map.md) — which chapter owns the rule.
4. The owner chapter:
   [foundations](./docs/01-foundations.md) ·
   [aspnetcore](./docs/02-aspnetcore.md) ·
   [data](./docs/03-data.md) ·
   [testing](./docs/04-testing.md) ·
   [performance](./docs/05-performance.md) ·
   [cloud-native](./docs/06-cloud-native.md) ·
   [client](./docs/07-client.md).
