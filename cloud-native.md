# Cloud-Native .NET 10 on Kubernetes — Opinionated Best Practices

*Audience: staff/senior engineers shipping .NET 10 services to AKS (or any CNCF-conformant cluster). Companion to [`../aks-shared-infra.md`](../aks-shared-infra.md). Assumes Aspire 13 GA (Nov 2025) and .NET 10 GA (Nov 2025).*

> **One-line position.** Use **Aspire for the inner loop + ServiceDefaults**, the **.NET SDK container build** for images, **OpenTelemetry + standard resilience pipelines** everywhere, and **Workload Identity + CSI Key Vault** for secrets. Everything else is either the cluster's job or the platform team's job — not your service's.

---

## 1. .NET Aspire — what it is, what it isn't

**What it is (Aspire 13 GA, Nov 2025):**

- An **inner-loop orchestration + opinionated-defaults** toolkit. `AppHost` is a C# program that models your distributed app (services, Postgres, Redis, RabbitMQ, Azure Storage, …) so `aspire run` / F5 spins up the whole graph with a dashboard, OTel wired up, and connection strings injected.
- **ServiceDefaults** is a shared project you `AddServiceDefaults()` into every service: OpenTelemetry (traces/metrics/logs via OTLP), health checks (`/health`, `/alive`), standard resilience on `HttpClient`, service discovery.
- **Integrations** (`Aspire.Hosting.*` / `Aspire.*.Client`) give you typed, tested wiring for Postgres, Redis, RabbitMQ, Kafka, Azure Service Bus, Cosmos, Blob, Key Vault, OpenAI, etc. — with health checks and OTel pre-registered.
- **Dashboard** (also usable standalone via OTLP) is the best local trace/log/metric UI in the ecosystem.
- As of Aspire 13, the product is rebranded from ".NET Aspire" to "Aspire" and supports **polyglot app models** (Python, JS) in the same AppHost; the CLI is GA and AOT-compiled.

**What it is *not*:**

- **Not a deployment runtime.** `AppHost` does not run in production. `aspire publish` / `azd` / Bicep / Helm generate manifests; the cluster runs the service, not AppHost.
- **Not a PaaS.** It doesn't replace AKS, ACA, or App Service.
- **Not a service mesh.** No mTLS, no L7 policy, no traffic shifting.
- **Not required.** A well-factored `Program.cs` with the same OTel/health/resilience setup is equivalent — Aspire just removes the boilerplate and the "every team reinvents wiring" problem.

**Do / Don't:**

- ✅ Adopt Aspire when you have ≥2 services, ≥1 dependency (Postgres/Redis/bus), and devs currently `docker-compose up` by hand.
- ✅ Put `ServiceDefaults` in every new service, even solo ones — you get OTel + health + resilience for free.
- ✅ Use Aspire integrations over hand-rolled `IHostBuilder` wiring; they set the OTel `ActivitySource`/meter names correctly.
- ❌ Don't ship `Aspire.Hosting.*` to prod. AppHost references are `<IsAspireHost>` and build-time only.
- ❌ Don't let Aspire's generated Bicep/Helm be the *final* deployment artifact for a regulated workload — it's a starting point; platform teams own the real manifests.
- ❌ Don't fight Aspire's defaults (resource naming, port allocation) in dev — you'll lose a week.

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

What you get by default on .NET 10:

- Base image: `mcr.microsoft.com/dotnet/aspnet:10.0` family. Pick `ContainerFamily=noble-chiseled` (Ubuntu 24.04, distroless-style) or `noble-chiseled-extra` if you need `icu`/`tzdata`.
- **Non-root by default** (`UID 1654 / app`). Don't undo this.
- Reproducible layers, SBOM-friendly, deterministic tags.

**Base image trade-offs (.NET 10):**

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
- ✅ NativeAOT (`PublishAot=true`) for short-lived workers, CLI tools, functions; base image is `runtime-deps:10.0-noble-chiseled`.
- ❌ Don't run as root. Don't `chmod 777`. Don't `apt-get install curl` "for health checks" — use an HTTP probe.
- ❌ Don't bake secrets, connection strings, or `appsettings.Production.json` into the image.
- ❌ Don't use `:latest`. Ever. Tag with immutable SHAs; promote by digest.

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
- **Readiness** = "can I serve traffic right now?" — checks DB/Redis/bus. Removes from Service endpoints, does *not* restart.

**Rule:** a failing downstream dependency must **never** trip liveness. Otherwise Postgres hiccups cascade into pod restart storms.

### Requests/limits + .NET GC

.NET 10's GC reads cgroup limits. Set **both** requests and limits, and make them equal for latency-sensitive workloads:

```yaml
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits:   { cpu: "1",    memory: "512Mi" }
```

- `DOTNET_GCHeapHardLimit` / `DOTNET_GCConserveMemory=5` for memory-tight workers.
- `DOTNET_TieredPGO=1` is default in .NET 10 — leave on.
- `DOTNET_gcServer=1` is default on multi-core; don't override.
- **Never** set memory limit without request — scheduler will misplace you.
- Avoid CPU throttling: if p99 matters, use `cpu.requests ≈ cpu.limits` or drop the limit and rely on PDBs/priority classes.

### Graceful shutdown

```yaml
terminationGracePeriodSeconds: 45
lifecycle:
  preStop: { exec: { command: ["/bin/sleep", "5"] } }  # let ingress depropagate
```

```csharp
builder.Services.Configure<HostOptions>(o =>
{
    o.ShutdownTimeout = TimeSpan.FromSeconds(30); // < terminationGracePeriodSeconds
    o.BackgroundServiceExceptionBehavior = BackgroundServiceExceptionBehavior.StopHost;
});
```

- Implement `IHostApplicationLifetime.ApplicationStopping` in workers: stop pulling from the queue, drain in-flight, then return.
- ASP.NET Core drains HTTP automatically; the only reason requests get killed is `ShutdownTimeout` < actual drain time.

### HPA / KEDA / PDB

- **HPA on CPU is a last resort.** CPU is a lagging indicator. Prefer:
  - **KEDA** on queue depth (Service Bus, RabbitMQ, Kafka lag) for workers.
  - HPA on **custom OTel metrics** (RPS, p95 latency) via Prometheus adapter for APIs.
- **PDB always.** `minAvailable: 1` (or `maxUnavailable: 1` for HA workloads) — otherwise node drains take everyone.
- Scale-to-zero only for truly bursty non-interactive workloads. Cold-start on JIT .NET is 2–5s; NativeAOT is ~100ms.

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

---

## 5. Observability — OpenTelemetry, one SDK, three signals

Standard wiring (in `ServiceDefaults`):

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(builder.Environment.ApplicationName))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("EntraAuth.*")
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddMeter("EntraAuth.*")
        .AddOtlpExporter());

builder.Logging.AddOpenTelemetry(o =>
{
    o.IncludeFormattedMessage = true;
    o.IncludeScopes = true;
    o.AddOtlpExporter();
});
```

- **Exporter: OTLP only.** No vendor SDKs in code. Rewire the collector, not the app.
- **Dev**: OTLP → Aspire dashboard (`OTEL_EXPORTER_OTLP_ENDPOINT` auto-injected by AppHost).
- **Prod**: OTLP → OpenTelemetry Collector → Grafana Tempo/Loki/Mimir **or** Azure Monitor OTLP. Don't put vendor SDKs on the hot path.
- **Semantic conventions**: use stable names (`http.request.method`, `db.system.name`, `messaging.system`). Don't invent `my.http.verb`.
- **ActivitySource naming**: `{Org}.{Product}.{Component}` — e.g., `EntraAuth.Catalog.Domain`. Stable. Filterable in sampling rules.
- **Log correlation**: logs auto-include `TraceId`/`SpanId` when OTel logging provider is registered. Turn off Serilog's parallel trace injection if both are wired — pick one.

**Sampling:** tail-based in the collector, not head-based in the app. Sample errors and slow traces at 100%, baseline at 5–10%.

---

## 6. Resilience — Polly v8 standard pipelines, not hand-rolled retries

.NET 10 uses `Microsoft.Extensions.Resilience` (Polly v8 under the hood). The standard HTTP pipeline gives you: total-timeout → retry (with jitter) → circuit breaker → attempt-timeout.

```csharp
builder.Services.AddHttpClient<CatalogClient>()
    .AddStandardResilienceHandler(o =>
    {
        o.Retry.MaxRetryAttempts = 3;
        o.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
        o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(2);
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(10);
    });
```

**Do:**

- ✅ `AddStandardResilienceHandler` on every outbound `HttpClient`. Make it the default.
- ✅ **Idempotency keys** on POSTs that mutate money/state. `Idempotency-Key` header, dedup store with TTL in Redis.
- ✅ **Outbox pattern** for "save row + publish event" — never two-phase commit. Transactional outbox table, a relay pushes to the bus.
- ✅ **Chaos**: add `AddChaosLatency` / `AddChaosFault` strategies behind a feature flag in non-prod to test resilience paths.

**Don't:**

- ❌ Retry on non-idempotent verbs without an idempotency key. You'll double-charge customers.
- ❌ Stack retries at multiple layers (client + gateway + mesh). Pick one.
- ❌ Circuit-break on **every** exception type. Exclude `OperationCanceledException` from request cancellation.

---

## 7. Service-to-service

**Auth:** see [`../aks-shared-infra.md`](../aks-shared-infra.md) for the tenant-allow-list + `azp` client-app pattern. The short version: validate JWTs in the service with the shared `EntraAuth.Auth` library; do **not** use a mesh for authZ.

**Transport:**

- **HTTP/JSON** by default. Boring, debuggable, works with every tool.
- **gRPC** when: internal, schema-stable, hot path, streaming. Not for public APIs (browser story still painful).
- **HTTP/2 keep-alive** and connection pooling matter more than protocol choice for p99.

**Service mesh (Istio / Linkerd / AKS Istio addon):** only worth it for:

- Uniform **mTLS** between services (if you can't terminate at the edge + trust the CNI).
- Traffic shifting, canary, retry policies **you can't get from the standard resilience pipeline**.
- L7 telemetry when you have non-.NET services you can't instrument.

**Not** for auth. AuthN/Z stays in the service. A mesh is plumbing, not policy.

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

---

## 9. Data Protection — shared key ring, always

ASP.NET Core Data Protection encrypts antiforgery tokens, cookies, OAuth state, `IDataProtector` payloads. **Keys are per-replica unless you persist them.** On rollout, half your users get 500s until old pods die.

**Correct setup for K8s:**

```csharp
builder.Services
    .AddDataProtection()
    .SetApplicationName("catalog-api")                      // same across replicas + environments
    .PersistKeysToAzureBlobStorage(blobClient, "keys.xml")  // shared storage
    .ProtectKeysWithAzureKeyVault(kvKeyId, credential);     // envelope-encrypted at rest
```

- `ApplicationName` must match across replicas, or decryption fails.
- Blob + Key Vault is the canonical pair. Alternatives: Redis (ephemeral-ish, avoid for production key material), cluster PV (works but operationally painful).
- Rotate Key Vault keys on a schedule; Data Protection handles ring rollover automatically.

**Symptom of getting this wrong:** intermittent `CryptographicException: The key {guid} was not found in the key ring` after deploy. Usually in antiforgery / auth cookie / OIDC state.

---

## 10. Health checks — three endpoints, not one

```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(cs,   name: "postgres", tags: ["ready"])
    .AddRedis(cs,    name: "redis",    tags: ["ready"])
    .AddAzureServiceBusQueue(..., tags: ["ready"])
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"]);

app.MapHealthChecks("/health/live",    new() { Predicate = r => r.Tags.Contains("live")   });
app.MapHealthChecks("/health/ready",   new() { Predicate = r => r.Tags.Contains("ready")  });
app.MapHealthChecks("/health/startup", new() { Predicate = r => r.Tags.Contains("ready")  });
```

- Liveness has **no** external checks. Only "am I alive in-process?"
- Readiness checks critical dependencies. Non-critical (telemetry, feature flags) → degraded, not unhealthy.
- Startup = readiness with a longer grace window. Same checks, different probe.
- **Cache** expensive health checks (`CacheDuration` on the result) — otherwise a 5s probe storm hammers Postgres.
- Expose health on a **separate non-public port** or gate with a network policy. Don't leak dependency topology via `/health`.

---

## 11. Graceful shutdown — drain, don't drop

Checklist per service:

- [ ] `HostOptions.ShutdownTimeout` set (default 30s is fine if `terminationGracePeriodSeconds` ≥ 45s).
- [ ] `preStop: sleep 5` so ingress/endpoint-slice propagation finishes before the process exits.
- [ ] Background workers: `ExecuteAsync(CancellationToken stoppingToken)` respects the token, drains in-flight unit of work, commits offsets.
- [ ] Message-bus consumers: stop `ReceiveMessagesAsync`, finish in-flight handlers, `CompleteMessageAsync`, then close.
- [ ] gRPC servers: `Kestrel` handles `GOAWAY` on SIGTERM automatically; don't fight it.
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

---

## 12. CI/CD — reproducible, signed, scanned

**Must-haves:**

- ✅ **GitHub Actions OIDC + FIC** for Azure deploy. No `AZURE_CLIENT_SECRET` in secrets.
- ✅ **SBOM** via `Microsoft.Sbom.Targets` (`GenerateSBOM=true`). Attach to release.
- ✅ **Container signing**: Sigstore `cosign` or Notation. Verify in admission controller (Ratify / Kyverno / Gatekeeper).
- ✅ **Dependency scan** in PR: `dotnet list package --vulnerable --include-transitive`, `dotnet nuget audit` (GA in .NET 10), plus GHAS/Dependabot.
- ✅ **Reproducible builds**: `ContinuousIntegrationBuild=true`, `Deterministic=true`, pinned SDK via `global.json`.
- ✅ **Image digests**, not tags, in Helm/Kustomize `values.yaml`. Promote by digest across dev→stage→prod.

**Don't:**

- ❌ Build once per environment. Build once, promote.
- ❌ Use long-lived service principals. FIC is free and better.
- ❌ Skip `dotnet nuget audit` because "it's noisy." Set a severity threshold, triage weekly.

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

---

## 14. Multi-tenancy at runtime

See `../aks-shared-infra.md` for the tenant allow-list + `IssuerValidator` pattern enforced at token validation. Below is *runtime-layer* guidance:

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
ServiceDefaults      → AddServiceDefaults() in every service
Container            → dotnet publish /t:PublishContainer, noble-chiseled, non-root
Probes               → startup + live (self) + ready (deps)
Secrets              → CSI + Key Vault + Workload Identity, never K8s Secret
Observability        → OpenTelemetry + OTLP, vendor-neutral
Resilience           → AddStandardResilienceHandler on every HttpClient
Data Protection      → Blob + Key Vault, shared ApplicationName
Shutdown             → HostOptions.ShutdownTimeout < terminationGracePeriodSeconds
CI/CD                → OIDC + SBOM + cosign + promote-by-digest
Auth                 → EntraAuth.Auth (see ../aks-shared-infra.md), not a mesh
```

If your service diverges from this without a written reason in the repo, it's technical debt.

---

## Sources

### Authoritative

- **.NET Aspire / Aspire docs** — <https://learn.microsoft.com/dotnet/aspire/> (Aspire 13 GA, Nov 2025)
- **dotnet/aspire** repo — <https://github.com/dotnet/aspire>
- **.NET 10 container images** — <https://github.com/dotnet/dotnet-docker> and <https://learn.microsoft.com/dotnet/core/docker/container-images>
- **Chiseled Ubuntu containers for .NET** — <https://github.com/dotnet/dotnet-docker/blob/main/documentation/ubuntu-chiseled.md>
- **.NET SDK container build (`PublishContainer`)** — <https://learn.microsoft.com/dotnet/core/docker/publish-as-container>
- **.NET on AKS guidance** — <https://learn.microsoft.com/azure/aks/> and <https://learn.microsoft.com/dotnet/architecture/cloud-native/>
- **Kubernetes probes** — <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>
- **OpenTelemetry .NET SDK** — <https://opentelemetry.io/docs/languages/net/> and <https://github.com/open-telemetry/opentelemetry-dotnet>
- **OTel semantic conventions** — <https://opentelemetry.io/docs/specs/semconv/>
- **Microsoft.Extensions.Resilience / Polly v8** — <https://learn.microsoft.com/dotnet/core/resilience/> and <https://github.com/App-vNext/Polly>
- **Data Protection in K8s** — <https://learn.microsoft.com/aspnet/core/security/data-protection/configuration/overview>
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
