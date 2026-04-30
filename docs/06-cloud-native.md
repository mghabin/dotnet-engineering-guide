# Cloud-Native .NET on Kubernetes — Opinionated Best Practices

*Audience: staff/senior engineers shipping .NET 8/9/10 services to AKS (or any CNCF-conformant cluster). Companion to [`aks-shared-infra.md`](https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/aks-shared-infra.md). Baseline: **Aspire 9.x GA** for **.NET 8/9** services and **Aspire 13 GA** for **.NET 10** services.*

> **One-line position.** Use **Aspire 9.x (net8/9) or Aspire 13 (net10) for the inner loop + ServiceDefaults**, the **.NET SDK container build** for images, **OpenTelemetry + standard resilience pipelines** everywhere, and **Workload Identity + CSI Key Vault** for secrets. Everything else is either the cluster's job or the platform team's job — not your service's.

---

## 1. .NET Aspire — what it is, what it isn't

**What it is (Aspire 9.x GA for net8/9; Aspire 13 GA for net10):**

- An **inner-loop orchestration + opinionated-defaults** toolkit. The **AppHost** project is a C# program that models your distributed app (services, Postgres, Redis, RabbitMQ, Azure Storage, …) so `dotnet run` on the AppHost (or `aspire run` with the Aspire CLI) spins up the whole graph with a dashboard, OTel wired up, and connection strings injected.
- **ServiceDefaults** is a shared project you `AddServiceDefaults()` into every service: OpenTelemetry (traces/metrics/logs via OTLP), service discovery, standard resilience on `HttpClient`, and a small set of health checks (`/health` + `/alive`) **mapped only in the Development environment** by `MapDefaultEndpoints()`. Note: when you add any **client integration** to a service in an Aspire solution, it pulls in the `ServiceDefaults` reference and calls `AddServiceDefaults()` for you — you don't have to remember to call it yourself.
- **Two distinct integration families** — keep them straight:
  - **Hosting integrations** — `Aspire.Hosting.*` packages (e.g., `Aspire.Hosting.PostgreSQL`, `Aspire.Hosting.Redis`, `Aspire.Hosting.Azure.ServiceBus`). **AppHost-only.** They model resources, run containers locally, and emit connection info. **Never reference these from a service project.**
  - **Client integrations** — `Aspire.<Vendor>.<Tech>` packages (e.g., `Aspire.Npgsql`, `Aspire.StackExchange.Redis`, `Aspire.Azure.Messaging.ServiceBus`). **Service-project packages.** They register typed clients in DI, wire configuration, register health checks, and add OTel instrumentation. There is **no `.Client` suffix** on the package name.
- **Dashboard** (also usable standalone via OTLP) is the best local trace/log/metric UI in the ecosystem.

**Version pin guidance:**

- net8 / net9 services → **Aspire 9.x** (still GA-shipping for those TFMs).
- net10 services → **Aspire 13** (versions jumped 9.x → 13.0, skipping 10–12 to align with the .NET 10 wave). Aspire 13 requires the **.NET 10 SDK**, introduces **polyglot AppHost** support (Python, JS resources alongside .NET), and a new `aspire do` build/publish/deploy pipeline. The AppHost csproj is now `<Project Sdk="Aspire.AppHost.Sdk/13.0.0">` and `Aspire.Hosting.AppHost` is **no longer an explicit `PackageReference`** — the SDK encapsulates it.
- The rebrand from **.NET Aspire → Aspire** and the new docs hub at **<https://aspire.dev>** reflect the multi-language direction.

**What it is *not*:**

- **Not a deployment runtime.** AppHost does not run in production. `aspire publish` / `azd` / Bicep / Helm generate manifests; the cluster runs the service, not AppHost. On Aspire 13, **`aspire do`** is the forward direction for build/publish/deploy pipelines (parallel execution, dependency tracking).
- **Not a PaaS.** It doesn't replace AKS, ACA, or App Service.
- **Not a service mesh.** No mTLS, no L7 policy, no traffic shifting.
- **Not required.** A well-factored `Program.cs` with the same OTel/health/resilience setup is equivalent — Aspire just removes the boilerplate.

**Do / Don't:**

- ✅ Adopt Aspire when you have ≥2 services, ≥1 dependency (Postgres/Redis/bus), and devs currently `docker-compose up` by hand.
- ✅ Put `ServiceDefaults` in every new service, even solo ones — you get OTel + service discovery + resilience for free. (Adding any client integration does this for you.)
- ✅ Use Aspire **client** integrations over hand-rolled wiring; they set the OTel `ActivitySource`/meter names correctly and register the right health checks.
- ❌ Don't ship `Aspire.Hosting.*` to prod. Hosting packages belong to the AppHost project, which is a dev-time tool.
- ❌ Don't assume `AddServiceDefaults()` gives you Kubernetes probes — it doesn't (see §10).
- ❌ Don't let Aspire's generated Bicep/Helm be the *final* deployment artifact for a regulated workload — it's a starting point; platform teams own the real manifests.

**Sources:**

- Aspire overview — <https://aspire.dev/get-started/aspire-overview/>
- Integrations overview (hosting vs client) — <https://aspire.dev/integrations/overview/> and <https://learn.microsoft.com/dotnet/aspire/fundamentals/integrations-overview>
- ServiceDefaults — <https://aspire.dev/get-started/csharp-service-defaults/>
- Microsoft Learn .NET Aspire — <https://learn.microsoft.com/dotnet/aspire/>
- Aspire 9 What's New (covers 9 → 13 evolution) — <https://learn.microsoft.com/dotnet/aspire/whats-new/dotnet-aspire-9>

---

## 2. Containerization — no Dockerfile if you can avoid it

**Default: SDK container build.** .NET 8+ ships a first-party container target.

```bash
dotnet publish -c Release /t:PublishContainer \
  -p:ContainerRegistry=myregistry.azurecr.io \
  -p:ContainerRepository=catalog-api \
  -p:ContainerImageTag=$GITHUB_SHA \
  -p:ContainerFamily=noble-chiseled \
  -p:RuntimeIdentifier=linux-x64
```

What you get by default:

- Base image: the `mcr.microsoft.com/dotnet/aspnet` family for the targeted TFM. Pick `ContainerFamily=noble-chiseled` (Ubuntu 24.04, distroless-style) or `noble-chiseled-extra` if you need `icu`/`tzdata`.
- **Non-root by default** (`UID 1654 / app`). Don't undo this.
- Reproducible layers, SBOM-friendly, deterministic tags.

**Base image trade-offs:**

| Family | Size | Shell | Use when |
|---|---|---|---|
| `noble-chiseled` | smallest (~110 MB aspnet) | no | default for APIs/workers |
| `noble-chiseled-extra` | +~10 MB | no | needs `tzdata`, globalization |
| `noble` (full Ubuntu) | ~220 MB | yes | debug only, never prod |
| `alpine` (musl) | small | yes | only if you *need* musl; expect globalization/ICU quirks |
| `azurelinux` (CBL-Mariner) | small | yes | Microsoft-supported distro; fine if your platform standardizes on it |

**Do / Don't:**

- ✅ Prefer `dotnet publish /t:PublishContainer` over a Dockerfile. One less artifact, reproducible by MSBuild.
- ✅ Chiseled + non-root + read-only root filesystem (`securityContext.readOnlyRootFilesystem: true`).
- ✅ Multi-arch: `-p:ContainerRuntimeIdentifiers="linux-x64;linux-arm64"` — ARM nodes are ~20% cheaper on AKS.
- ✅ NativeAOT (`PublishAot=true`) for short-lived workers, CLI tools, functions; base image is the matching `runtime-deps:*-noble-chiseled`.
- ❌ Don't run as root. Don't `chmod 777`. Don't `apt-get install curl` "for health checks" — chiseled images have no shell or package manager; use an HTTP probe.
- ❌ Don't bake secrets, connection strings, or `appsettings.Production.json` into the image.
- ❌ Don't use `:latest`. Ever. Tag with immutable SHAs; promote by digest.

**Sources:**

- Chiseled Ubuntu containers for .NET — <https://github.com/dotnet/dotnet-docker/blob/main/documentation/ubuntu-chiseled.md>
- `dotnet publish /t:PublishContainer` — <https://learn.microsoft.com/dotnet/core/docker/publish-as-container>
- .NET container images — <https://learn.microsoft.com/dotnet/core/docker/container-images>

> **Chiseled in .NET 10 — extensible slices.** Featured tags now include both `8.0-noble-chiseled` and `10.0-noble-chiseled` for `aspnet`, `runtime`, and `runtime-deps` repos under `mcr.microsoft.com/dotnet/*`. There is still **no chiseled SDK image** (build with full Ubuntu SDK, run on chiseled). **New in .NET 10:** the chisel manifest is shipped, so you can install additional package slices (e.g., `libicu74_libs`, `tzdata-legacy_zoneinfo`) via `chisel`/`chisel-wrapper` from `canonical/chisel-releases` — this is your escape hatch when you need ICU/tzdata/native deps without leaving chiseled. **.NET 8/9 chiseled images remain extension-impossible** — you must switch to `noble-chiseled-extra` or full `noble`.

---

## 3. Kubernetes / AKS — probes, limits, GC, shutdown

### Probes — three, not one

```yaml
startupProbe:   { httpGet: { path: /health/startup,  port: 8080 }, failureThreshold: 30, periodSeconds: 2 }
livenessProbe:  { httpGet: { path: /health/live,     port: 8080 }, periodSeconds: 10,   failureThreshold: 3 }
readinessProbe: { httpGet: { path: /health/ready,    port: 8080 }, periodSeconds: 5,    failureThreshold: 2 }
```

- **Startup** gates the other two — essential for slow-starting JIT apps and DB migrations.
- **Liveness** = "is the process wedged?" — checks nothing external. Restart-only.
- **Readiness** = "can I serve traffic right now?" — checks DB/Redis/bus. Removes from EndpointSlice (no traffic), does *not* restart. Note: Pod termination already flips endpoint `ready=false` on delete, so a readiness probe is **not strictly required** for graceful drain — but it is required to gate traffic during slow startup or transient dep outages.

**Rule:** a failing downstream dependency must **never** trip liveness. Otherwise Postgres hiccups cascade into pod restart storms.

These three endpoints are **not** what `MapDefaultEndpoints()` from Aspire ServiceDefaults gives you — see §10.

### Requests/limits + .NET GC — and what QoS class you actually get

The Kubernetes QoS class is a function of how you set requests vs limits on **every** container in the pod:

- **Guaranteed** — every container has CPU **and** memory `requests == limits` (and both set). Lowest eviction priority, predictable.
- **Burstable** — at least one resource has a request set, and requests ≠ limits (or limits are unset).
- **BestEffort** — no requests or limits set anywhere. Don't ship this.

The naive "set both" recipe below is **Burstable**, not Guaranteed:

```yaml
# Burstable QoS (cpu request != limit, memory request != limit)
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits:   { cpu: "1",    memory: "512Mi" }
```

Pick one and **document the choice**:

```yaml
# Guaranteed QoS — latency-sensitive APIs that must not be throttled or evicted
resources:
  requests: { cpu: "500m", memory: "512Mi" }
  limits:   { cpu: "500m", memory: "512Mi" }
```

```yaml
# Burstable QoS — workers / batch where headroom > predictability
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits:   { cpu: "1",    memory: "512Mi" }
```

**.NET GC notes:**

- The .NET GC reads cgroup memory limits to size the heap. Always set a memory limit on .NET pods.
- `DOTNET_GCHeapHardLimit` / `DOTNET_GCConserveMemory` for memory-tight workers.
- `DOTNET_TieredPGO=1` is on by default in current .NET — leave it.
- Server GC is on by default on multi-core; don't override.
- **Never** set memory limit without a request — scheduler will misplace you.

### CPU model: requests vs limits

Two different Linux primitives:

- **Requests** drive **scheduling** (which node a pod lands on) and translate into **CPU shares** (relative weight under contention). They never throttle a pod that has CPU available.
- **Limits** are enforced as a **CFS quota** — a hard cap measured per 100 ms period. Once a pod uses its quota, the kernel **throttles** it for the remainder of the period, regardless of idle CPU on the node. This shows up as p99 spikes that don't correlate with load.

Pragmatic guidance:

- **Latency-sensitive API**: set CPU `request == limit` (Guaranteed) so the share matches the cap and the slice is predictable.
- **Throughput-sensitive worker on a dedicated node pool**: consider **omitting the CPU limit** so the pod can burst. You accept that other pods on the same node may compete and that the node is intentionally overcommitted; the platform team must size requests honestly so the node isn't oversold. PDBs and PriorityClasses help with eviction order — they do **not** substitute for CPU fairness.
- Always keep a **memory limit**: memory is incompressible, and going over it gets you OOM-killed, not throttled.

### Graceful shutdown

```yaml
terminationGracePeriodSeconds: 45
lifecycle:
  preStop:
    httpGet:
      path: /shutdown   # app-implemented drain endpoint
      port: 8080
```

```csharp
// HTTP-based preStop works on chiseled images (no shell available).
// "/bin/sleep" execs do NOT work on chiseled images — there is no /bin/sh, no coreutils.
app.MapPost("/shutdown", (IHostApplicationLifetime life) =>
{
    // Mark unready, let endpoint slices propagate, then return.
    // Combine with a short Task.Delay or rely on app-level drain logic.
    life.StopApplication();
    return Results.Accepted();
});

builder.Services.Configure<HostOptions>(o =>
{
    o.ShutdownTimeout = TimeSpan.FromSeconds(30); // < terminationGracePeriodSeconds
    o.BackgroundServiceExceptionBehavior = BackgroundServiceExceptionBehavior.StopHost;
});
```

- Implement `IHostApplicationLifetime.ApplicationStopping` in workers: stop pulling from the queue, drain in-flight, then return.
- ASP.NET Core drains HTTP automatically; the only reason requests get killed is `ShutdownTimeout` < actual drain time.
- Align `terminationGracePeriodSeconds` ≥ `HostOptions.ShutdownTimeout` + readiness-probe drain window (typical: 60s pod / 30s host shutdown / 10–15s drain). If they invert, the kubelet SIGKILLs you mid-drain.
- If you must use `exec` for `preStop`, you cannot use chiseled images — they ship without a shell or `sleep`. Either move to an HTTP `preStop`, ship `noble`/`noble-chiseled-extra` (extra still has no shell), or accept the size penalty of `noble`.

### HPA / KEDA / PDB

- **HPA on CPU is a last resort.** CPU is a lagging indicator. Prefer:
  - **KEDA** on queue depth (Service Bus, RabbitMQ, Kafka lag) for workers.
  - HPA on **custom OTel metrics** (RPS, p95 latency) via Prometheus adapter for APIs.
- **PDB always.** `minAvailable: 1` (or `maxUnavailable: 1` for HA workloads) — otherwise node drains take everyone.
- Scale-to-zero only for truly bursty non-interactive workloads. Cold-start on JIT .NET is 2–5s; NativeAOT is ~100ms.

**Sources:**

- Kubernetes Pod QoS — <https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/>
- Manage resources for containers (CPU shares & CFS quota) — <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/>
- Probes (canonical concept page) — <https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/>
- Container lifecycle hooks — <https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/>
- Hosted services / `HostOptions.ShutdownTimeout` — <https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services>
- Chiseled image constraints — <https://github.com/dotnet/dotnet-docker/blob/main/documentation/ubuntu-chiseled.md>

---

## 4. Configuration — ConfigMap + CSI Key Vault, not `appsettings.Production.json`

Precedence (high → low) in production:

1. Env vars (K8s `env` / `envFrom`) — highest, CSI-mounted secrets exposed here.
2. ConfigMap mounted as files (`/config/*.json`) via `AddJsonFile(..., reloadOnChange: true)`.
3. `appsettings.{Environment}.json` — safe defaults only, never secrets.
4. `appsettings.json` — defaults.

**Do:**

- ✅ Non-sensitive config → ConfigMap mounted as files or `envFrom`.
- ✅ Sensitive config → **[Secrets Store CSI Driver](https://learn.microsoft.com/azure/aks/csi-secrets-store-driver) + Azure Key Vault provider**, mounted as files, **synced into env vars** via `SecretProviderClass`.
- ✅ `builder.Configuration.AddKeyPerFile("/mnt/secrets", optional: true, reloadOnChange: true);` — one secret per file, rotated in-place.
- ✅ Prefer **immutable rollouts** (restart pods on config change via checksum annotation) over `reloadOnChange` for anything that affects auth, connection strings, or feature flags that alter behavior. `reloadOnChange` is fine for log levels and knobs.

**Don't:**

- ❌ Native K8s `Secret` objects in plain YAML. They're base64, not encrypted. Use CSI → Key Vault.
- ❌ `appsettings.Production.json` checked in with anything but placeholders.
- ❌ Environment-specific code paths keyed off `IHostEnvironment.EnvironmentName == "Production"`. Config should drive behavior, not env-name branches.

**Sources:**

- CSI Secrets Store Driver on AKS — <https://learn.microsoft.com/azure/aks/csi-secrets-store-driver>
- ASP.NET Core configuration — <https://learn.microsoft.com/aspnet/core/fundamentals/configuration/>

---

## 5. Observability — OpenTelemetry, one SDK, three signals

ServiceDefaults wires the **instrumentation and logging providers**; the **OTLP exporter only activates when the standard env var `OTEL_EXPORTER_OTLP_ENDPOINT` is set**. That's how Aspire's AppHost lights up the dashboard locally and how a sidecar/Daemonset OTel Collector lights it up in prod. Hand-rolled `Program.cs` should follow the same pattern:

```csharp
var otel = builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(builder.Environment.ApplicationName))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation(o =>
        {
            // Don't trace probe traffic — otherwise every 5s readiness check is a span.
            o.Filter = ctx =>
                !ctx.Request.Path.StartsWithSegments("/health") &&
                !ctx.Request.Path.StartsWithSegments("/alive");
        })
        .AddHttpClientInstrumentation()
        .AddSource("EntraAuth.*"))
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddMeter("EntraAuth.*"));

builder.Logging.AddOpenTelemetry(o =>
{
    o.IncludeFormattedMessage = true;
    o.IncludeScopes = true;
});

// Activate the OTLP exporter only when the endpoint is configured.
// This is what ServiceDefaults does internally and matches the OTel spec env vars.
if (!string.IsNullOrWhiteSpace(builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]))
{
    otel.UseOtlpExporter(); // applies to traces, metrics, logs
}
```

- **Exporter: OTLP only.** No vendor SDKs in code. Rewire the collector, not the app.
- **Pin core packages to `OpenTelemetry` 1.15.x** (Traces, Metrics, **and** Logs are all stable). The OTLP exporter is `OpenTelemetry.Exporter.OpenTelemetryProtocol`. Instrumentation packages (`OpenTelemetry.Instrumentation.AspNetCore`, `…HttpClient`, `…Runtime`) live in **`opentelemetry-dotnet-contrib`**, not the core repo — they version independently. Note: in `OpenTelemetry` 1.15.3, OTLP disk retry now requires `OTEL_DOTNET_EXPERIMENTAL_OTLP_DISK_RETRY_DIRECTORY_PATH` to be set explicitly when `OTEL_DOTNET_EXPERIMENTAL_OTLP_RETRY=disk` (no temp-dir fallback).
- **Dev**: OTLP → Aspire dashboard (`OTEL_EXPORTER_OTLP_ENDPOINT` auto-injected by AppHost).
- **Prod**: OTLP → OpenTelemetry Collector → Grafana Tempo/Loki/Mimir **or** Azure Monitor OTLP. Don't put vendor SDKs on the hot path.
- **Filter probes** out of HTTP server traces — otherwise every 5s readiness check is a span and a metric data point. ServiceDefaults already excludes its `/health` and `/alive` paths; if you map your own probe paths (see §10), filter those too.
- **Query strings are redacted by default** in `OpenTelemetry.Instrumentation.AspNetCore` (PII-safe). If you genuinely need them on `url.path`/`url.full`, opt in with `OTEL_DOTNET_EXPERIMENTAL_ASPNETCORE_DISABLE_URL_QUERY_REDACTION=true` — and then make sure your sampler/exporter pipeline doesn't ship the raw query to a vendor.
- **Semantic conventions**: use stable names (`http.request.method`, `http.response.status_code`, `http.route`, `db.system.name`, `messaging.system`). Don't invent `my.http.verb`. AspNetCore instrumentation tracks http-spans semconv ≥ 1.23.
- **ActivitySource naming**: `{Org}.{Product}.{Component}` — e.g., `EntraAuth.Catalog.Domain`. Stable. Filterable in sampling rules.
- **Log correlation**: logs auto-include `TraceId`/`SpanId` when the OTel logging provider is registered. Turn off Serilog's parallel trace injection if both are wired — pick one.

**Sampling:** tail-based in the collector, not head-based in the app. Sample errors and slow traces at 100%, baseline at 5–10%.

**Sources:**

- OpenTelemetry .NET — <https://opentelemetry.io/docs/languages/dotnet/>
- `open-telemetry/opentelemetry-dotnet` releases (1.15.x) — <https://github.com/open-telemetry/opentelemetry-dotnet/releases/latest>
- `OpenTelemetry.Instrumentation.AspNetCore` (Filter/Enrich, query redaction) — <https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/main/src/OpenTelemetry.Instrumentation.AspNetCore/README.md>
- OTel SDK environment variables (`OTEL_EXPORTER_OTLP_ENDPOINT`) — <https://opentelemetry.io/docs/specs/otel/protocol/exporter/>
- Aspire ServiceDefaults (OTLP activation, probe filtering) — <https://aspire.dev/get-started/csharp-service-defaults/>

---

## 6. Resilience — Polly v8 standard pipelines, not hand-rolled retries

`Microsoft.Extensions.Http.Resilience` (Polly v8 under the hood, sitting on `Microsoft.Extensions.Resilience`) ships a standard HTTP pipeline with five strategies in fixed order, outermost → innermost:

| # | Strategy | Default |
|---|---|---|
| 1 | Rate limiter | Queue 0, Permit 1000 (per pipeline) |
| 2 | Total request timeout | 30s |
| 3 | Retry | Max 3, exponential, jitter on, base delay 2s |
| 4 | Circuit breaker | Failure ratio 10%, min throughput 100, sampling 30s, break 5s |
| 5 | Attempt timeout | 10s |

Triggers (handled as transient): HTTP `5xx`, `408`, `429`, plus `HttpRequestException` and `TimeoutRejectedException`.

```csharp
builder.Services.AddHttpClient<CatalogClient>()
    .AddStandardResilienceHandler(o =>
    {
        o.Retry.MaxRetryAttempts = 3;
        o.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
        o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(2);
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(10);

        // Retries fire for ALL HTTP methods by default — including POST/PUT/PATCH/DELETE.
        // Disable for non-idempotent verbs to avoid duplicate side effects.
        o.Retry.DisableForUnsafeHttpMethods();
    });
```

> **POST-retry hazard.** `AddStandardResilienceHandler` retries every method by default. If your POSTs are not idempotent (no idempotency key, no dedup store), call **`DisableForUnsafeHttpMethods()`** on the retry options — or better, design every mutating endpoint to take an `Idempotency-Key` header and dedup server-side.

> **Don't stack handlers.** `AddStandardResilienceHandler` is *the* pipeline; do **not** chain a second `AddResilienceHandler` or a hand-rolled `Polly` handler on the same `HttpClient` — you'll get nested retries and circuit breakers that interact in surprising ways. To customize, call **`RemoveAllResilienceHandlers()`** first, then re-add a single configured pipeline.

**Do:**

- ✅ `AddStandardResilienceHandler` on every outbound `HttpClient`. Make it the default.
- ✅ Call `DisableForUnsafeHttpMethods()` on the retry strategy unless you have idempotency keys end-to-end.
- ✅ **Idempotency keys** on POSTs that mutate money/state. `Idempotency-Key` header, dedup store with TTL in Redis.
- ✅ **Outbox pattern** for "save row + publish event" — never two-phase commit. Transactional outbox table, a relay pushes to the bus. There is **no first-party Aspire outbox integration**; pick one of: **Dapr** state-management outbox (transactional state store + `outboxPublishPubsub`/`outboxPublishTopic` metadata), or a library outbox in **MassTransit** / **NServiceBus** / **Wolverine**. Aspire integrates with these only at the broker level (e.g., `Aspire.Hosting.RabbitMQ`, `Aspire.Azure.Messaging.ServiceBus`).
- ✅ **Chaos**: add `AddChaosLatency` / `AddChaosFault` strategies behind a feature flag in non-prod to test resilience paths.

**Don't:**

- ❌ Retry on non-idempotent verbs without an idempotency key. You'll double-charge customers.
- ❌ Stack retries at multiple layers (client + gateway + mesh). Pick one. And don't stack two resilience handlers on the same `HttpClient`.
- ❌ Circuit-break on **every** exception type. Exclude `OperationCanceledException` from request cancellation.

**Sources:**

- HTTP resilience (`AddStandardResilienceHandler`, defaults, `DisableForUnsafeHttpMethods`, `RemoveAllResilienceHandlers`) — <https://learn.microsoft.com/dotnet/core/resilience/http-resilience>
- Polly v8 — <https://www.pollydocs.org/>
- Dapr state-management outbox — <https://docs.dapr.io/developing-applications/building-blocks/state-management/howto-outbox/>

---

## 7. Service-to-service

**Auth:** see [`aks-shared-infra.md`](https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/aks-shared-infra.md) for the tenant-allow-list + `azp` client-app pattern. The short version: validate JWTs in the service with the shared `EntraAuth.Auth` library; do **not** use a mesh for authZ.

### Service discovery — what the URI scheme actually means

Aspire ServiceDefaults adds `Microsoft.Extensions.ServiceDiscovery` to every `HttpClient`. The contract is a **logical URI** that the resolver turns into one or more concrete endpoints:

- **`http://catalog`** / **`https://catalog`** — resolve `catalog` to its endpoint, must be that scheme.
- **`https+http://catalog`** — multi-scheme: prefer `https` endpoints, fall back to `http`. Useful when dev runs http and prod runs https for the same logical service.
- **`https://_endpoint.catalog`** — **named endpoint** form. Resolves to the endpoint named `endpoint` on service `catalog` (services may expose multiple — e.g., `_grpc.catalog`, `_admin.catalog`).

```csharp
// AppHost (dev) — model the dependency, scheme, and endpoint name.
var catalog = builder.AddProject<Projects.Catalog_Api>("catalog")
                     .WithHttpsEndpoint(name: "grpc"); // additional named endpoint

builder.AddProject<Projects.Web>("web")
       .WithReference(catalog);    // injects services__catalog__https/http into Web

// Service code — use the logical URI; the resolver picks the real endpoint.
builder.Services.AddHttpClient<CatalogClient>(c => c.BaseAddress = new Uri("https+http://catalog"));
builder.Services.AddHttpClient<CatalogGrpcClient>(c => c.BaseAddress = new Uri("https://_grpc.catalog"));
```

**Resolvers — pick the right one for the platform:**

- **Configuration resolver** (default in dev) — reads `Services:<name>:<scheme>:<index>` from `IConfiguration`. AppHost populates this; in prod, ConfigMap/env vars can populate it for static topologies. Multi-scheme fallback uses the `+` separator (e.g., `https+http://basket`); allowed schemes are filtered via `ServiceDiscoveryOptions.AllowedSchemes`.
- **Pass-through resolver** (`AddPassThroughServiceEndpointProvider`) — turn the logical URI into a literal hostname and let the platform (Kubernetes `Service` / CoreDNS) resolve it. Use this on K8s when each logical service maps 1:1 to a `Service` of the same name.
- **DNS SRV resolver** (`AddDnsSrvServiceEndpointProvider`) — query SRV records to pick up multiple endpoints per service. Use this on K8s when you expose multiple **named** ports: create a headless `Service` (`clusterIP: None`) with named ports, then resolve via `https+http://_dashboard.basket` (the `_endpoint` segment selects the named port). This is the canonical K8s pattern when one logical service has more than one port (e.g., HTTP + admin + gRPC).

```csharp
// Service project — choose the resolver that matches your runtime.
builder.Services.AddServiceDiscovery();
// or, for K8s headless Services with named ports:
// builder.Services.AddServiceDiscoveryCore();
// builder.Services.AddDnsSrvServiceEndpointProvider();

// Apply to every HttpClient by default:
builder.Services.ConfigureHttpClientDefaults(http => http.AddServiceDiscovery());
```

### Transport

- **HTTP/JSON** by default. Boring, debuggable, works with every tool.
- **gRPC** when: internal, schema-stable, hot path, streaming. Not for public APIs (browser story still painful).
- **HTTP/2 keep-alive** and connection pooling matter more than protocol choice for p99.

### Service mesh (Istio / Linkerd / AKS Istio addon)

Worth it only for:

- Uniform **mTLS** between services (if you can't terminate at the edge + trust the CNI).
- Traffic shifting, canary, retry policies **you can't get from the standard resilience pipeline**.
- L7 telemetry when you have non-.NET services you can't instrument.

**Not** for auth. AuthN/Z stays in the service. A mesh is plumbing, not policy.

**Sources:**

- Aspire service discovery (incl. K8s DNS SRV named endpoints) — <https://learn.microsoft.com/dotnet/aspire/service-discovery/overview>
- `Microsoft.Extensions.ServiceDiscovery` — <https://learn.microsoft.com/dotnet/core/extensions/service-discovery>
- aspire.dev service discovery — <https://aspire.dev/fundamentals/service-discovery/>
- Kubernetes DNS for Services — <https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/>

---

## 8. Secrets & identity — Workload Identity only

Preference order:

1. **Azure AD Workload Identity + Federated Identity Credentials (FIC)** — pod assumes a managed identity via a projected service-account token. No secrets on disk, no client secrets anywhere. This is the default.
2. **User-assigned Managed Identity** (for non-K8s compute, or legacy).
3. **Key Vault references** from Key Vault (via CSI) — for third-party secrets (Stripe, SendGrid) that can't do MI.
4. **Client secrets / certs** — last resort, with rotation SLA and alerting.

```yaml
serviceAccountName: catalog-sa
# catalog-sa has annotation: azure.workload.identity/client-id=<uami-client-id>
# FIC links uami ↔ (cluster issuer, namespace, sa-name)
```

```csharp
// In code — never a client secret
var credential = new DefaultAzureCredential();
// Works locally (az login), in CI (OIDC), in AKS (Workload Identity). One code path.
```

**Don't** put secrets in `appsettings.Production.json`. Don't commit `.env`. Don't pass secrets as command-line args (they end up in `ps`).

**Sources:**

- Azure Workload Identity on AKS — <https://learn.microsoft.com/azure/aks/workload-identity-overview>
- `DefaultAzureCredential` — <https://learn.microsoft.com/dotnet/api/azure.identity.defaultazurecredential>

---

## 9. Data Protection — shared key ring, scoped per app + environment

ASP.NET Core Data Protection encrypts antiforgery tokens, cookies, OAuth state, `IDataProtector` payloads. **Keys are per-replica unless you persist them.** On rollout, half your users get 500s until old pods die.

**Correct setup for K8s:**

```csharp
var appName  = $"catalog-api.{builder.Environment.EnvironmentName.ToLowerInvariant()}";
var blobUri  = new Uri("https://contosokeys.blob.core.windows.net/data-protection/keys.xml");
// Versionless Key Vault key URI — required so KV-side rotation works without a code change.
var keyId    = new Uri("https://contoso.vault.azure.net/keys/data-protection");
var credential = new DefaultAzureCredential();   // ManagedIdentityCredential in prod

builder.Services
    .AddDataProtection()
    // Scope to the trust boundary that must share protected payloads:
    // app + environment. NEVER share a key ring across environments.
    .SetApplicationName(appName)
    .PersistKeysToAzureBlobStorage(blobUri, credential)        // per-env container/blob
    .ProtectKeysWithAzureKeyVault(keyId, credential);          // envelope-encrypted at rest
```

**NuGet:** `Azure.Extensions.AspNetCore.DataProtection.Blobs` + `Azure.Extensions.AspNetCore.DataProtection.Keys`.

**Required RBAC** on the Workload Identity / managed identity the pod uses:

- **`Storage Blob Data Contributor`** — scoped to the data-protection blob container (read/write the keys XML).
- **`Key Vault Crypto User`** — scoped to the wrapping key in Key Vault (wrap/unwrap only; no key management).

- `ApplicationName` must match across **replicas of the same app in the same environment**, or decryption fails on the next pod.
- **Do not** share `ApplicationName` across environments (dev/stage/prod). Sharing means a token minted in dev validates in prod and vice versa — a privilege/replay risk and an operational footgun. Use a separate blob path/container and Key Vault key per environment too.
- Use a **versionless** Key Vault key URI (`…/keys/<name>` without a `/<version>` suffix) so KV-side key rotation is picked up automatically. If you pin a version, rotation on the KV side breaks decryption of newly-wrapped keys.
- Blob + Key Vault is the canonical pair. Alternatives: Redis (ephemeral-ish, avoid for production key material), cluster PV (works but operationally painful).
- Rotate Key Vault keys on a schedule; Data Protection handles ring rollover automatically.

**Symptom of getting this wrong:** intermittent `CryptographicException: The key {guid} was not found in the key ring` after deploy. Usually in antiforgery / auth cookie / OIDC state.

**Sources:**

- Data Protection configuration — <https://learn.microsoft.com/aspnet/core/security/data-protection/configuration/overview>
- Key storage providers (Azure Blob + Key Vault) — <https://learn.microsoft.com/aspnet/core/security/data-protection/implementation/key-storage-providers>
- `SetApplicationName` semantics — <https://learn.microsoft.com/aspnet/core/security/data-protection/configuration/overview#setapplicationname>

---

## 10. Health checks — three endpoints for K8s, not what ServiceDefaults gives you

**ServiceDefaults reality check.** `MapDefaultEndpoints()` from Aspire ServiceDefaults maps **only `/health` and `/alive`**, and **only when `app.Environment.IsDevelopment()`**. It does **not** give you `live`/`ready`/`startup`, and it deliberately does not expose health endpoints in production by default. For Kubernetes you must map your own probe endpoints explicitly:

```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(cs,                   name: "postgres", tags: ["ready"])
    .AddRedis(cs,                    name: "redis",    tags: ["ready"])
    .AddAzureServiceBusQueue(/*...*/, tags: ["ready"])
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"]);

// Map your own K8s-facing probe endpoints — independent of MapDefaultEndpoints().
app.MapHealthChecks("/health/live",    new HealthCheckOptions { Predicate = r => r.Tags.Contains("live")  });
app.MapHealthChecks("/health/ready",   new HealthCheckOptions { Predicate = r => r.Tags.Contains("ready") });
app.MapHealthChecks("/health/startup", new HealthCheckOptions { Predicate = r => r.Tags.Contains("ready") });
```

- Liveness has **no** external checks. Only "am I alive in-process?"
- Readiness checks critical dependencies. Non-critical (telemetry, feature flags) → degraded, not unhealthy.
- Startup = readiness with a longer grace window. Same checks, different probe.
- Expose health on a **separate non-public port** or gate with a network policy. Don't leak dependency topology via `/health`.
- **Filter `/health/*` from OTel ASP.NET Core instrumentation** (see §5) so probes don't dominate your traces and metrics.

### Caching expensive checks — there is no `CacheDuration` on `HealthCheckOptions`

`HealthCheckOptions` does **not** expose a `CacheDuration` property. To prevent a 5s probe storm from hammering Postgres or Redis, use one of:

- **Wrap an expensive check in a cached one:**

  ```csharp
  public sealed class CachedHealthCheck(IHealthCheck inner, TimeSpan ttl) : IHealthCheck
  {
      private readonly object _gate = new();
      private DateTimeOffset _expiresAt;
      private HealthCheckResult _last;

      public async Task<HealthCheckResult> CheckHealthAsync(
          HealthCheckContext context, CancellationToken ct = default)
      {
          lock (_gate)
          {
              if (DateTimeOffset.UtcNow < _expiresAt) return _last;
          }
          var result = await inner.CheckHealthAsync(context, ct);
          lock (_gate) { _last = result; _expiresAt = DateTimeOffset.UtcNow.Add(ttl); }
          return result;
      }
  }
  ```

- **Background health check publishers** (`IHealthCheckPublisher` + `HealthCheckPublisherOptions.Period`) — run dependency checks once per N seconds in the background; probes read the cached snapshot.
- **Library-specific throttling** (e.g., AspNetCore.HealthChecks.* `HealthCheckRegistration` with a custom `IHealthCheck` that throttles internally).

**Sources:**

- ASP.NET Core health checks — <https://learn.microsoft.com/aspnet/core/host-and-deploy/health-checks>
- `HealthCheckOptions` API — <https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.diagnostics.healthchecks.healthcheckoptions>
- `IHealthCheckPublisher` — <https://learn.microsoft.com/dotnet/api/microsoft.extensions.diagnostics.healthchecks.ihealthcheckpublisher>
- Aspire ServiceDefaults endpoints (`MapDefaultEndpoints`) — <https://aspire.dev/get-started/csharp-service-defaults/>

---

## 11. Graceful shutdown — drain, don't drop

Checklist per service:

- [ ] `HostOptions.ShutdownTimeout` set (default 30s is fine if `terminationGracePeriodSeconds` ≥ 45s).
- [ ] HTTP `preStop` hook (or app-level drain) so ingress / endpoint-slice propagation finishes before the process exits. **Do not** use `exec` `["/bin/sleep","5"]` on chiseled images — there is no shell.
- [ ] Background workers: `ExecuteAsync(CancellationToken stoppingToken)` respects the token, drains in-flight unit of work, commits offsets.
- [ ] Message-bus consumers: stop `ReceiveMessagesAsync`, finish in-flight handlers, `CompleteMessageAsync`, then close.
- [ ] gRPC servers: Kestrel handles `GOAWAY` on SIGTERM automatically; don't fight it.
- [ ] HTTP clients: dispose on shutdown to close keep-alive sockets cleanly (otherwise you leak TIME_WAITs on neighbors).

```csharp
public sealed class OutboxRelay(IBus bus, ILogger<OutboxRelay> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try { await PumpOnce(stoppingToken); }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) { return; }
            catch (Exception ex) { log.LogError(ex, "pump failed"); await Task.Delay(1000, stoppingToken); }
        }
    }
}
```

**Sources:**

- Hosted services / `HostOptions.ShutdownTimeout` — <https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services>
- Pod termination — <https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination>

---

## 12. CI/CD — reproducible, signed, scanned

**Must-haves:**

- ✅ **GitHub Actions OIDC + FIC** for Azure deploy. No `AZURE_CLIENT_SECRET` in secrets.
- ✅ **SBOM** via `Microsoft.Sbom.Targets` (`GenerateSBOM=true`). Attach to release.
- ✅ **Container signing**: Sigstore `cosign` or Notation. Verify in admission controller (Ratify / Kyverno / Gatekeeper).
- ✅ **Dependency scan** in PR: `dotnet list package --vulnerable --include-transitive`, `dotnet nuget audit`, plus GHAS/Dependabot.
- ✅ **Reproducible builds**: `ContinuousIntegrationBuild=true`, `Deterministic=true`, pinned SDK via `global.json`.
- ✅ **Image digests**, not tags, in Helm/Kustomize `values.yaml`. Promote by digest across dev→stage→prod.

**Don't:**

- ❌ Build once per environment. Build once, promote.
- ❌ Use long-lived service principals. FIC is free and better.
- ❌ Skip `dotnet nuget audit` because "it's noisy." Set a severity threshold, triage weekly.

**Sources:**

- GitHub Actions OIDC → Azure — <https://learn.microsoft.com/azure/developer/github/connect-from-azure-openid-connect>
- `dotnet nuget audit` — <https://learn.microsoft.com/nuget/concepts/auditing-packages>
- Sigstore — <https://docs.sigstore.dev/>

---

## 13. Networking — HTTP/2 + H3, forwarded headers, mTLS via mesh

- **HTTP/2** for gRPC and intra-cluster. Kestrel defaults are fine.
- **HTTP/3 (QUIC)** at the public edge (Front Door, AKS Application Gateway for Containers). Inside the cluster it rarely buys anything.
- Behind a reverse proxy/ingress/Front Door, **always**:

  ```csharp
  app.UseForwardedHeaders(new ForwardedHeadersOptions
  {
      ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto,
      KnownNetworks = { /* add your ingress CIDR */ },
      KnownProxies  = { /* add your ingress IPs */ }
  });
  ```

  Without it, `HttpContext.Connection.RemoteIpAddress` is the ingress pod, `Request.Scheme` is `http`, and OIDC redirect URIs come out wrong.
- **mTLS** between services: mesh job. Do **not** hand-roll `SslStream` or custom `HttpClientHandler` for this.
- **NetworkPolicy**: default-deny egress, explicit allow. Otherwise a compromised pod egresses to `169.254.169.254` and steals IMDS tokens.

**Sources:**

- ASP.NET Core forwarded headers — <https://learn.microsoft.com/aspnet/core/host-and-deploy/proxy-load-balancer>
- Kubernetes NetworkPolicy — <https://kubernetes.io/docs/concepts/services-networking/network-policies/>

---

## 14. Multi-tenancy at runtime

See `https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/aks-shared-infra.md` for the tenant allow-list + `IssuerValidator` pattern enforced at token validation. Below is *runtime-layer* guidance:

**Isolation strategies** (pick per data-class, not per product):

| Strategy | Blast radius | Cost | Use when |
|---|---|---|---|
| DB-per-tenant | smallest | highest | regulated data, big-customer SLAs, noisy neighbors unacceptable |
| Schema-per-tenant | small | medium | moderate regulatory pressure, ~100s of tenants |
| Row-level (`TenantId` column + filter) | largest | lowest | SaaS with 1000s+ tenants, uniform schema |

**Per-tenant connection routing:**

```csharp
public sealed class TenantDbContextFactory(ITenantContext tenant, IConnectionStringResolver resolver)
{
    public AppDbContext Create() =>
        new(new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(resolver.Resolve(tenant.TenantId))
            .Options);
}
```

- **Cache keys** must be tenant-prefixed: `$"t:{tenantId}:catalog:{sku}"`. Forget this once and you leak data across tenants.
- **OTel resource attribute** `tenant.id` on every span/log — critical for triage. Never log tenant PII, only the tenant GUID.
- **Rate limits** per tenant, not per IP. Use `RateLimiter` with a partition key on `tenant.id`.
- **Background workers** must carry tenant context explicitly — there's no `HttpContext`. Pass it on the message envelope.

**Sources:**

- ASP.NET Core rate limiting — <https://learn.microsoft.com/aspnet/core/performance/rate-limit>
- Multi-tenant SaaS on Azure — <https://learn.microsoft.com/azure/architecture/guide/multitenant/overview>

---

## 15. Cost & efficiency

- **Right-size from real data**, not guesses. Use VPA in recommend-mode for 2 weeks, then set requests to p95.
- **ARM64** nodepools for stateless services — typically 15–25% cheaper at equal perf on AKS Dpsv5/Epsv5.
- **NativeAOT** (`PublishAot=true`) for short-lived workers, cron jobs, CLI tools, serverless: ~10× faster cold start, ~3× less memory, smaller image. **Not** for big ASP.NET apps with heavy reflection (EF Core, some AutoMapper setups still painful).
- **Spot / Preemptible nodepools** for: batch jobs, log shippers, async workers with retry. **Not** for user-facing APIs or stateful sets.
- **Premium SKUs** (Premium SSD v2, Premium Service Bus, Cosmos autoscale) only where p99/durability actually requires them. Audit quarterly — teams over-provision and never roll back.
- **Dashboards on spend.** Every team owns a "cost per 1k requests" metric. If nobody owns it, it grows.

---

## TL;DR — the opinionated baseline

```
ServiceDefaults      → AddServiceDefaults() in every service (dev /health + /alive only)
K8s probes           → MapHealthChecks for live/ready/startup explicitly
Container            → dotnet publish /t:PublishContainer, noble-chiseled, non-root
Secrets              → CSI + Key Vault + Workload Identity, never K8s Secret
Observability        → OpenTelemetry + OTLP exporter, env-var-driven activation, filter /health/*
Resilience           → AddStandardResilienceHandler on every HttpClient
Service discovery    → https+http://service / https://_endpoint.service, pick resolver per platform
QoS                  → Guaranteed (cpu+mem req == lim) for latency-critical, Burstable on purpose
Shutdown             → HTTP preStop drain (chiseled-safe), HostOptions.ShutdownTimeout < terminationGracePeriodSeconds
Data Protection      → Blob + Key Vault, ApplicationName scoped to app+env
CI/CD                → OIDC + SBOM + cosign + promote-by-digest
Auth                 → EntraAuth.Auth (see https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/aks-shared-infra.md), not a mesh
```

If your service diverges from this without a written reason in the repo, it's technical debt.

---

## Sources (consolidated)

### Authoritative

- **.NET Aspire docs** — <https://learn.microsoft.com/dotnet/aspire/> and <https://aspire.dev/>
- **Aspire 9 → 13 What's New** — <https://learn.microsoft.com/dotnet/aspire/whats-new/dotnet-aspire-9>
- **Aspire integrations (hosting vs client)** — <https://aspire.dev/integrations/overview/> and <https://learn.microsoft.com/dotnet/aspire/fundamentals/integrations-overview>
- **Aspire ServiceDefaults** — <https://aspire.dev/get-started/csharp-service-defaults/>
- **Aspire service discovery (incl. K8s DNS SRV)** — <https://learn.microsoft.com/dotnet/aspire/service-discovery/overview> and <https://aspire.dev/fundamentals/service-discovery/>
- **`Microsoft.Extensions.ServiceDiscovery`** — <https://learn.microsoft.com/dotnet/core/extensions/service-discovery>
- **dotnet/aspire** repo — <https://github.com/dotnet/aspire>
- **.NET container images** — <https://github.com/dotnet/dotnet-docker> and <https://learn.microsoft.com/dotnet/core/docker/container-images>
- **Chiseled Ubuntu containers for .NET** — <https://github.com/dotnet/dotnet-docker/blob/main/documentation/ubuntu-chiseled.md>
- **.NET SDK container build (`PublishContainer`)** — <https://learn.microsoft.com/dotnet/core/docker/publish-as-container>
- **.NET on AKS guidance** — <https://learn.microsoft.com/azure/aks/> and <https://learn.microsoft.com/dotnet/architecture/cloud-native/>
- **Kubernetes probes (concept)** — <https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/>
- **Kubernetes Pod QoS** — <https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/>
- **Kubernetes resource management (CPU shares & CFS quota)** — <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/>
- **Kubernetes lifecycle hooks** — <https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/>
- **OpenTelemetry .NET SDK** — <https://opentelemetry.io/docs/languages/dotnet/>
- **`opentelemetry-dotnet` releases (1.15.x)** — <https://github.com/open-telemetry/opentelemetry-dotnet/releases/latest>
- **`OpenTelemetry.Instrumentation.AspNetCore`** — <https://github.com/open-telemetry/opentelemetry-dotnet-contrib/blob/main/src/OpenTelemetry.Instrumentation.AspNetCore/README.md>
- **OTel exporter env vars** — <https://opentelemetry.io/docs/specs/otel/protocol/exporter/>
- **OTel semantic conventions** — <https://opentelemetry.io/docs/specs/semconv/>
- **HTTP resilience (`AddStandardResilienceHandler`)** — <https://learn.microsoft.com/dotnet/core/resilience/http-resilience> and <https://www.pollydocs.org/>
- **Dapr state-management outbox** — <https://docs.dapr.io/developing-applications/building-blocks/state-management/howto-outbox/>
- **Health checks API (`HealthCheckOptions`)** — <https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.diagnostics.healthchecks.healthcheckoptions>
- **`IHealthCheckPublisher`** — <https://learn.microsoft.com/dotnet/api/microsoft.extensions.diagnostics.healthchecks.ihealthcheckpublisher>
- **Hosted services / `HostOptions.ShutdownTimeout`** — <https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services>
- **Data Protection in K8s (config + Azure Blob/Key Vault providers)** — <https://learn.microsoft.com/aspnet/core/security/data-protection/configuration/overview> and <https://learn.microsoft.com/aspnet/core/security/data-protection/implementation/key-storage-providers>
- **Azure Workload Identity** — <https://learn.microsoft.com/azure/aks/workload-identity-overview>
- **Secrets Store CSI Driver + Key Vault** — <https://learn.microsoft.com/azure/aks/csi-secrets-store-driver>
- **KEDA** — <https://keda.sh/docs/>
- **GitHub Actions OIDC → Azure** — <https://learn.microsoft.com/azure/developer/github/connect-from-azure-openid-connect>
- **Microsoft.Sbom.Targets** — <https://github.com/microsoft/sbom-tool>
- **Sigstore cosign / Notation** — <https://docs.sigstore.dev/> / <https://notaryproject.dev/>
- **`dotnet nuget audit`** — <https://learn.microsoft.com/nuget/concepts/auditing-packages>

### Community (talks + blogs)

- **David Fowler** — Aspire design talks, `dotnet/aspire` issues & discussions; distributed app model rationale.
- **Damian Edwards** — .NET Conf / NDC Aspire keynotes and ServiceDefaults walkthroughs.
- **Steve Gordon** — <https://www.stevejgordon.co.uk/> — HttpClientFactory internals, OpenTelemetry .NET, perf.
- **Andrew Lock** — <https://andrewlock.net/> — ASP.NET Core configuration, Data Protection in K8s, hosting lifetimes.
- **Khalid Abuhakmeh** — <https://khalidabuhakmeh.com/> — Aspire integrations, container tips.
- **Maarten Balliauw** — <https://blog.maartenballiauw.be/> — NuGet, Azure, .NET infra.
- **Tom Kerkhove** — <https://www.tomkerkhove.be/> — KEDA, Azure messaging, workload identity.
- **Mark Heath** — <https://markheath.net/> — .NET on Azure patterns, practical cloud-native samples.
