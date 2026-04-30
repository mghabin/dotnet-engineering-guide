# ASP.NET Core 10 — Server-Side Best Practices

Opinionated defaults for APIs and background services on **.NET 10 (LTS, Nov 2025)**.
Terse on purpose.
If something isn't here, assume the framework default is fine.

Cross-chapter ownership (per [`coverage-map.md`](../coverage-map.md)):
- DI lifetimes, options pattern, logging primitives, source-generated loggers — owned by [Chapter 01 — foundations](./01-foundations.md).
- EF Core registration, `DbContext` scoping, exception translation — owned by [Chapter 03 — data](./03-data.md).
- `WebApplicationFactory`, Testcontainers, hosted-service tests — owned by [Chapter 04 — testing](./04-testing.md).
- OTel application metrics, meter design, NativeAOT, GC tuning — owned by [Chapter 05 — performance](./05-performance.md).
- OTLP exporter wiring, Aspire `ServiceDefaults`, K8s probes, workload identity — owned by [Chapter 06 — cloud-native](./06-cloud-native.md).
- Blazor auth-state serialization, render-mode-specific auth — owned by [Chapter 07 — client](./07-client.md).

Canonical reference for §10 (AuthN/AuthZ) on Entra: the [**EntraAuthPatterns sample repo**](https://github.com/mghabin/entra-auth-patterns-dotnet) — runnable code, `validation.md`, `best-practices.md`.

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
- [OWASP API Security Top 10 — API8:2023 Security Misconfiguration](https://owasp.org/API-Security/editions/2023/en/0xa8-security-misconfiguration/) — endpoint metadata (auth, CORS, rate-limit) belongs on the route group, not scattered per endpoint.
- DI surface inside endpoints/filters references the lifetime rules in [Chapter 01 — foundations §DI](./01-foundations.md).

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
- [FluentValidation — ASP.NET Core integration](https://docs.fluentvalidation.net/en/latest/aspnet.html) — auto-validation in `FluentValidation.AspNetCore` is deprecated; invoke validators yourself.
- [Model binding](https://learn.microsoft.com/aspnet/core/mvc/models/model-binding) and [`[AsParameters]`](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/parameter-binding#parameter-binding-for-argument-lists-with-asparameters).
- [Keyed DI services](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#keyed-services) — `[FromKeyedServices]`.
- [OWASP API Security Top 10 — API3:2023 Broken Object Property Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa3-broken-object-property-level-authorization/) — bind explicit DTOs, never the persistence model.
- DataAnnotations, `IValidatableObject`, options validation primitives owned by [Chapter 01 — foundations](./01-foundations.md).

---

## 3. ProblemDetails & Error Handling

Rule: **every app-generated 4xx/5xx is `application/problem+json`** (RFC 9457, which obsoletes RFC 7807). The middleware below covers exception-driven and status-code-driven responses; auth challenges (401/403 from the JWT bearer handler) need explicit handling — see "Auth challenges" below — because the bearer handler emits its own `WWW-Authenticate` response that you must preserve.

The in-box `Microsoft.AspNetCore.Mvc.ProblemDetails` type predates 9457, and the wire format is byte-compatible with 7807 — same members (`type`, `title`, `status`, `detail`, `instance`), same `application/problem+json` media type, no new package.
Only the `type` URI semantics tightened: maintain a per-org problem-type catalog at a stable URL (e.g. `https://errors.example.com/orders/concurrency-conflict`) and never use the RFC URL itself as `type` for app errors — it conveys no domain meaning and clients that branch on `type` will collide across unrelated errors.

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
- [RFC 9457 — Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc9457) (obsoletes RFC 7807; wire-format compatible).
- [RFC 9457 §4 — defining new problem types](https://datatracker.ietf.org/doc/html/rfc9457#section-4) — `type` URI conventions and registry guidance.
- [RFC 7807 — original Problem Details](https://datatracker.ietf.org/doc/html/rfc7807) — kept for clients that branch on the older `type` URIs during migration.
- [RFC 6750 §3 — `WWW-Authenticate` for bearer challenges](https://datatracker.ietf.org/doc/html/rfc6750#section-3) — `error`, `error_description`, `scope` parameters preserved on 401.
- [Handle errors — `IExceptionHandler` and `AddProblemDetails`](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling).
- [JWT bearer events](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.authentication.jwtbearer.jwtbearerevents) — preserving `WWW-Authenticate`.
- [Microsoft.Identity.Web claims challenges / CAE](https://learn.microsoft.com/entra/identity-platform/claims-challenge).
- [OWASP API Security Top 10 — API7:2023 Server Side Request Forgery / API8:2023 Misconfiguration](https://owasp.org/API-Security/editions/2023/en/0xa8-security-misconfiguration/) — never leak stack traces or internal URIs in `detail`.
- EF Core `DbUpdateConcurrencyException` → 409 ProblemDetails mapping owned by [Chapter 03 — data](./03-data.md).

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
- Mark deprecated versions via `.HasDeprecatedApiVersion(1.0)`. This sets `api-deprecated-versions` automatically (when `ReportApiVersions = true`), but **`Sunset` headers do not flow automatically** — register an explicit sunset policy inside `AddApiVersioning` via `options.Policies.Sunset(...)` (the supported `Asp.Versioning.Http` API surface; there is no `AddSunsetPolicyBuilder()` extension):

```csharp
builder.Services.AddApiVersioning(o =>
{
    o.Policies.Sunset(new ApiVersion(1, 0))
        .Effective(DateTimeOffset.Parse("2026-06-30T00:00:00Z"))
        .Link("https://docs.example.com/api/v1-sunset")
            .Title("Migration guide");
    // …existing options…
}).AddApiExplorer(/* … */);
```

- Don't version on minor patches; only breaking contract changes justify a new major.

Sources:
- [Asp.Versioning wiki — versioning policies](https://github.com/dotnet/aspnet-api-versioning/wiki).
- [Asp.Versioning wiki — Sunset Policy](https://github.com/dotnet/aspnet-api-versioning/wiki/Sunset-Policy) — `ApiVersioningOptions.Policies.Sunset(...)`.
- [`Asp.Versioning.Http` on NuGet](https://www.nuget.org/packages/Asp.Versioning.Http) — current package surface.
- [`Asp.Versioning.Http.OpenApi` on NuGet](https://www.nuget.org/packages/Asp.Versioning.Http.OpenApi) — bridge into `Microsoft.AspNetCore.OpenApi` document-per-version.
- [RFC 8594 — the Sunset HTTP header](https://datatracker.ietf.org/doc/html/rfc8594).

---

## 5. OpenAPI

**`Microsoft.AspNetCore.OpenApi`** has been the default since **.NET 9** — Swashbuckle was removed from the `dotnet new webapi` template in 9.0, not 10.0.
It emits **OpenAPI 3.1 / JSON Schema 2020-12 by default starting in .NET 9** (`OpenApiOptions.OpenApiVersion` overrides); .NET 10 adds YAML serving (`MapOpenApi("/openapi/{documentName}.yaml")`).
Swashbuckle still works but is community-maintained and lags 3.1.

```csharp
builder.Services.AddOpenApi("v1", o =>
{
    o.AddDocumentTransformer<BearerSecuritySchemeTransformer>();
    o.AddOperationTransformer<AddCorrelationIdHeaderTransformer>();
});
builder.Services.AddOpenApi("v2");        // one document per major version

app.MapOpenApi();                          // /openapi/v1.json, /openapi/v2.json
app.MapOpenApi("/openapi/{documentName}.yaml");   // .NET 10
```

- Customize via `IOpenApiDocumentTransformer` / `IOpenApiOperationTransformer` / `IOpenApiSchemaTransformer`. No more `ISchemaFilter` gymnastics.
- For the UI, pick **[Scalar](https://scalar.com/)** (`Scalar.AspNetCore`) — it's what the Microsoft samples now show. Use Swagger UI only if you have established tooling around it.
- Don't commit generated `.json`/`.yaml` to the repo unless it's the contract (contract-first). Let CI publish it.

**Bearer security scheme transformer.** The most common Swashbuckle migration question is "where did the Authorize button go?" — `AddOpenApi` has no built-in bearer scheme; you add one via a document transformer that probes the registered authentication schemes:

```csharp
internal sealed class BearerSecuritySchemeTransformer(IAuthenticationSchemeProvider schemes)
    : IOpenApiDocumentTransformer
{
    public async Task TransformAsync(OpenApiDocument doc, OpenApiDocumentTransformerContext _, CancellationToken ct)
    {
        if (!(await schemes.GetAllSchemesAsync()).Any(s => s.Name == JwtBearerDefaults.AuthenticationScheme))
            return;
        doc.Components ??= new OpenApiComponents();
        doc.Components.SecuritySchemes["Bearer"] = new OpenApiSecurityScheme
        {
            Type = SecuritySchemeType.Http, Scheme = "bearer", BearerFormat = "JWT",
            In = ParameterLocation.Header, Description = "RFC 6750 bearer token"
        };
        doc.SecurityRequirements.Add(new OpenApiSecurityRequirement
        {
            [new OpenApiSecurityScheme { Reference = new() { Type = ReferenceType.SecurityScheme, Id = "Bearer" } }] = []
        });
    }
}
```

**Versioned documents.** Pair each `AddApiVersioning(...).AddApiExplorer()` major version with its own `AddOpenApi("vN")` and a one-line operation transformer that filters `ApiDescription.GroupName == documentName` so `MapGroup("/api/v{version:apiVersion}")` endpoints land in the right document.
The `Asp.Versioning.Http.OpenApi` package supplies the `IApiDescriptionProvider` glue — without it, versioned endpoints either miss every document or duplicate across all of them.

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
- [Generate OpenAPI documents (`Microsoft.AspNetCore.OpenApi`)](https://learn.microsoft.com/aspnet/core/fundamentals/openapi/aspnetcore-openapi?view=aspnetcore-9.0) — `AddOpenApi`/`MapOpenApi`, `OpenApiVersion`, build-time generation.
- [What's new in ASP.NET Core 9.0](https://learn.microsoft.com/aspnet/core/release-notes/aspnetcore-9.0) — built-in OpenAPI shipped in 9.0.
- [Customize OpenAPI documents with transformers](https://learn.microsoft.com/aspnet/core/fundamentals/openapi/customize-openapi) — document/operation/schema transformers.
- [`Asp.Versioning.Http.OpenApi` on NuGet](https://www.nuget.org/packages/Asp.Versioning.Http.OpenApi) — versioned-document bridge.
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
- [OpenTelemetry .NET — getting started (community)](https://opentelemetry.io/docs/languages/net/getting-started/) — vendor-neutral wiring of `ActivitySource`/`Meter` to OTLP.
- [OpenTelemetry Semantic Conventions for HTTP](https://opentelemetry.io/docs/specs/semconv/http/http-spans/) — span/attribute names that `AddAspNetCoreInstrumentation` and `AddHttpClientInstrumentation` emit.
- [OpenTelemetry Specification — Trace Context (W3C)](https://www.w3.org/TR/trace-context/) — propagation contract used by the HTTP instrumentations.
- [`opentelemetry-dotnet` GitHub](https://github.com/open-telemetry/opentelemetry-dotnet) — exporter/instrumentation issue tracker; check before pinning a preview.
- Logging primitives owned by [Chapter 01 — foundations](./01-foundations.md); meter design + application metrics owned by [Chapter 05 — performance](./05-performance.md); OTLP exporters and `ServiceDefaults` owned by [Chapter 06 — cloud-native](./06-cloud-native.md).

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
Apply it once to every typed client with `services.ConfigureHttpClientDefaults(b => b.AddStandardResilienceHandler())`; for clients that need a different pipeline, chain `.RemoveAllResilienceHandlers().AddResilienceHandler("name", b => …)` (or `AddStandardHedgingHandler()`) so the override is explicit instead of forking the defaults.

Pipeline order (outermost → innermost): rate limiter → total timeout → retry → circuit breaker → attempt timeout.
Defaults: total timeout 30s, retry max 3 with exponential backoff + jitter (base 2s), circuit breaker at 10% failure ratio over 30s sampling with 5s break, attempt timeout 10s.
Transient triggers: HTTP `5xx`, `408`, `429`, plus `HttpRequestException` and `TimeoutRejectedException`.

**Do:**

- Make `AddStandardResilienceHandler` the default for every outbound `HttpClient` — apply it once via `services.ConfigureHttpClientDefaults(b => b.AddStandardResilienceHandler())`.
- Call **`o.Retry.DisableForUnsafeHttpMethods()`** unless every mutating endpoint you call accepts an `Idempotency-Key` header and dedups server-side — the standard handler retries **all** HTTP methods by default, including POST/PATCH/PUT/DELETE.
- Set a **total request timeout** (end-to-end budget) *and* a per-attempt timeout — without the total, retries stack unboundedly.
- Tune the circuit breaker on failure ratio + sampling duration, not raw counts.
- Use `AddStandardHedgingHandler` only for read-heavy fan-out across replicas/regions — never hedge writes or non-idempotent calls.
- To customize, call **`RemoveAllResilienceHandlers()`** first, then add a single configured pipeline — pass options to `AddStandardResilienceHandler` instead of forking defaults to tune one knob.
- Reach for a custom `AddResilienceHandler("name", b => …)` pipeline only when you need non-standard ordering, custom predicates, or strategies the standard handler doesn't expose.

**Don't:**

- Don't assume the standard handler skips POST/PATCH — it does not; you must opt out explicitly with `DisableForUnsafeHttpMethods()` (or a `Retry.ShouldHandle` predicate that requires an idempotency key).
- Don't stack resilience handlers — chaining a second `AddResilienceHandler` or hand-rolled Polly handler on the same `HttpClient` produces nested retries and circuit breakers that interact in surprising ways.
- Don't stack retries across layers (client + gateway + mesh) — pick one layer.
- Don't circuit-break on every exception — exclude `OperationCanceledException` from request cancellation.
- Don't fall back by silently swallowing a 500 — fallback should return an empty list or a cached snapshot, and surface the failure to telemetry.

Sources:
- [Build resilient HTTP apps with `Microsoft.Extensions.Http.Resilience`](https://learn.microsoft.com/dotnet/core/resilience/http-resilience) — definitive for `HttpStandardResilienceOptions`, `HttpRetryStrategyOptions`, pipeline order, and `RemoveAllResilienceHandlers()`.
- [`Microsoft.Extensions.Http.Resilience` on NuGet](https://www.nuget.org/packages/Microsoft.Extensions.Http.Resilience).
- [Polly v8 strategies (community)](https://www.pollydocs.org/strategies/index.html) — semantics beyond what the standard handler exposes.
- [Polly v8 — pipelines and resilience strategies](https://www.pollydocs.org/pipelines/index.html) — ordering rules when you build a custom pipeline.
- [Polly GitHub — `App-vNext/Polly`](https://github.com/App-vNext/Polly) — issues, samples, and chaos-engineering helpers (`Polly.Simmy`).
- [Resilient apps with `Microsoft.Extensions.Resilience`](https://learn.microsoft.com/dotnet/core/resilience/).
- [OWASP API Security Top 10 — API4:2023 Unrestricted Resource Consumption](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/) — total + per-attempt timeouts are the primary defense against runaway retry storms.

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
- This is **per-instance**. For accurate global limits across replicas, terminate at APIM / Envoy / a Redis-backed Lua limiter — there is no first-party distributed limiter as of .NET 10. Built-in is good enough for abuse prevention.
- Wire the **`Microsoft.AspNetCore.RateLimiting`** meter into the §6 OTel `AddMeter(...)` list so dashboards see lease/queue/rejection counters; pair with `DisableRateLimiting()` on endpoints (e.g. health probes) that must never be throttled.

Sources:
- [Rate limiting middleware in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/rate-limit) — algorithms, partitioning, `DisableRateLimiting()`.
- [ASP.NET Core built-in metrics](https://learn.microsoft.com/aspnet/core/log-mon/metrics/built-in) — `Microsoft.AspNetCore.RateLimiting` meter.
- [System.Threading.RateLimiting reference](https://learn.microsoft.com/dotnet/api/system.threading.ratelimiting) — token-bucket / sliding-window / fixed-window primitives.
- [OWASP API Security Top 10 — API4:2023 Unrestricted Resource Consumption](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/) — partition on identity, not IP, for authenticated APIs.
- [RFC 6585 §4 — `429 Too Many Requests`](https://datatracker.ietf.org/doc/html/rfc6585#section-4) and [RFC 7231 §7.1.3 — `Retry-After`](https://datatracker.ietf.org/doc/html/rfc7231#section-7.1.3) — wire-format expectations.
- OTel meter wiring referenced from §6 and owned by [Chapter 05 — performance](./05-performance.md).

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
- Remember the **default policy never caches authenticated requests** (`HttpContext.User.Identity.IsAuthenticated` short-circuits) — call `b.Cache()` inside the policy to opt back in *after* you have proven `VaryByValue` covers every identity dimension.
- For multi-instance, back it with a distributed `IOutputCacheStore`. **Microsoft does not ship a Redis `IOutputCacheStore`** — implement one over `IDistributedCache` (Set/GetByTag indexes) or use `Aspire.StackExchange.Redis.OutputCaching` from the .NET Aspire stack. Adding `Microsoft.Extensions.Caching.StackExchangeRedis` alone changes nothing about output caching.

Don't:
- Cache personalized responses under a shared key. Audit `VaryByValue`.
- Rely on output cache for rate limiting or expensive-computation gating — use rate limiter and `IMemoryCache`+`LazyCache` respectively.

Sources:
- [Output caching middleware in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/caching/output) — policies, tag invalidation, default-policy short-circuit on auth.
- [Output caching — `IOutputCacheStore` extensibility](https://learn.microsoft.com/aspnet/core/performance/caching/output#ioutputcachestore) — required surface for a custom (e.g. Redis) store.
- [RFC 9111 — HTTP Caching](https://datatracker.ietf.org/doc/html/rfc9111) — `Cache-Control`/`Vary`/`Age` semantics that output caching deliberately bypasses.
- [`Aspire.StackExchange.Redis.OutputCaching` package](https://www.nuget.org/packages/Aspire.StackExchange.Redis.OutputCaching) — first-party distributed `IOutputCacheStore`.

---

## 10. Authn/Authz

JWT Bearer + policy-based authorization.
**Canonical reference for everything in this section:** [**EntraAuthPatterns sample**](https://github.com/mghabin/entra-auth-patterns-dotnet) — runnable `Program.cs`, the per-claim validation in [`docs/validation.md`](https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/validation.md), and credential-hygiene rules in [`docs/best-practices.md`](https://github.com/mghabin/entra-auth-patterns-dotnet/blob/main/docs/best-practices.md).
The rules below are the **minimum inline checks** every API must encode in policy — don't rely on a downstream document to remember them.

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
- **Canonical sample:** [`mghabin/entra-auth-patterns-dotnet`](https://github.com/mghabin/entra-auth-patterns-dotnet) — runnable Entra JWT validation, policy split, CAE wiring; this section's rules are derived from it.
- [RFC 6749 — OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749) — grant types, `scope` parameter semantics that map to the `scp` claim.
- [RFC 6750 — Bearer Token Usage](https://datatracker.ietf.org/doc/html/rfc6750) — `Authorization: Bearer`, `WWW-Authenticate` challenge format, error codes.
- [RFC 8252 — OAuth 2.0 for Native Apps](https://datatracker.ietf.org/doc/html/rfc8252) — public-client constraints; PKCE is mandatory, no embedded user-agents.
- [RFC 9068 — JWT profile for OAuth 2.0 access tokens](https://datatracker.ietf.org/doc/html/rfc9068) — required claims (`iss`, `aud`, `exp`, `sub`, `client_id`, `scope`) and the `at+jwt` typ header.
- [RFC 9449 — DPoP (sender-constrained tokens)](https://datatracker.ietf.org/doc/html/rfc9449) — for when bearer-as-string is no longer enough (e.g. open-banking-class APIs).
- [OWASP API Security Top 10 — API2:2023 Broken Authentication](https://owasp.org/API-Security/editions/2023/en/0xa2-broken-authentication/) — the threat model behind the policy split below.
- [OWASP API Security Top 10 — API5:2023 Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/) — why deny-by-default `FallbackPolicy` is non-negotiable.
- [OWASP ASVS v4 — V3 Session / V4 Access Control](https://owasp.org/www-project-application-security-verification-standard/) — verification checklist that maps cleanly onto policy/requirement code.
- [Access token claims reference (Microsoft Entra)](https://learn.microsoft.com/entra/identity-platform/access-token-claims-reference) — exact semantics of `scp`, `roles`, `azp`, `appid`, `tid`, `ver`.
- [Microsoft identity platform — access tokens](https://learn.microsoft.com/entra/identity-platform/access-tokens) — overview and v1↔v2 differences.
- [Microsoft.Identity.Web — protected web API](https://learn.microsoft.com/entra/msal/dotnet/microsoft-identity-web/web-apis).
- [Authorization in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authorization/introduction) — policies, requirements, handlers, fallback policy.
- [Continuous Access Evaluation (CAE)](https://learn.microsoft.com/entra/identity/conditional-access/concept-continuous-access-evaluation) and [claims challenges](https://learn.microsoft.com/entra/identity-platform/claims-challenge).
- [Continuous Access Evaluation in Microsoft.Identity.Web](https://learn.microsoft.com/entra/msal/dotnet/microsoft-identity-web/continuous-access-evaluation) — outbound `tokenAcquisitionOptions.Claims` retry path.
- Blazor auth-state serialization and render-mode-specific token flow owned by [Chapter 07 — client](./07-client.md).

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

**CAE / claims-challenge on outbound calls.** When a downstream API returns `401` with `WWW-Authenticate: Bearer error="insufficient_claims", claims="…"`, the cached token is stale by Entra's policy — retrying with the same token (which the standard resilience handler will happily do on a 5xx-like predicate) loops forever.
Parse the base64url-encoded `claims` parameter and re-acquire via `ITokenAcquisition.GetAccessTokenForUserAsync(scopes, tokenAcquisitionOptions: new() { Claims = claimsChallenge })` (or `GetAccessTokenForAppAsync` for app-only) before the next attempt; treat raw 401 as non-retryable in `Retry.ShouldHandle` so the resilience handler hands control back to the auth layer.

Sources:
- [`IHttpClientFactory` guidelines](https://learn.microsoft.com/dotnet/core/extensions/httpclient-factory) and [`SocketsHttpHandler.PooledConnectionLifetime`](https://learn.microsoft.com/dotnet/api/system.net.http.socketshttphandler.pooledconnectionlifetime).
- [Continuous Access Evaluation in Microsoft.Identity.Web](https://learn.microsoft.com/entra/msal/dotnet/microsoft-identity-web/continuous-access-evaluation) — outbound CAE retry path.
- [Claims challenges, claims requests, and client capabilities](https://learn.microsoft.com/entra/identity-platform/claims-challenge).
- [RFC 6749 §6 — refreshing an access token](https://datatracker.ietf.org/doc/html/rfc6749#section-6) and [RFC 6750 §3](https://datatracker.ietf.org/doc/html/rfc6750#section-3) — bearer/refresh wire format the CAE retry obeys.
- [`davidfowl/AspNetCoreDiagnosticScenarios`](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios) — `HttpClient` lifetime/socket-exhaustion failure modes documented end-to-end.
- OTLP exporter wiring for outbound HTTP spans owned by [Chapter 06 — cloud-native](./06-cloud-native.md).

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

Sources:
- [`BackgroundService` & `IHostedService`](https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services) and [Graceful shutdown](https://learn.microsoft.com/aspnet/core/fundamentals/host/generic-host#host-shutdown).
- [Azure Service Bus — sessions, dead-lettering, scheduled messages](https://learn.microsoft.com/azure/service-bus-messaging/message-sessions).
- [RabbitMQ — quorum queues](https://www.rabbitmq.com/quorum-queues.html) and [publisher confirms](https://www.rabbitmq.com/confirms.html).
- [Apache Kafka — consumer semantics](https://kafka.apache.org/documentation/#semantics) — at-least-once/exactly-once trade-offs that drive idempotency design.
- Hosted-service unit and integration testing patterns owned by [Chapter 04 — testing](./04-testing.md).

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

Sources:
- [Configuration providers in .NET](https://learn.microsoft.com/dotnet/core/extensions/configuration-providers) — provider order, binding rules.
- [Azure Key Vault configuration provider](https://learn.microsoft.com/aspnet/core/security/key-vault-configuration).
- [Azure App Configuration — Key Vault references](https://learn.microsoft.com/azure/azure-app-configuration/use-key-vault-references-dotnet-core) — rotation without redeploy.
- Options pattern, `IOptionsSnapshot<T>`/`IOptionsMonitor<T>`, `ValidateOnStart` owned by [Chapter 01 — foundations](./01-foundations.md).
- Workload Identity / Managed Identity wiring owned by [Chapter 06 — cloud-native](./06-cloud-native.md).

---

## 14. Security

Transport:
- `UseHttpsRedirection()` and `UseHsts()` (not in dev). Production HSTS max-age ≥ 1 year, `includeSubDomains`, and — once confident — submit for preload.
- Behind a proxy: `UseForwardedHeaders` **before** auth, with `KnownProxies`/`KnownNetworks` set. Blindly trusting `X-Forwarded-For` is a spoofing bug.
- **Middleware order behind a TLS-terminating proxy:** `UseForwardedHeaders()` → `UseHsts()` → `UseHttpsRedirection()` → `UseRouting()` → `UseCors()` → `UseAuthentication()` → `UseAuthorization()` → endpoints. Without forwarded headers running first, `Request.Scheme` is `http` after termination, HSTS is silently skipped, and `UseHttpsRedirection()` will either no-op or loop.

CSRF — depends on how the endpoint authenticates the caller, not on the service as a whole. Use this matrix:

| Endpoint authenticates via | Antiforgery required? | Why |
|---|---|---|
| `Bearer` token only (Authorization header) | No | Tokens aren't sent automatically by the browser; no ambient credential to forge. |
| Cookie (session, `.AspNetCore.Identity.*`) | **Yes** — `AddAntiforgery` + token validation | Browser attaches cookies on cross-origin POSTs by default. |
| Cookie + Bearer fallback (hybrid) | **Yes for the cookie-backed path** | Treat each endpoint by its strongest ambient credential; protect every cookie-callable endpoint. |
| BFF (browser → BFF cookie session → API via server-side token) | **Yes on the BFF**; API behind it is bearer-only | The BFF is the cookie surface; the upstream API never sees the cookie. |

Implementation:
- Minimal APIs (≥ .NET 8): `app.UseAntiforgery()` **automatically validates** the token on any endpoint that binds `IFormFile`, `IFormFileCollection`, or `[FromForm]` — no `RequireAntiforgery()` needed. Pure-bearer JSON endpoints have no form binding and are not affected. Use `.DisableAntiforgery()` only on form-binding endpoints that are provably not browser-callable (e.g. an S2S file upload behind an app-only token).
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
- [RFC 6797 — HTTP Strict Transport Security](https://datatracker.ietf.org/doc/html/rfc6797) — `max-age`, `includeSubDomains`, preload semantics.
- [RFC 7239 — Forwarded HTTP Extension](https://datatracker.ietf.org/doc/html/rfc7239) — the standardized successor to `X-Forwarded-*`.
- [OWASP API Security Top 10 — API8:2023 Security Misconfiguration](https://owasp.org/API-Security/editions/2023/en/0xa8-security-misconfiguration/) — TLS, headers, CORS, error verbosity.
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/) — current recommended values for CSP/Permissions-Policy/Referrer-Policy.
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) — drives the matrix above.
- [OWASP Cross-Origin Resource Sharing Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#cross-origin-resource-sharing) — `AllowAnyOrigin` + credentials is a known anti-pattern.
- Workload Identity / Managed Identity, key rotation, secret-store wiring owned by [Chapter 06 — cloud-native](./06-cloud-native.md).

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
- **Default for internet-facing pods behind an ingress controller:** lower `MaxRequestBodySize` to the largest legitimate payload (1–10 MiB for JSON APIs, explicit per-endpoint override for upload routes) and raise `KeepAliveTimeout` to 2 minutes to amortize TLS handshake cost over warm connections from the ingress.
- **Fallback:** keep framework defaults (30 MiB body, 130 s keep-alive) only for internal-only services that the ingress already protects with its own limits.

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

Sources:
- [CORS in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/cors) — preflight ordering and exposed headers.
- [Kestrel limits](https://learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel/options) — every knob explained.
- [Forwarded Headers Middleware](https://learn.microsoft.com/aspnet/core/host-and-deploy/proxy-load-balancer) — `KnownNetworks`/`KnownProxies` requirement.
- [RFC 7239 — Forwarded HTTP Extension](https://datatracker.ietf.org/doc/html/rfc7239) — standardized successor to `X-Forwarded-*`.
- [OWASP API Security Top 10 — API4:2023 Unrestricted Resource Consumption](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/) — body-size and rate caps belong on the ingress *and* on Kestrel.
- [`Microsoft.AspNetCore.HttpOverrides` source](https://github.com/dotnet/aspnetcore/tree/main/src/Middleware/HttpOverrides/src) — exact ordering rules used by the middleware.

---

## 16. gRPC

**Default: REST + JSON over HTTP/2.** Choose gRPC only for the cases below; REST is the fallback because it composes with browsers, CDNs, and existing telemetry without extra plumbing.

Choose gRPC when **all** of the following hold:
- Internal service-to-service, you control both ends, and you want typed contracts + HTTP/2 multiplexing.
- Streaming (client/server/bidi) maps naturally to your domain.
- Polyglot clients benefit from generated code.

Stay on REST when any of the following hold:
- Public API, browsers are first-class clients, or CDN/caching matters.
- Your clients are .NET-only and `Refit` + JSON covers the typing need.

Rules:
- **Proto-first.** Code-first (`protobuf-net.Grpc`) is tempting in .NET-only orgs, but proto files are the durable contract — language-agnostic, diff-able, versioned.
- Always set **deadlines** on the client (`CallOptions.Deadline`). No deadline = server leak under load.
- Flow the server `CancellationToken` into every downstream call.
- For browsers, use **gRPC-Web** (`UseGrpcWeb`) as the default; fall back to gRPC-JSON transcoding only when you must serve OpenAPI from the same proto. Native gRPC isn't reachable from browsers.
- `ConfigureKestrel` to accept HTTP/2 (`ListenAnyIP(..., o => o.Protocols = HttpProtocols.Http2)`) and allow plaintext only in dev.

Sources:
- [gRPC on .NET](https://learn.microsoft.com/aspnet/core/grpc/) and [gRPC-Web](https://learn.microsoft.com/aspnet/core/grpc/browser).
- [gRPC official docs (community/CNCF)](https://grpc.io/docs/) — protocol semantics, deadlines, status codes.
- [Protocol Buffers — language guide (proto3)](https://protobuf.dev/programming-guides/proto3/) — versioning rules (`reserved`, field numbers).
- [grpc-ecosystem/awesome-grpc](https://github.com/grpc-ecosystem/awesome-grpc) — interceptors, gateways, observability adapters.
- [OpenTelemetry Semantic Conventions for RPC](https://opentelemetry.io/docs/specs/semconv/rpc/) — span/attribute names emitted by the gRPC instrumentation.

---

## 17. SignalR

For real-time. Rules differ from stateless APIs because connections are **long-lived and sticky**:

- **Default backplane: Azure SignalR Service** (offloads connection management, sharded, auto-scaled, removes the sticky-session requirement).
- **Self-hosted fallback: Redis backplane** (`Microsoft.AspNetCore.SignalR.StackExchangeRedis`) — pick this only when Azure SignalR is unavailable (sovereign cloud, on-prem, regulatory).
- Without a backplane, a message sent from pod A cannot reach a client connected to pod B. Bugs in this category are hard to reproduce in dev.
- With self-hosted SignalR (no Azure SignalR), **sticky sessions are mandatory** for WebSocket negotiation fallback to long-polling. In K8s that means session affinity on the Service/Ingress.
- Authenticate Hubs via `[Authorize]`; pass the JWT via `accessTokenFactory` on the client. Don't invent a second auth channel.
- Backpressure: set `MaximumReceiveMessageSize`, `StreamBufferCapacity`, and handle `HubException` on the client. One rogue client should not OOM the server.
- Log with `ActivitySource` — SignalR invocations are first-class spans.

Sources:
- [SignalR scale-out](https://learn.microsoft.com/aspnet/core/signalr/scale) and [Azure SignalR Service](https://learn.microsoft.com/azure/azure-signalr/signalr-overview) — backplane choices.
- [SignalR — security considerations](https://learn.microsoft.com/aspnet/core/signalr/security) — auth, CORS, message-size limits.
- [RFC 6455 — The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455) — handshake/upgrade contract that drives sticky-session and ingress config.
- [OWASP API Security Top 10 — API4:2023 Unrestricted Resource Consumption](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/) — `MaximumReceiveMessageSize` and per-connection caps are the primary backpressure defense.
- Blazor Server hub lifetimes and circuit auth-state owned by [Chapter 07 — client](./07-client.md).

---

## 18. Health Checks

The health-check probe contract — endpoint naming (`/health/live`, `/health/ready`, `/health/startup`), tag-driven predicates, what each probe must and must not include, and how to cache expensive checks — is owned by **`06-cloud-native.md` §10**.
That chapter aligns the endpoints with the Aspire ServiceDefaults convention and with the Kubernetes probe model, so the contract lives next to the deployment surface that consumes it.
This chapter intentionally does not restate it; wire `AddHealthChecks(...)` per ch06 §10 and map endpoints exactly as documented there.

Sources:
- ch06 §10 — health-check probe contract and endpoint naming.
- [Health checks in ASP.NET Core](https://learn.microsoft.com/aspnet/core/host-and-deploy/health-checks) — `MapHealthChecks`, predicates, tags, `IHealthCheckPublisher`.

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
- [Health checks in ASP.NET Core](https://learn.microsoft.com/aspnet/core/host-and-deploy/health-checks) — live/ready/startup split, predicates, publishers.
- [.NET Architecture e-books](https://dotnet.microsoft.com/learn/dotnet/architecture-guides) — "Cloud Native", "Microservices", "Modernizing .NET Apps" — still the best long-form guidance from the team.
- [devblogs.microsoft.com/dotnet — ASP.NET Core tag](https://devblogs.microsoft.com/dotnet/category/aspnet/) — preview posts are where new APIs get explained first.

### Community

- [Andrew Lock — andrewlock.net](https://andrewlock.net/) — deep, careful dives on middleware, auth, configuration, source generators. The reference blog for ASP.NET Core internals.
- [David Fowler — gists and talks](https://gist.github.com/davidfowl) — especially [davidfowl/AspNetCoreDiagnosticScenarios](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios): async/await, `HttpClient`, and threading traps straight from the architect.
- [Khalid Abuhakmeh — khalidabuhakmeh.com](https://khalidabuhakmeh.com/) — pragmatic tips, Minimal APIs, tooling.
- [Steve Gordon — stevejgordon.co.uk](https://www.stevejgordon.co.uk/) and NDC talks — `HttpClientFactory` internals, performance, diagnostics.
- [Jimmy Bogard — jimmybogard.com](https://www.jimmybogard.com/) — domain modeling, MediatR, vertical slice architecture, ProblemDetails patterns.
- [Gérald Barré (Meziantou) — meziantou.net](https://www.meziantou.net/) — edge cases, analyzers, source generators, niche perf.
- [Maarten Balliauw](https://blog.maartenballiauw.be/) — NuGet/Kestrel/ASP.NET internals.
- [Scalar — scalar.com](https://scalar.com/) — OpenAPI UI alternative to Swagger UI, first-class ASP.NET Core integration.
- [Polly docs — pollydocs.org](https://www.pollydocs.org/) — v8 strategy semantics beyond what `AddStandardResilienceHandler` exposes.
- [`mghabin/entra-auth-patterns-dotnet`](https://github.com/mghabin/entra-auth-patterns-dotnet) — canonical sample for §10 (JWT validation, policy split, CAE).

### Standards & specifications (IETF / W3C)

- [RFC 6749 — OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749), [RFC 6750 — Bearer Token Usage](https://datatracker.ietf.org/doc/html/rfc6750), [RFC 8252 — OAuth 2.0 for Native Apps](https://datatracker.ietf.org/doc/html/rfc8252), [RFC 9068 — JWT access-token profile](https://datatracker.ietf.org/doc/html/rfc9068), [RFC 9449 — DPoP](https://datatracker.ietf.org/doc/html/rfc9449).
- [RFC 9457 — Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc9457) (obsoletes [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807)).
- [RFC 6797 — HSTS](https://datatracker.ietf.org/doc/html/rfc6797), [RFC 7239 — Forwarded HTTP Extension](https://datatracker.ietf.org/doc/html/rfc7239), [RFC 8594 — Sunset header](https://datatracker.ietf.org/doc/html/rfc8594), [RFC 9111 — HTTP Caching](https://datatracker.ietf.org/doc/html/rfc9111), [RFC 6455 — WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455).
- [W3C Trace Context](https://www.w3.org/TR/trace-context/) — propagation contract used by OTel HTTP instrumentation.

### Security baselines (OWASP)

- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x00-header/) — primary threat model for §3, §7, §8, §10, §14, §17.
- [OWASP ASVS v4](https://owasp.org/www-project-application-security-verification-standard/) — verification checklist that maps onto policy/requirement code.
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/) — current recommended values for §14 headers.
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) — drives the §14 CSRF matrix.

### Observability (OpenTelemetry)

- [OpenTelemetry .NET — getting started](https://opentelemetry.io/docs/languages/net/getting-started/) and [`opentelemetry-dotnet` GitHub](https://github.com/open-telemetry/opentelemetry-dotnet).
- [OpenTelemetry Semantic Conventions — HTTP](https://opentelemetry.io/docs/specs/semconv/http/http-spans/) and [RPC](https://opentelemetry.io/docs/specs/semconv/rpc/) — span/attribute names emitted by ASP.NET Core, `HttpClient`, and gRPC instrumentations.
