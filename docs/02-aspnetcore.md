# ASP.NET Core 10 — Server-Side Best Practices

Opinionated defaults for APIs and background services on **.NET 10 (LTS, Nov 2025)**. Terse on purpose. If something isn't here, assume the framework default is fine. Companion docs: [`validation.md`](https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/validation.md), [`best-practices.md`](https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/best-practices.md).

---

## 1. Minimal APIs vs Controllers

**Default: Minimal APIs.** They are no longer "the new thing" — in .NET 10 they match controllers on parameter binding, filters, validation, OpenAPI, and auth. Reach for MVC controllers only when you genuinely need: model binding providers, action filters with DI-heavy ordering, `[ApiController]` conventions across dozens of endpoints, Razor/Views, or OData.

**Org-level rule (service-level, not per-feature):**
- Greenfield HTTP-only services use Minimal APIs.
- Existing MVC services stay on controllers; don't half-migrate.
- Do not mix Minimal APIs and controllers in one bounded feature without a written exception in the service README.

Do:
- One file per bounded feature; wire endpoints via an extension `public static RouteGroupBuilder MapOrders(this IEndpointRouteBuilder e)`.
- Use `MapGroup("/v1/orders")` and layer `RequireAuthorization`, `WithTags`, `WithOpenApi`, `AddEndpointFilter` on the group, not on each endpoint.
- Return `TypedResults` (not `Results`) — they carry the response type for OpenAPI and make handlers unit-testable without hitting `HttpContext`.
- Return `Results<Ok<Order>, NotFound, ValidationProblem>` union types; the framework infers all status codes for OpenAPI.

Don't:
- Use `IResult` as the return type on a public endpoint — you lose the OpenAPI contract.
- Nest `MapGroup` more than two levels; it becomes unreadable.
- Put cross-cutting logic in endpoint filters that should be authorization policies or middleware.

```csharp
var orders = app.MapGroup("/v1/orders")
    .RequireAuthorization("OrdersRead")
    .WithTags("Orders")
    .AddEndpointFilter<RequestLoggingFilter>();

orders.MapGet("/{id:guid}", async Task<Results<Ok<Order>, NotFound>>
    (Guid id, IOrderService svc, CancellationToken ct) =>
{
    var o = await svc.FindAsync(id, ct);
    return o is null ? TypedResults.NotFound() : TypedResults.Ok(o);
});
```

**Controllers still win** for: complex multipart uploads with per-part binders, large domains already on MVC, and apps using `ApplicationPartManager` for plugin assemblies.

Sources:
- [Minimal APIs overview](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview).
- [Choose between controllers and Minimal APIs](https://learn.microsoft.com/aspnet/core/fundamentals/apis).
- [`TypedResults` vs `Results`](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/responses).

---

## 2. Model Binding & Validation

Binding sources — be explicit, don't rely on inference for anything past simple types:
- `[FromRoute]` for IDs, `[FromQuery]` for filters/paging, `[FromBody]` for the payload, `[FromHeader]` for tenancy/correlation, `[FromServices]` for DI dependencies.
- `[AsParameters]` to bundle query+route+header into a single DTO — keeps signatures sane and OpenAPI clean.

Validation — the .NET 10 story changed. Pick one lane:

**(a) Source-generated validation via `Microsoft.Extensions.Validation` (NEW in .NET 10).** The recommended default for Minimal APIs.

```csharp
builder.Services.AddValidation();        // scans for [Validatable] types
app.MapPost("/orders", (CreateOrder dto) => TypedResults.Ok())
   .WithParameterValidation();           // or register the endpoint filter globally
```

Mark DTOs with `[Validatable]`; DataAnnotations attributes (`[Required]`, `[Range]`, `[StringLength]`, `[EmailAddress]`) are picked up and validated at compile-time-emitted code — no reflection per request. On failure you get a `ValidationProblemDetails` automatically.

**(b) FluentValidation** when rules cross fields, need DI (DB lookups for uniqueness), or vary by HTTP method. Don't mix the two on the same DTO. Register explicitly — `FluentValidation.AspNetCore`'s auto-MVC-integration has been deprecated; invoke validators yourself in an endpoint filter.

**(c) `IValidatableObject`** — only for a single, isolated cross-field rule. Anything larger, escalate to FluentValidation.

Don't:
- Throw from validators. Return `ValidationProblem`.
- Swallow `ModelState` errors; return `TypedResults.ValidationProblem(errors)` which emits RFC 9457 `application/problem+json`.

**Keyed services in handlers.** When a feature uses several implementations of the same interface (e.g. `IPaymentGateway` for "stripe" and "adyen"), inject by key with `[FromKeyedServices]` rather than a hand-rolled factory:

```csharp
builder.Services.AddKeyedScoped<IPaymentGateway, StripeGateway>("stripe");
builder.Services.AddKeyedScoped<IPaymentGateway, AdyenGateway>("adyen");

// Minimal API
orders.MapPost("/{id:guid}/pay/stripe",
    ([FromKeyedServices("stripe")] IPaymentGateway gw, Guid id) => gw.ChargeAsync(id));

// Controller
public sealed class PaymentsController(
    [FromKeyedServices("adyen")] IPaymentGateway adyen) : ControllerBase { … }
```

Use keyed services when the key is a stable, small set known at composition time. Reach for a factory or strategy registry when keys are dynamic (tenant-driven, config-driven), or when you need to enumerate all implementations.

Sources:
- [Validation in Minimal APIs (.NET 10)](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/min-api-validation).
- [Model binding](https://learn.microsoft.com/aspnet/core/mvc/models/model-binding) and [`[AsParameters]`](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/parameter-binding#parameter-binding-for-argument-lists-with-asparameters).
- [Keyed DI services](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#keyed-services) — `[FromKeyedServices]`.

---

## 3. ProblemDetails & Error Handling

Rule: **every app-generated 4xx/5xx is `application/problem+json`** (RFC 9457, which obsoletes RFC 7807). The middleware below covers exception-driven and status-code-driven responses; auth challenges (401/403 from the JWT bearer handler) need explicit handling — see "Auth challenges" below — because the bearer handler emits its own `WWW-Authenticate` response that you must preserve.

```csharp
builder.Services.AddProblemDetails(o =>
{
    o.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Extensions["requestId"] = Activity.Current?.Id;
    };
});
builder.Services.AddExceptionHandler<DomainExceptionHandler>();
builder.Services.AddExceptionHandler<UnhandledExceptionHandler>();  // last, catch-all

var app = builder.Build();
app.UseExceptionHandler();
app.UseStatusCodePages();
```

Implement `IExceptionHandler` (.NET 8+) — one per exception family. Return `true` when handled, `false` to fall through to the next handler. This replaces the old delegate-based `UseExceptionHandler("/error")` pattern.

Do:
- Map domain exceptions to RFC 9457 ProblemDetails (`type` = stable URI, `title` = human summary, `status`, `detail`, `instance` = request path, custom extensions for domain codes).
- Return 409 for concurrency, 422 for semantic validation, 400 for schema validation, 404 for missing, 403 for forbidden (authenticated), 401 only when auth failed.
- Include `traceId`/`Activity.Id` so support can grep logs.

Don't:
- Leak stack traces, connection strings, or SQL in `detail`. The developer exception page is **development only** — guard with `app.Environment.IsDevelopment()`.
- Catch `Exception` in handlers and hide it. Let the unhandled handler log + 500.
- Return plain strings for errors — breaks every client that uses `application/problem+json`.

**Auth challenges (401/403).** `AddProblemDetails` + `UseStatusCodePages` does **not** automatically convert bearer-handler challenges into ProblemDetails, and naively replacing the response body will drop the `WWW-Authenticate` header — which clients (and Microsoft.Identity.Web's CAE / claims-challenge flow) need. Wire the JWT bearer events to write ProblemDetails *while preserving* the challenge header:

```csharp
builder.Services.Configure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, o =>
{
    o.Events ??= new JwtBearerEvents();
    o.Events.OnChallenge = async ctx =>
    {
        ctx.HandleResponse();
        // Let the handler set WWW-Authenticate first (incl. CAE "insufficient_claims").
        ctx.Response.Headers.Append(HeaderNames.WWWAuthenticate,
            $"Bearer error=\"{ctx.Error ?? "invalid_token"}\", error_description=\"{ctx.ErrorDescription}\"");
        ctx.Response.StatusCode = StatusCodes.Status401Unauthorized;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Type = "https://datatracker.ietf.org/doc/html/rfc6750#section-3",
            Title = "Unauthorized",
            Status = 401,
            Detail = ctx.ErrorDescription
        }, contentType: "application/problem+json");
    };
    o.Events.OnForbidden = async ctx =>
    {
        ctx.Response.StatusCode = StatusCodes.Status403Forbidden;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Type = "https://datatracker.ietf.org/doc/html/rfc9457",
            Title = "Forbidden",
            Status = 403
        }, contentType: "application/problem+json");
    };
});
```

For step-up / CAE flows, use Microsoft.Identity.Web's `ReplyForbiddenWithWwwAuthenticateHeaderAsync` so the `claims=` parameter is included; bare `Forbid()` will not carry it.

Sources:
- [RFC 9457 — Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc9457) (obsoletes RFC 7807).
- [Handle errors — `IExceptionHandler` and `AddProblemDetails`](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling).
- [JWT bearer events](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.authentication.jwtbearer.jwtbearerevents) — preserving `WWW-Authenticate`.
- [Microsoft.Identity.Web claims challenges / CAE](https://learn.microsoft.com/entra/identity-platform/claims-challenge).

---

## 4. API Versioning

Use **`Asp.Versioning.Http`** (+ `Asp.Versioning.Mvc.ApiExplorer` for OpenAPI). The package is the community-owned successor to `Microsoft.AspNetCore.Mvc.Versioning`.

```csharp
builder.Services.AddApiVersioning(o =>
{
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.AssumeDefaultVersionWhenUnspecified = false;  // force clients to ask
    o.ReportApiVersions = true;                     // api-supported-versions header
    o.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("api-version"));
}).AddApiExplorer(o => { o.GroupNameFormat = "'v'VVV"; o.SubstituteApiVersionInUrl = true; });

var v1 = app.MapGroup("/api/v{version:apiVersion}").HasApiVersion(1.0);
var v2 = app.MapGroup("/api/v{version:apiVersion}").HasApiVersion(2.0);
```

Opinions:
- **URL segment versioning** (`/v1/...`) is the least surprising. Header/media-type are clever but break caches, browsers, and CDN rules.
- Mark deprecated versions via `.HasDeprecatedApiVersion(1.0)`. This sets `api-deprecated-versions` automatically (when `ReportApiVersions = true`), but **`Sunset` headers do not flow automatically** — register an explicit sunset policy:

```csharp
builder.Services.AddApiVersioning(o => { /* … */ })
    .AddApiExplorer(/* … */);

builder.Services.AddSunsetPolicyBuilder()
    .Map(new ApiVersion(1, 0))
        .Effective(DateTimeOffset.Parse("2026-06-30T00:00:00Z"))
        .Link("https://docs.example.com/api/v1-sunset").Title("Migration guide");
```

- Don't version on minor patches; only breaking contract changes justify a new major.

Sources:
- [Asp.Versioning wiki — versioning policies](https://github.com/dotnet/aspnet-api-versioning/wiki).
- [Asp.Versioning sunset policies](https://github.com/dotnet/aspnet-api-versioning/wiki/Sunset-Policy).
- [RFC 8594 — the Sunset HTTP header](https://datatracker.ietf.org/doc/html/rfc8594).

---

## 5. OpenAPI

**`Microsoft.AspNetCore.OpenApi`** is built-in (.NET 9+) and is the default in .NET 10; **Swashbuckle** is no longer in the `webapi` template. Swashbuckle still works but is community-maintained and lags OpenAPI 3.1 support.

```csharp
builder.Services.AddOpenApi("v1", o =>
{
    o.AddDocumentTransformer<SecuritySchemeTransformer>();
    o.AddOperationTransformer<AddCorrelationIdHeaderTransformer>();
});

app.MapOpenApi();         // /openapi/v1.json  (also YAML in .NET 10)
```

- .NET 10 emits **OpenAPI 3.1** by default (JSON Schema 2020-12), YAML supported.
- Customize via `IOpenApiDocumentTransformer` / `IOpenApiOperationTransformer` / `IOpenApiSchemaTransformer`. No more `ISchemaFilter` gymnastics.
- For the UI, pick **[Scalar](https://scalar.com/)** (`Scalar.AspNetCore`) — it's what the Microsoft samples now show. Use Swagger UI only if you have established tooling around it.
- Don't commit generated `.json`/`.yaml` to the repo unless it's the contract (contract-first). Let CI publish it.

**Build-time generation.** For contract publishing, SDK generation, and CI diffing, generate the document at build time instead of (or in addition to) runtime. Add the package and opt in via the MSBuild property:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.ApiDescription.Server" Version="10.0.*" PrivateAssets="all" />
</ItemGroup>
<PropertyGroup>
  <OpenApiGenerateDocuments>true</OpenApiGenerateDocuments>
  <OpenApiDocumentsDirectory>$(MSBuildProjectDirectory)/openapi</OpenApiDocumentsDirectory>
</PropertyGroup>
```

The build emits `openapi/<assembly>.json` per document. Publish it from CI as a build artifact (and into your API catalog / SDK pipeline) so consumers don't depend on a running instance.

Sources:
- [Microsoft.AspNetCore.OpenApi overview](https://learn.microsoft.com/aspnet/core/fundamentals/openapi/overview).
- [Generate OpenAPI documents at build time](https://learn.microsoft.com/aspnet/core/fundamentals/openapi/aspnetcore-openapi#generate-openapi-documents-at-build-time).
- [`Scalar.AspNetCore`](https://github.com/scalar/scalar/tree/main/integrations/aspnetcore).

---

## 6. Logging & Observability

**Structured logging, always.** A `Log.Information($"...{x}")` is a bug waiting to be filed — it destroys structure and allocates.

```csharp
public static partial class Log
{
    [LoggerMessage(EventId = 1001, Level = LogLevel.Information,
        Message = "Order {OrderId} accepted for customer {CustomerId}")]
    public static partial void OrderAccepted(ILogger logger, Guid orderId, Guid customerId);
}
```

Do:
- `LoggerMessage` source generators (`[LoggerMessage]`) for hot paths. Zero allocation, compile-time format check.
- Template placeholders in PascalCase; they become log property names.
- Emit `ActivitySource` spans for cross-service work; the OTel exporter maps them to traces.
- `System.Diagnostics.Metrics.Meter` for business counters/histograms; don't invent your own metrics abstraction.

Don't:
- `logger.LogInformation($"user {id}")` — string interpolation hides the structure.
- Log PII or tokens. Ever.
- Swallow `Activity.Current` by creating disconnected `Activity` instances.

**OpenTelemetry** is the default telemetry stack. `ActivitySource` and `Meter` names registered with OTel are matched **as exact literal strings** — there is no wildcard or prefix match. Centralize the names as constants and register each one explicitly:

```csharp
public static class Telemetry
{
    public const string ApiSource    = "Orders.Api";
    public const string DomainSource = "Orders.Domain";
    public const string ApiMeter     = "Orders.Api";
    public const string DomainMeter  = "Orders.Domain";
}

builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("orders-api"))
    .WithTracing(t => t.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation()
                       .AddSource(Telemetry.ApiSource, Telemetry.DomainSource)
                       .AddOtlpExporter())
    .WithMetrics(m => m.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation()
                       .AddRuntimeInstrumentation()
                       .AddMeter(Telemetry.ApiMeter, Telemetry.DomainMeter)
                       .AddOtlpExporter());
```

Ship logs via OTel too (`builder.Logging.AddOpenTelemetry(...)`) and you'll get a single wire protocol for everything. **.NET Aspire** wires this up for you in dev — use it.

Sources:
- [OpenTelemetry for .NET](https://learn.microsoft.com/dotnet/core/diagnostics/observability-with-otel).
- [Distributed tracing instrumentation walkthroughs](https://learn.microsoft.com/dotnet/core/diagnostics/distributed-tracing-instrumentation-walkthroughs) — exact source names.
- [`LoggerMessage` source generator](https://learn.microsoft.com/dotnet/core/extensions/logger-message-generator).
- [`System.Diagnostics.Metrics`](https://learn.microsoft.com/dotnet/core/diagnostics/metrics).

---

## 7. Resilience

Use **`Microsoft.Extensions.Http.Resilience`** (Polly v8 under the hood). Do not hand-roll retries.

```csharp
builder.Services.AddHttpClient<IBillingClient, BillingClient>(c =>
        c.BaseAddress = new Uri("https://billing.internal"))
    .AddStandardResilienceHandler(o =>
    {
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);
        o.AttemptTimeout.Timeout      = TimeSpan.FromSeconds(10);
        o.Retry.MaxRetryAttempts      = 3;
        o.CircuitBreaker.FailureRatio = 0.5;
    });
```

`AddStandardResilienceHandler` = rate limiter → total timeout → retry → circuit breaker → attempt timeout. Sane default for idempotent calls.

Rules of thumb:
- **`AddStandardResilienceHandler` is the default** for outbound HTTP. Use it everywhere unless you need a strategy it doesn't expose. This is the same default `06-cloud-native.md` calls out — pick this chapter's wiring as the source of truth.
- **Reach for a custom Polly v8 pipeline** (via `AddResilienceHandler("name", b => b.AddRetry(...).AddTimeout(...))`) only when you need non-standard ordering, custom predicates (e.g., retry on a domain error code), or strategies the standard handler doesn't expose. Don't fork the defaults to "tune one knob" — pass options to `AddStandardResilienceHandler` instead.
- **Hedging** (`AddStandardHedgingHandler`) is for read-heavy fan-out across replicas/regions; cuts tail latency. Don't hedge writes or non-idempotent calls.
- **Retry only idempotent requests** (GET, PUT, DELETE; POST only if you have idempotency keys). The standard handler retries safe verbs by default — verify on each client.
- Always set a **total timeout** (end-to-end budget) *and* a per-attempt timeout. Without the total, retries stack unboundedly.
- Tune **circuit breaker** on failure ratio + sampling duration, not raw counts. Default (50% of 100 samples over 30s, 30s break) is fine for most.
- **Fallback** is a last resort — an empty list or cached snapshot. Don't fallback to silently ignoring a 500.

Sources:
- [`Microsoft.Extensions.Http.Resilience`](https://learn.microsoft.com/dotnet/core/resilience/http-resilience) — standard handler, hedging, options.
- [Polly v8 strategies](https://www.pollydocs.org/strategies/index.html).
- [Resilient apps with `Microsoft.Extensions.Resilience`](https://learn.microsoft.com/dotnet/core/resilience/).

---

## 8. Rate Limiting

Built into ASP.NET Core (`Microsoft.AspNetCore.RateLimiting`). Use it; don't rely on a reverse proxy alone (can't reason about auth'd identities).

```csharp
builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.AddPolicy("per-user", ctx =>
        RateLimitPartition.GetTokenBucketLimiter(
            ctx.User.FindFirst("oid")?.Value ?? ctx.Connection.RemoteIpAddress!.ToString(),
            _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 100, TokensPerPeriod = 100,
                ReplenishmentPeriod = TimeSpan.FromMinutes(1),
                QueueLimit = 0
            }));
});

app.UseRateLimiter();
ordersGroup.RequireRateLimiting("per-user");
```

- **Partition on authenticated identity** (`oid`/`sub`), falling back to IP. Never partition on IP alone for authenticated APIs — NAT kills you.
- **Token bucket** for APIs (burst + sustain). **Sliding window** for strict quota. **Concurrency** for expensive endpoints (exports, reports).
- Emit `Retry-After`. Surface the policy name in problem details for debugging.
- This is **per-instance**. For accurate global limits, use Redis/Envoy/APIM in front. Built-in is good enough for abuse prevention.

---

## 9. Output Caching

`AddOutputCache` supersedes response caching for most scenarios. Response caching obeys `Cache-Control` negotiation; output caching gives you explicit server-side control with tag invalidation.

```csharp
builder.Services.AddOutputCache(o =>
{
    o.AddPolicy("PublicShort", b => b.Expire(TimeSpan.FromSeconds(30)).Tag("public"));
    o.AddPolicy("PerUser", b => b.SetVaryByRouteValue("tenantId").VaryByValue((ctx, _) =>
        new("u", ctx.User.FindFirstValue("oid") ?? "")).Expire(TimeSpan.FromMinutes(5)));
});

app.UseOutputCache();

catalog.MapGet("/{id}", GetProduct).CacheOutput("PublicShort");
// Invalidate:
await cacheStore.EvictByTagAsync("public", ct);
```

Do:
- Cache only GETs that are safe to serve stale-for-a-short-window.
- Use **tags** for fan-out invalidation (`product:{id}`, `tenant:{id}`).
- Vary by auth-relevant things (tenant, role) or explicitly bypass for authenticated users.
- For multi-instance, back it with **Redis** (`Microsoft.Extensions.Caching.StackExchangeRedis` + custom `IOutputCacheStore`). In-memory = divergence.

Don't:
- Cache personalized responses under a shared key. Audit `VaryByValue`.
- Rely on output cache for rate limiting or expensive-computation gating — use rate limiter and `IMemoryCache`+`LazyCache` respectively.

---

## 10. Authn/Authz

JWT Bearer + policy-based authorization. See [`validation.md`](https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/validation.md) for Entra-specific token validation and [`best-practices.md`](https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/best-practices.md) for credential hygiene. The rules below are the **minimum inline checks** every API should encode in policy — don't rely on a downstream document to remember them.

### 10.1 Token validation invariants

Every bearer token your API accepts must clear these checks before any policy runs. Microsoft.Identity.Web wires most of them, but you are responsible for verifying the configuration:

- **Signature** — keys fetched from OIDC metadata; never pin keys, never disable signature validation.
- **`iss` (issuer)** — `https://login.microsoftonline.com/<tid>/v2.0` for v2; `https://sts.windows.net/<tid>/` for v1. Single-tenant: pin `TenantId`. Multi-tenant: implement `IssuerValidator` with an explicit tenant allow-list and verify `tid` matches the issuer's tenant segment.
- **`aud` (audience)** — pin to the API's client ID (GUID) for v2; App ID URI (`api://…`) for v1. Accept both during v1↔v2 migration via `ValidAudiences`. Never disable audience validation.
- **`exp` / `nbf` (lifetime)** — leave `ValidateLifetime = true` and the default 5-minute clock skew.
- **`ver`** — know which version your API issues (`requestedAccessTokenVersion` in the manifest); don't silently accept the other.
- **`scp` (delegated) vs `roles` (app)** — never share one policy that accepts either. Branch explicitly (see 10.2).
- **`azp` / `appid`** — for endpoints that accept app tokens, allow-list the calling client app IDs. Roles alone are not enough if the role is broad.
- **CAE / claims challenges** — honor `WWW-Authenticate: Bearer error="insufficient_claims"` from downstream APIs; emit it from your own API for step-up (ACRS) using `ReplyForbiddenWithWwwAuthenticateHeaderAsync` so the `claims=` parameter is preserved.

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

builder.Services.Configure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, o =>
{
    o.MapInboundClaims = false;                    // keep "scp"/"roles"/"azp" verbatim
    o.TokenValidationParameters.ValidAudiences = new[]
    {
        builder.Configuration["AzureAd:ClientId"]!,        // v2 (GUID)
        $"api://{builder.Configuration["AzureAd:ClientId"]}" // v1 (App ID URI), drop after migration
    };
    o.TokenValidationParameters.ValidateIssuer   = true;
    o.TokenValidationParameters.ValidateAudience = true;
    o.TokenValidationParameters.ValidateLifetime = true;
});
```

### 10.2 Policies — split delegated and app

Treat "user can do X" and "service can do X" as separate capabilities, even when they map to the same business operation. This mirrors Entra's claim model (`scp` for delegated, `roles` for app) and the sample's `RequireClientApp` allow-list.

```csharp
const string ScopeClaim = "http://schemas.microsoft.com/identity/claims/scope"; // "scp" when MapInboundClaims=false set above; pick one consistently
var allowedClientApps = builder.Configuration.GetSection("Auth:AllowedClientApps").Get<string[]>() ?? Array.Empty<string>();

builder.Services.AddAuthorization(o =>
{
    o.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser().Build();        // deny-by-default

    // Delegated (user) — must carry scp; must NOT be an app-only token.
    o.AddPolicy("OrdersReadDelegated", p => p
        .RequireAuthenticatedUser()
        .RequireAssertion(ctx =>
        {
            var scp = ctx.User.FindFirst("scp")?.Value
                   ?? ctx.User.FindFirst(ScopeClaim)?.Value;
            return scp is not null
                && scp.Split(' ').Contains("Orders.Read");
        }));

    // App (S2S) — must carry roles AND azp/appid in the allow-list AND no scp.
    o.AddPolicy("OrdersReadApp", p => p
        .RequireAuthenticatedUser()
        .RequireAssertion(ctx =>
        {
            var hasScp   = ctx.User.HasClaim(c => c.Type == "scp" || c.Type == ScopeClaim);
            var hasRole  = ctx.User.HasClaim("roles", "Orders.ReadAll");
            var clientId = ctx.User.FindFirst("azp")?.Value      // v2
                        ?? ctx.User.FindFirst("appid")?.Value;   // v1
            return !hasScp
                && hasRole
                && clientId is not null
                && allowedClientApps.Contains(clientId);
        }));

    o.AddPolicy("Admin", p => p.Requirements.Add(new AdminRequirement()));
});
builder.Services.AddScoped<IAuthorizationHandler, AdminRequirementHandler>();

// Endpoint that accepts either user or app — use a composite policy that
// runs the right branch, never a single assertion that ORs the claims.
orders.MapGet("/{id:guid}", GetOrder)
      .RequireAuthorization("OrdersReadDelegated", "OrdersReadApp"); // ASP.NET evaluates as OR across named policies; each policy itself enforces its own invariants.
```

> Why two policies, not one assertion that ORs `scp` and `roles`?
> A naive `HasScope("X") || HasAppRole("Y")` lets an app token satisfy a user-only endpoint (no `azp` allow-list, no `scp`/`roles` separation), and lets a user token satisfy an app-only endpoint. Both are real privilege-escalation bugs in production APIs. The split also lets you log, meter, and rate-limit the two paths separately.

Do:
- **Deny by default** via `FallbackPolicy`. Opt in to anonymous with `[AllowAnonymous]` / `.AllowAnonymous()`.
- **One policy per capability per token type** (`OrdersReadDelegated`, `OrdersReadApp`). Requirements + handlers for anything beyond claim presence.
- **App-only endpoints**: enforce `roles` *and* `azp`/`appid` allow-list *and* absence of `scp`. The allow-list is the primary defense; roles alone are not enough.
- For dynamic policies (tenant-scoped, per-resource), implement `IAuthorizationPolicyProvider`.
- Use **resource-based** auth (`IAuthorizationService.AuthorizeAsync(user, resource, "EditOrder")`) for "can this user edit *this* order".
- Inject keyed authorization handlers with `[FromKeyedServices]` when the same requirement has tenant- or product-specific evaluators.

Don't:
- `[Authorize(Roles = "Admin")]` when roles come from tokens with non-standard claim types — it will silently fail. Use policies.
- Put business rules in JWT claims that change often (stale tokens).
- Disable `ValidateIssuer`, `ValidateAudience`, or `ValidateLifetime`. Ever.

Sources:
- [Microsoft identity platform — access tokens](https://learn.microsoft.com/entra/identity-platform/access-tokens) — `scp`, `roles`, `azp`, `appid`, `tid`, `ver`.
- [Microsoft.Identity.Web — protected web API](https://learn.microsoft.com/entra/msal/dotnet/microsoft-identity-web/web-apis).
- [Authorization in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authorization/introduction) — policies, requirements, handlers, fallback policy.
- [RFC 6750 — Bearer Token Usage](https://datatracker.ietf.org/doc/html/rfc6750).
- [Continuous Access Evaluation (CAE)](https://learn.microsoft.com/entra/identity/conditional-access/concept-continuous-access-evaluation) and [claims challenges](https://learn.microsoft.com/entra/identity-platform/claims-challenge).
- Companion: [`validation.md`](https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/validation.md), [`best-practices.md`](https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/best-practices.md).

---

## 11. HttpClient

Always `IHttpClientFactory`. **Never `new HttpClient()`** in a long-lived process — socket exhaustion and stale DNS.

```csharp
builder.Services.AddHttpClient<IGraphClient, GraphClient>(c =>
{
    c.BaseAddress = new Uri("https://graph.microsoft.com/v1.0/");
    c.DefaultRequestVersion = HttpVersion.Version20;
    c.DefaultVersionPolicy  = HttpVersionPolicy.RequestVersionOrHigher;
    c.Timeout = TimeSpan.FromSeconds(100);        // belt; resilience handler is braces
})
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(2),   // rotates DNS
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(1),
    EnableMultipleHttp2Connections = true
})
.AddStandardResilienceHandler()
.SetHandlerLifetime(Timeout.InfiniteTimeSpan);   // see below
```

Rules:
- Use **typed clients** (`AddHttpClient<TClient, TImpl>`) over named clients. Typed clients are injectable and testable.
- If you set `PooledConnectionLifetime` on a `SocketsHttpHandler`, set `HandlerLifetime` to `InfiniteTimeSpan` — otherwise the factory rotates handlers *and* you pay DNS refresh twice.
- For high-throughput services hitting one endpoint, bypass the factory's rotation with a singleton `HttpClient` over a `SocketsHttpHandler` with `PooledConnectionLifetime` set. The factory is great; it's not mandatory.
- Set HTTP/2 (or /3 behind Envoy/YARP) explicitly; don't rely on the server to upgrade.

---

## 12. Background Work

Prefer **`BackgroundService`** over raw `IHostedService`. It gives you `ExecuteAsync(CancellationToken)` and wired shutdown.

```csharp
public sealed class Dispatcher(IServiceScopeFactory scopes, ILogger<Dispatcher> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await using var scope = scopes.CreateAsyncScope();
            try { await scope.ServiceProvider.GetRequiredService<IWorker>().RunOnceAsync(ct); }
            catch (OperationCanceledException) when (ct.IsCancellationRequested) { break; }
            catch (Exception ex) { Log.DispatcherFailed(log, ex); }
            await Task.Delay(TimeSpan.FromSeconds(1), ct);
        }
    }
}
```

Do:
- `CreateAsyncScope` per unit of work — `BackgroundService` itself is a singleton and can't resolve scoped services directly.
- Honor the `CancellationToken` on every `await`. Cooperative shutdown is a contract, not a suggestion.
- Configure **graceful shutdown** budget: `builder.Services.Configure<HostOptions>(o => o.ShutdownTimeout = TimeSpan.FromSeconds(30))`. Kestrel: `ConfigureKestrel(k => k.Limits...)`. In K8s set `terminationGracePeriodSeconds` ≥ this + probe delays.
- **Idempotency** is mandatory for queue workers: use message ID dedup tables, or outbox + at-least-once semantics. Assume every message can be redelivered.
- Queue-backed workers: **Azure Service Bus** (sessions for FIFO per key, dead-letter, scheduled), **RabbitMQ** (quorum queues, publisher confirms), or **Kafka** for log-style. Do not invent a DB-polling queue unless you have a reason.

Don't:
- `async void` handlers. Ever.
- Swallow `OperationCanceledException` during shutdown — let the loop exit.
- Do long CPU-bound work without yielding; you'll block the graceful shutdown signal.

---

## 13. Configuration

Layered sources, environment-per-ring:
1. `appsettings.json` (committed defaults, no secrets).
2. `appsettings.{Environment}.json` (ring-specific non-secret overrides).
3. **Key Vault** (`Azure.Extensions.AspNetCore.Configuration.Secrets` or Key Vault references in App Configuration / App Service).
4. Environment variables (last-resort overrides, NOT secrets).

```csharp
builder.Services.AddOptions<BillingOptions>()
    .Bind(builder.Configuration.GetSection("Billing"))
    .ValidateDataAnnotations()
    .ValidateOnStart();          // fail fast at boot if misconfigured
```

Do:
- **`ValidateOnStart`** on every options object. Miss this and you'll find out about typos in production at the first request.
- `IOptionsSnapshot<T>` for per-request reload (cheap, scoped). `IOptionsMonitor<T>` for singletons that need change notifications. `IOptions<T>` for truly static config.
- Key Vault references via **App Configuration** so rotation flows without redeploy.

Don't:
- Put secrets in environment variables in production. They leak via `/proc/*/environ`, dumps, and diagnostic endpoints.
- Read `IConfiguration` directly deep in the code. Bind to a typed options class.

---

## 14. Security

Transport:
- `UseHttpsRedirection()` and `UseHsts()` (not in dev). Production HSTS max-age ≥ 1 year, `includeSubDomains`, and — once confident — submit for preload.
- Behind a proxy: `UseForwardedHeaders` **before** auth, with `KnownProxies`/`KnownNetworks` set. Blindly trusting `X-Forwarded-For` is a spoofing bug.

CSRF — depends on how the endpoint authenticates the caller, not on the service as a whole. Use this matrix:

| Endpoint authenticates via | Antiforgery required? | Why |
|---|---|---|
| `Bearer` token only (Authorization header) | No | Tokens aren't sent automatically by the browser; no ambient credential to forge. |
| Cookie (session, `.AspNetCore.Identity.*`) | **Yes** — `AddAntiforgery` + token validation | Browser attaches cookies on cross-origin POSTs by default. |
| Cookie + Bearer fallback (hybrid) | **Yes for the cookie-backed path** | Treat each endpoint by its strongest ambient credential; protect every cookie-callable endpoint. |
| BFF (browser → BFF cookie session → API via server-side token) | **Yes on the BFF**; API behind it is bearer-only | The BFF is the cookie surface; the upstream API never sees the cookie. |

Implementation:
- Minimal APIs: `app.UseAntiforgery()` and apply `[ValidateAntiForgeryToken]` / `RequireAntiforgery()` per endpoint or group; opt cookie-free endpoints out with `DisableAntiforgery()`.
- MVC: enable globally via `AddControllersWithViews(o => o.Filters.Add<AutoValidateAntiforgeryTokenAttribute>())`; `[IgnoreAntiforgeryToken]` for pure-bearer controllers.
- For SPA-on-cookie BFFs, ship the antiforgery token in a JS-readable cookie (e.g. `XSRF-TOKEN`) and have the SPA echo it as a header.

CORS:
- Name policies; **never** `AllowAnyOrigin` with `AllowCredentials` — the browser rejects it and you've advertised a bug. List specific origins; use wildcards only for dev.
- CORS runs *before* auth. Preflight `OPTIONS` must succeed without credentials.

Security headers:
- `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Content-Security-Policy` (APIs: `default-src 'none'; frame-ancestors 'none'`), `Permissions-Policy`. Use `NetEscapades.AspNetCore.SecurityHeaders` or write middleware; don't hand-roll per response.
- Do not set `Server` header (Kestrel: `AddServerHeader = false`).

Data Protection:
- In Kubernetes or scaled-out: **persist keys** (`PersistKeysToAzureBlobStorage` or a mounted PVC) and **protect them** (`ProtectKeysWithAzureKeyVault` / Azure Managed HSM). Default in-memory keys = every pod rotates its own = cookies/auth tickets break on restart.

Secrets:
- No secrets in repos, images, env vars (prod), or pipeline variables. Key Vault + Managed Identity / Workload Identity. See [`best-practices.md`](https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/best-practices.md).

Sources:
- [Enforce HTTPS / HSTS](https://learn.microsoft.com/aspnet/core/security/enforcing-ssl).
- [Antiforgery in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/anti-request-forgery).
- [CORS in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/cors).
- [Data Protection — key storage providers](https://learn.microsoft.com/aspnet/core/security/data-protection/implementation/key-storage-providers).
- [BFF pattern guidance — Duende / IdentityServer](https://docs.duendesoftware.com/identityserver/v7/bff/) (concept reference; pattern, not endorsement).

---

## 15. CORS gotchas, Uploads, Kestrel, Forwarded Headers

CORS:
- The browser will swallow errors if your middleware order is wrong. `UseRouting` → `UseCors` → `UseAuthentication` → `UseAuthorization` → endpoints.
- Exposing custom response headers requires `WithExposedHeaders` — `Retry-After`, `Location`, correlation IDs, etc.

Uploads / streaming:
- For large files, **disable buffering**: `DisableRequestSizeLimit` on the endpoint *plus* stream from `HttpContext.Request.Body` (or `IFormFile.OpenReadStream()`). Never read `.ToArrayAsync()` on a multi-GB payload.
- Enforce size on the way in: `Kestrel.Limits.MaxRequestBodySize`, per-endpoint `[RequestSizeLimit]` / `.WithMetadata(new RequestSizeLimitAttribute(...))`.
- For downloads, return `Results.Stream(stream, contentType, filename)` or `PhysicalFile` — do not buffer into memory.

Kestrel limits — review every one for your threat model:
- `MaxConcurrentConnections`, `MaxConcurrentUpgradedConnections`, `MaxRequestBodySize`, `MaxRequestHeadersTotalSize`, `MinRequestBodyDataRate`, `MinResponseDataRate`, `KeepAliveTimeout`, `RequestHeadersTimeout`.
- Defaults are reasonable for internet-facing, but behind an ingress controller you often want to *lower* `MaxRequestBodySize` and *raise* `KeepAliveTimeout`.

Forwarded headers (behind YARP/App Gateway/nginx/Envoy):

```csharp
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto | ForwardedHeaders.XForwardedHost;
    o.KnownNetworks.Clear(); o.KnownProxies.Clear();
    o.KnownNetworks.Add(new IPNetwork(IPAddress.Parse("10.0.0.0"), 8));  // your VNet/pod CIDR
});
app.UseForwardedHeaders();   // FIRST middleware
```

Without `KnownNetworks`/`KnownProxies`, ASP.NET Core ignores forwarded headers from non-loopback by design. Many `RemoteIp = 127.0.0.1` bugs trace to this.

---

## 16. gRPC

Choose gRPC when:
- Internal service-to-service, you control both ends, and you want typed contracts + HTTP/2 multiplexing.
- Streaming (client/server/bidi) maps naturally to your domain.
- Polyglot clients benefit from generated code.

Stick with REST when:
- Public API, browsers are first-class clients, or CDN/caching matters.
- Your clients are .NET-only and `Refit` + JSON is fine — don't introduce gRPC just for typing.

Rules:
- **Proto-first**. Code-first (`protobuf-net.Grpc`) is tempting in .NET-only orgs, but proto files are the durable contract — language-agnostic, diff-able, versioned.
- Always set **deadlines** on the client (`CallOptions.Deadline`). No deadline = server leak under load.
- Flow the server `CancellationToken` into every downstream call.
- For browsers, use **gRPC-Web** (`UseGrpcWeb`) or gRPC-JSON transcoding. Native gRPC isn't reachable from browsers.
- `ConfigureKestrel` to accept HTTP/2 (`ListenAnyIP(..., o => o.Protocols = HttpProtocols.Http2)`) and allow plaintext only in dev.

---

## 17. SignalR

For real-time. Rules differ from stateless APIs because connections are **long-lived and sticky**:

- **Scale-out needs a backplane.** Options: **Azure SignalR Service** (recommended — offloads connection management, sharded, auto-scaled) or **Redis backplane** (`Microsoft.AspNetCore.SignalR.StackExchangeRedis`).
- Without a backplane, a message sent from pod A cannot reach a client connected to pod B. Bugs in this category are hard to reproduce in dev.
- With self-hosted SignalR (no Azure SignalR), **sticky sessions are mandatory** for WebSocket negotiation fallback to long-polling. In K8s that means session affinity on the Service/Ingress.
- Azure SignalR removes the sticky requirement — clients connect to it, not to your pods.
- Authenticate Hubs via `[Authorize]`; pass the JWT via `accessTokenFactory` on the client. Don't invent a second auth channel.
- Backpressure: set `MaximumReceiveMessageSize`, `StreamBufferCapacity`, and handle `HubException` on the client. One rogue client should not OOM the server.
- Log with `ActivitySource` — SignalR invocations are first-class spans.

---

## Sources

### Authoritative (Microsoft / .NET team)

- [What's new in ASP.NET Core in .NET 10](https://learn.microsoft.com/aspnet/core/release-notes/aspnetcore-10.0) — OpenAPI 3.1, validation, auth metrics, route syntax highlighting.
- [Minimal APIs overview](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview) — route groups, filters, typed results.
- [`TypedResults` vs `Results`](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/responses) — when to use each.
- [Validation in Minimal APIs (.NET 10)](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/min-api-validation) — `AddValidation`, `[Validatable]`, source-generated validators.
- [Handle errors — `IExceptionHandler` and `AddProblemDetails`](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling) — the pattern replacing delegate-based exception handlers.
- [Asp.Versioning (API versioning)](https://github.com/dotnet/aspnet-api-versioning/wiki) — community-maintained but the de facto standard.
- [Microsoft.AspNetCore.OpenApi](https://learn.microsoft.com/aspnet/core/fundamentals/openapi/overview) — built-in OpenAPI in .NET 9/10 and transformer APIs.
- [ASP.NET Core logging](https://learn.microsoft.com/aspnet/core/fundamentals/logging/) and [`LoggerMessage` source generator](https://learn.microsoft.com/dotnet/core/extensions/logger-message-generator) — structured, zero-alloc logging.
- [OpenTelemetry for .NET](https://learn.microsoft.com/dotnet/core/diagnostics/observability-with-otel) — tracing + metrics + logs guidance.
- [`Microsoft.Extensions.Http.Resilience`](https://learn.microsoft.com/dotnet/core/resilience/http-resilience) — `AddStandardResilienceHandler`, hedging, Polly v8 strategies.
- [Rate limiting middleware](https://learn.microsoft.com/aspnet/core/performance/rate-limit) — built-in algorithms and partitioning.
- [Output caching](https://learn.microsoft.com/aspnet/core/performance/caching/output) — policies, tag invalidation, store customization.
- [Authorization in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authorization/introduction) — policies, requirements, handlers, fallback policy.
- [`IHttpClientFactory` guidelines](https://learn.microsoft.com/dotnet/core/extensions/httpclient-factory) and [`SocketsHttpHandler.PooledConnectionLifetime`](https://learn.microsoft.com/dotnet/api/system.net.http.socketshttphandler.pooledconnectionlifetime).
- [`BackgroundService` & `IHostedService`](https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services) and [Graceful shutdown](https://learn.microsoft.com/aspnet/core/fundamentals/host/generic-host#host-shutdown).
- [Options pattern, `ValidateOnStart`](https://learn.microsoft.com/dotnet/core/extensions/options) — bind, validate, fail fast.
- [Data Protection — key storage providers](https://learn.microsoft.com/aspnet/core/security/data-protection/implementation/key-storage-providers) — persist & protect in K8s.
- [Forwarded Headers Middleware](https://learn.microsoft.com/aspnet/core/host-and-deploy/proxy-load-balancer) — proxy config, `KnownNetworks`/`KnownProxies`.
- [Kestrel limits](https://learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel/options) — every knob explained.
- [gRPC on .NET](https://learn.microsoft.com/aspnet/core/grpc/) and [gRPC-Web](https://learn.microsoft.com/aspnet/core/grpc/browser) — protocols, deadlines, browser story.
- [SignalR scale-out](https://learn.microsoft.com/aspnet/core/signalr/scale) and [Azure SignalR Service](https://learn.microsoft.com/azure/azure-signalr/signalr-overview) — backplane choices.
- [.NET Architecture e-books](https://dotnet.microsoft.com/learn/dotnet/architecture-guides) — "Cloud Native", "Microservices", "Modernizing .NET Apps" — still the best long-form guidance from the team.
- [devblogs.microsoft.com/dotnet — ASP.NET Core tag](https://devblogs.microsoft.com/dotnet/category/aspnet/) — preview posts are where new APIs get explained first.

### Community

- [Andrew Lock — andrewlock.net](https://andrewlock.net/) — deep, careful dives on middleware, auth, configuration, source generators. The reference blog for ASP.NET Core internals.
- [David Fowler — gists and talks](https://gist.github.com/davidfowl) — especially [davidfowl/AspNetCoreDiagnosticScenarios](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios): async/await, `HttpClient`, and threading traps straight from the architect.
- [Khalid Abuhakmeh — khalidabuhakmeh.com](https://khalidabuhakmeh.com/) — pragmatic tips, Minimal APIs, tooling.
- [Nick Chapsas — YouTube](https://www.youtube.com/@nickchapsas) — current .NET features explained quickly; good for "what changed in the latest preview".
- [Steve Gordon — stevejgordon.co.uk](https://www.stevejgordon.co.uk/) and NDC talks — `HttpClientFactory` internals, performance, diagnostics.
- [Jimmy Bogard — jimmybogard.com](https://www.jimmybogard.com/) — domain modeling, MediatR, vertical slice architecture, ProblemDetails patterns.
- [Gérald Barré (Meziantou) — meziantou.net](https://www.meziantou.net/) — edge cases, analyzers, source generators, niche perf.
- [Maarten Balliauw](https://blog.maartenballiauw.be/) — NuGet/Kestrel/ASP.NET internals.
- [Scalar — scalar.com](https://scalar.com/) — OpenAPI UI alternative to Swagger UI, first-class ASP.NET Core integration.
- [Polly docs — pollydocs.org](https://www.pollydocs.org/) — v8 strategy semantics beyond what `AddStandardResilienceHandler` exposes.
