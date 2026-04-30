# Testing .NET 10 Apps — Opinionated Best Practices

This is a dense, opinionated guide.
It assumes `net10.0`, `Microsoft.Testing.Platform` (MTP), and xUnit v3 GA (shipped Dec 16, 2024).
Do/Don't notation is intentional.
If you disagree, write your own doc.

**Defaults at a glance** (each one is justified in its own section below):

- Test framework: **xUnit v3** under MTP. Fallback: MSTest v3 (also MTP-native). See §2.
- Assertion library: **Shouldly** for new code. Fallback: built-in `Assert` (zero deps) or **AwesomeAssertions** if migrating off FluentAssertions v8. See §3.
- Mock library: **NSubstitute**. Fallback: hand-rolled fakes for narrow ports. Never Moq in new code. See §4.
- Integration-test harness: **`WebApplicationFactory<Program>` + Testcontainers + Respawn**. Fallback: **Aspire `DistributedApplicationTestingBuilder`** when the system under test is the whole AppHost graph (see [Chapter 06 — cloud-native](./06-cloud-native.md)). See §5–§6.
- Time: **`TimeProvider` + `FakeTimeProvider`**. See §14.
- Snapshot: **Verify**. See §7.
- Property tests: **FsCheck** (mature) or **CsCheck** (C#-first). See §10.

**Cross-chapter wiring** (this chapter is the test-harness owner; the rules being tested live elsewhere):

- DI lifetimes that constrain how fakes and fixtures register → [Chapter 01 — foundations §DI lifetimes](./01-foundations.md).
- The endpoints, ProblemDetails contracts, and JWT/Entra surface that `WebApplicationFactory` exercises → [Chapter 02 — aspnetcore](./02-aspnetcore.md).
- EF Core modeling, migrations, and the outbox pattern that integration tests target → [Chapter 03 — data](./03-data.md).
- Aspire AppHost composition that `DistributedApplicationTestingBuilder` boots → [Chapter 06 — cloud-native](./06-cloud-native.md).

---

## 1. The pyramid (and why most teams get it wrong)

Three tiers. Ratios are guidance, not law:

| Tier | What it tests | Target share | Execution budget |
|---|---|---|---|
| **Unit** | Pure domain logic, parsers, mappers, policies. No I/O, no clock, no random. | 65–80% | < 10 ms each, whole suite < 30 s |
| **Integration** | One process, real dependencies via Testcontainers (DB, broker, cache), ASP.NET Core via `WebApplicationFactory`. | 15–30% | < 1 s each, whole suite < 5 min |
| **E2E / system** | Deployed environment, real auth, real network. Smoke + critical journeys only. | 1–5% | Minutes; run post-deploy or nightly |

**Do**
- Push logic *down* into pure functions so you can unit-test them without ceremony.
- Treat integration tests as the load-bearing tier for anything with SQL, HTTP, or messaging. In-memory substitutes lie.
- Run E2E against a real deployed environment, not against `WebApplicationFactory` pretending to be prod.

**Don't**
- Don't write unit tests for trivial getters, DI registrations, or `Controller` wiring — `WebApplicationFactory` covers those for free.
- Don't "integration test" by spinning up `WebApplicationFactory` with EF Core `UseInMemoryDatabase` — you're testing a fiction (no transactions, no constraints, no provider-specific SQL). Use Testcontainers.
- Don't chase a 1:1 test:code ratio. Chase *behaviors*.
- **Don't hit real cloud resources by default in CI.** Use containers (Testcontainers) or local emulators (Azurite, Cosmos DB Emulator, LocalStack) for the integration tier. Live cloud is reserved for explicitly tagged smoke / post-deploy jobs against a dedicated test subscription, never the default `dotnet test` run. Real cloud in PR CI is slow, flaky, leaks credentials, and racks up bills.

Sources:
- xUnit.net — *Comparing xUnit.net to other frameworks* — https://xunit.net/docs/comparisons
- Testcontainers for .NET — *Why Testcontainers?* — https://dotnet.testcontainers.org/
- learn.microsoft.com — *Integration tests in ASP.NET Core* — https://learn.microsoft.com/aspnet/core/test/integration-tests
- learn.microsoft.com — *Use Azurite emulator for local Azure Storage development* — https://learn.microsoft.com/azure/storage/common/storage-use-azurite

---

## 2. Frameworks: xUnit v3, MSTest, NUnit, TUnit

**Default: xUnit v3.**
It is GA (1.0.0, Dec 2024) and is the canonical choice for new .NET code.
**Fallback: MSTest v3** if your org already standardizes on it — both are MTP-native and the gap is small.
NUnit 4 is fine to keep on existing projects; do not migrate for its own sake.
TUnit is the right pick only when source-gen discovery or Native AOT tests are hard requirements.

### xUnit v3: what actually changed from v2

- **Tests run in a dedicated process.** v2 ran tests in-process with the runner (reflection-only loading of your assembly). v3 compiles each test project into an **executable** that the runner launches. Consequences:
  - `Main` entry point exists; you can debug a test project with F5.
  - Static state no longer leaks between unrelated test runs on the same machine.
  - `AppDomain` tricks, `ILogger` capture via shared statics, and module initializers behave like a real app.
- **Async lifetime for fixtures.** `IAsyncLifetime` is first-class for `IClassFixture<T>` and `ICollectionFixture<T>` — stop doing sync-over-async in `ctor`/`Dispose`.
- **`Assert.Equivalent(expected, actual)`** — deep structural equality built-in. Kills a big reason to reach for FluentAssertions.
- **`[Theory]` data via `TheoryData<>`** is strongly typed; prefer it over `object[][]`.
- **Cancellation tokens** are available to tests via `TestContext.Current.CancellationToken`. Wire long-running tests to honor it.
- **First-class `Microsoft.Testing.Platform` support.** v3 test projects are produced as self-contained executables (project SDK is `Microsoft.NET.Sdk`, no separate runner package required); `dotnet test` works, and so does running the test `.exe` directly.
- **Dynamically skippable tests.** `Assert.Skip(...)` skips at runtime; `[Fact(Explicit = true)]` excludes a test from default runs; `[Fact(SkipUnless = ...)]` / `SkipWhen` gate by predicate. No more `if (cond) return;` workarounds.
- **Templates.** `dotnet new install xunit.v3.templates`, then `dotnet new xunit3`.

### Microsoft.Testing.Platform (MTP): strategic direction, opt-in today

MTP is the modern runner host (not a test framework). By early 2025 every major framework — xUnit v3, MSTest, NUnit, **TUnit** (which is built entirely on MTP) — has an MTP adapter. **In .NET 10, `dotnet test` runs in two modes**: the legacy VSTest mode (still the **default**) and the new dedicated MTP mode. MTP is the strategic direction, but **it is opt-in today** — enable it in `global.json`:

```json
{
  "test": {
    "runner": "Microsoft.Testing.Platform"
  }
}
```

Requires the MTP runner packages at **1.7+**. The property is `test.runner` — *not* `sdk.testRunner`. The older `<TestingPlatformDotnetTestSupport>true</TestingPlatformDotnetTestSupport>` MSBuild property is the legacy opt-in that runs MTP-aware projects under VSTest mode; that bridge is being removed in MTP v2 on the .NET 10 SDK, so prefer the `global.json` switch for new work.

Enable MTP **per framework** — the property names differ. Don't paste MSTest properties into an xUnit project.

**xUnit v3** (`xunit.v3` package):

```xml
<PropertyGroup>
  <OutputType>Exe</OutputType>
  <UseMicrosoftTestingPlatformRunner>true</UseMicrosoftTestingPlatformRunner>
</PropertyGroup>
<ItemGroup>
  <PackageReference Include="xunit.v3" Version="1.*" />
  <!-- Optional during transition: keeps VS Test Explorer / `dotnet test` VSTest mode working -->
  <!-- <PackageReference Include="xunit.runner.visualstudio" Version="3.*" /> -->
  <!-- <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" /> -->
</ItemGroup>
```

**MSTest** uses `<EnableMSTestRunner>true</EnableMSTestRunner>` instead. **NUnit** uses its own `NUnit.Runner` package wiring. Don't cross the streams.

**Do**
- For xUnit v3, use `xunit.v3` + `<UseMicrosoftTestingPlatformRunner>true</UseMicrosoftTestingPlatformRunner>` + `<OutputType>Exe</OutputType>`.
- During transition you may keep `xunit.runner.visualstudio` and `Microsoft.NET.Test.Sdk` alongside MTP so VS Test Explorer and CI keep working until your runners flip — they coexist in xUnit v3 projects.
- Use MTP's first-class trx/coverage/retry extensions instead of VSTest logger packages where available.
- Pin runner mode globally with `global.json` so all projects agree.

**Don't**
- Don't put `<EnableMSTestRunner>true</EnableMSTestRunner>` in an xUnit project — that property is MSTest-specific and does nothing for xUnit.
- Don't claim "MTP replaces VSTest" yet. Both modes ship in .NET 10; teams must opt-in.
- Don't block on VSTest-only tooling for new projects — MTP is the direction of travel.

### Output capture (xUnit v3)

xUnit captures only what you route through it. Two mechanisms:

- **`ITestOutputHelper`** — inject in the test class constructor, write per-test (`output.WriteLine(...)`). This is the primary mechanism.
- **Assembly-level capture** — `[assembly: CaptureConsole]` and `[assembly: CaptureTrace]` route `Console.Out`/`Console.Error`/`Trace` from the test's logical context into the test's output.

**Caveat:** assembly-level capture only works for code running on the test's logical execution context. **Output written from background worker threads, `Task.Run` on threadpool threads detached from the test, or hosted services running in `WebApplicationFactory` may not be captured** and will silently drop. If you need that output, plumb it through your own `ILogger` + `FakeLogger`/`ITestOutputHelper` sink.

### MSTest / NUnit

- MSTest v3 has MTP integration (`<EnableMSTestRunner>true</EnableMSTestRunner>`), dynamic data sources, and is perfectly reasonable.
- NUnit (4.x) is fine, but `[SetUp]`/`[TearDown]` per-test ceremony tempts shared mutable state. xUnit's constructor-per-test model is better for isolation. If you stay on NUnit, use the **constraint model** (`Assert.That(actual, Is.EqualTo(expected))`) — it composes with `&`, `|`, `Is.Not`, `Has`, `Does` and is the canonical NUnit 4 syntax (the classic `Assert.AreEqual` API was removed from the default surface).
- **TUnit** is the newest entrant: built entirely on MTP, source-generated test discovery, parallel-by-default, Native AOT support. Microsoft Learn lists it alongside xUnit/NUnit/MSTest as a supported MTP-native framework. Adoption is still smaller; pick it when those properties (AOT, source-gen) actually matter, otherwise stick with xUnit v3.

Sources:
- xUnit.net — *What's New in v3* — https://xunit.net/docs/getting-started/v3/whats-new
- xUnit.net — *Microsoft.Testing.Platform support* — https://xunit.net/docs/getting-started/v3/microsoft-testing-platform
- xUnit.net — *Capturing output* — https://xunit.net/docs/capturing-output
- xUnit.net — *Migrating from v2 to v3* — https://xunit.net/docs/getting-started/v3/migration
- TUnit — *Project README* (MTP-native, source-gen discovery) — https://github.com/thomhurst/TUnit
- Microsoft .NET Blog — *MTP adoption across frameworks* — https://devblogs.microsoft.com/dotnet/mtp-adoption-frameworks/
- learn.microsoft.com — *Microsoft.Testing.Platform overview* — https://learn.microsoft.com/dotnet/core/testing/microsoft-testing-platform-intro

---

## 3. Assertions

**Default: Shouldly for new code.**
**Fallback: built-in `Assert` (xUnit v3) when you want zero extra dependencies, or AwesomeAssertions when migrating an existing FluentAssertions codebase.**
Add **Verify** alongside any of the above for structural / snapshot assertions (§7).

### Built-in `Assert` (xUnit v3)
Covers ~90% of needs now that `Assert.Equivalent`, `Assert.Collection`, `Assert.Multiple` exist. Zero dependencies, no license drama.

### FluentAssertions — ⚠️ license change
FluentAssertions **v8 (Jan 2025)** moved from Apache-2.0 to the **Xceed Community License**: free for non-commercial, ~$130/developer/year for commercial use. v7.x remains Apache-2.0. Implications:

**Do**
- Pin to `FluentAssertions < 8.0.0` if you want to stay free and Apache-licensed.
- Or purchase a commercial license — FA's fluent API is still the best in class for some assertions (`BeEquivalentTo` with config).
- Or migrate to **Shouldly** (MIT, actively maintained) or built-in `Assert.Equivalent`.

**Don't**
- Don't silently upgrade past 7.x in a commercial repo. Your CI will not fail, but your legal team might.
- Don't re-invent FA in helper methods — use Shouldly.

### Shouldly
MIT. Clean error messages (`result.ShouldBe(42)` prints the expression). Prefer for new projects.

### Verify (snapshot/approval)
Use for anything with non-trivial structured output: DTO shapes, generated SQL, HTTP responses, serialized payloads. See §7.

### Post-FluentAssertions migration matrix

If you're moving off FA v8 for licensing reasons, pick by goal — not by hype.

| Option | License | Best for | Migration cost from FA |
|---|---|---|---|
| Built-in `Assert` (xUnit v3) | Apache-2.0 | Minimum dependencies; teams happy with `Assert.Equal`/`Assert.Equivalent`/`Assert.Collection`. | High — manual rewrite; no fluent chains. |
| **AwesomeAssertions** | Apache-2.0 (FA v7 fork) | Teams with large existing FA codebases that want a near drop-in. Same `Should()` API, same `BeEquivalentTo` semantics. | **Lowest** — global namespace swap (`FluentAssertions` → `AwesomeAssertions`) gets most code compiling. |
| Shouldly | MIT | New / greenfield projects. Clean expression-aware messages (`result.ShouldBe(42)`). | Medium — different API surface, must rewrite. |
| Verify | MIT | Anything structured / snapshot-shaped. Complement, not replacement, for scalar assertions. | N/A — different paradigm. |
| TUnit assertions | MIT | Only if you're also moving to TUnit as the test framework. Don't adopt the assertions standalone. | High — couples you to a different runner. |

**Do**
- Default to **AwesomeAssertions** if your repo has thousands of `Should()` calls — that fork is the lowest-friction exit from FA v8 licensing.
- Use built-in `Assert` for new projects without legacy FA debt.
- Combine with Verify for structured output regardless of which scalar library you pick.

**Don't**
- Don't run a months-long manual port to Shouldly/`Assert` if AwesomeAssertions can ship today. The fork exists precisely for this case.
- Don't fork FA yourself — there are already maintained forks.

Sources:
- xUnit.net — *What's New in v3* (assertions) — https://xunit.net/docs/getting-started/v3/whats-new
- Shouldly — *Project README* — https://github.com/shouldly/shouldly
- Shouldly — *Documentation* — https://docs.shouldly.org/
- AwesomeAssertions — *Project README* (Apache-2.0 fork of FluentAssertions v7) — https://github.com/AwesomeAssertions/AwesomeAssertions
- Verify — *Project README* — https://github.com/VerifyTests/Verify
- TUnit — *Assertions* — https://github.com/thomhurst/TUnit/tree/main/TUnit.Assertions
- Xceed — *Fluent Assertions FAQ* (v8 licensing) — https://xceed.com/fluent-assertions-faq/

---

## 4. Mocks and fakes

**Default: NSubstitute.**
**Fallback: hand-rolled fakes** for narrow ports (1–3 methods).
**Never Moq in new code** — see SponsorLink fallout below.
DI lifetime rules for how these fakes register (Singleton / Scoped / Transient and captive-dependency traps) are owned by [Chapter 01 — foundations](./01-foundations.md); this chapter only covers the test-side mechanics.

### Moq — SponsorLink fallout
In Aug 2023, **Moq 4.20.0** shipped `SponsorLink`, a closed-source analyzer that read the developer's git email, hashed it (SHA-256, not salted), and phoned home. It was removed three days later in **4.20.2 (Aug 9, 2023)** under community pressure ("Remove SponsorLink since it breaks MacOS restore", PR #1375), and Moq has not re-added it through the latest **4.20.72 (Sep 2024)**. Trust is gone regardless. Many orgs banned Moq outright or pinned to 4.18.4.

**Don't**
- Don't take new dependencies on Moq in 2025+ code.
- If you must use Moq, pin tightly and audit each bump.

### NSubstitute
Cleaner syntax (no `.Object`), no lambda trees for setups, BSD licensed. Stable on **5.x**, with **6.0 in pre-release** (Mar 2025) and first-class MTP support.

```csharp
var clock = Substitute.For<TimeProvider>();
clock.GetUtcNow().Returns(new DateTimeOffset(2025, 1, 1, 0, 0, 0, TimeSpan.Zero));
```

**Do**
- Mock only the ports you own (`IEmailSender`, `IClock`). Don't mock `HttpClient`, `DbContext`, `ILogger<T>`.
- For `HttpClient`, use `MockHttp` or a delegating `HttpMessageHandler`. Better: Testcontainers + real server.
- For `ILogger<T>`, use `FakeLogger` from `Microsoft.Extensions.Diagnostics.Testing`.

### Hand-rolled fakes
For narrow ports (1–3 methods), a 20-line fake class is easier to read and debug than a mocking DSL. Keep them in a `Fakes/` folder in the test project.

### AutoFixture
Use for filling object graphs where the specific values don't matter. Customize aggressively:

```csharp
var fixture = new Fixture();
fixture.Customize<Money>(c => c.FromFactory(() => new Money(100, "USD")));
```

**Don't** use AutoFixture for domain objects with invariants — construct those explicitly via builders (§13).

Sources:
- NSubstitute — *Documentation* — https://nsubstitute.github.io/
- NSubstitute — *Project repository* — https://github.com/nsubstitute/NSubstitute
- devlooped/moq issue #1372 — *SponsorLink (4.20.0) discussion* — https://github.com/devlooped/moq/issues/1372
- devlooped/moq PR #1375 — *Remove SponsorLink since it breaks MacOS restore* — https://github.com/devlooped/moq/pull/1375
- learn.microsoft.com — *FakeLogger* — https://learn.microsoft.com/dotnet/core/extensions/logger-message-generator

---

## 5. `WebApplicationFactory` for ASP.NET Core

**Default integration harness for a single ASP.NET Core service: `WebApplicationFactory<Program>` + Testcontainers + Respawn.**
**Fallback: `Aspire.Hosting.Testing.DistributedApplicationTestingBuilder`** when the unit under test is the whole multi-service Aspire graph (see [Chapter 06 — cloud-native](./06-cloud-native.md) for the AppHost wiring this builder boots).

The standard way to test a real request pipeline in-process.
Fast (no network), realistic (full middleware), overrideable (real DI container).
The endpoints, ProblemDetails contracts, JWT bearer + Entra validation, and rate-limiting middleware exercised here are owned by [Chapter 02 — aspnetcore](./02-aspnetcore.md); assert against those contracts, do not redefine them.

### The spine

```csharp
public sealed class ApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.ConfigureTestServices(services =>
        {
            services.RemoveAll<IEmailSender>();
            services.AddSingleton<IEmailSender, FakeEmailSender>();
        });
    }
    public Task InitializeAsync() => Task.CompletedTask;
    public new Task DisposeAsync() => base.DisposeAsync().AsTask();
}
```

**Do**
- Use `ConfigureTestServices` (runs *after* `Startup`) — not `ConfigureServices`.
- Use `WithWebHostBuilder` to create a scoped factory *per test* when you need test-specific overrides; the base factory stays shared.
- Make `Program` accessible for tests: add `public partial class Program;` at the bottom of `Program.cs` (the minimal hosting model needs this).
- Use `factory.CreateClient(new WebApplicationFactoryClientOptions { AllowAutoRedirect = false })` for auth flows.

**Don't**
- Don't add `Microsoft.EntityFrameworkCore.InMemory` "just for tests." Use Testcontainers (§6).
- Don't new up `HttpClient` — always go through the factory so cookies/handlers/base address are correct.

### Auth in tests

Production auth (Entra ID, JWT) is painful in tests. Replace the scheme:

```csharp
services.AddAuthentication(defaults =>
{
    defaults.DefaultAuthenticateScheme = "Test";
    defaults.DefaultChallengeScheme = "Test";
}).AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { });
```

`TestAuthHandler` returns a `ClaimsPrincipal` built from request headers (`X-Test-User`, `X-Test-Roles`).
Each test sets headers to simulate a user — no tokens, no IdP round-trips.
The production scheme being replaced (JWT bearer with Microsoft Entra; validation of `iss`/`aud`/`scp`/`roles`/`azp`; policy-based authorization) is owned by [Chapter 02 — aspnetcore](./02-aspnetcore.md) — your test scheme must satisfy the same policies, not bypass them.

**Do**
- Keep the `TestAuthHandler` in the test project, not in production code.
- Assert on authorization policies (403 vs 401) with the same test harness — policies are part of your contract.

**Don't**
- Don't disable authentication in tests. Authz bugs hide there.
- Don't hardcode bearer tokens from a real IdP — they expire, leak, and slow CI.

### Background work: `IHostedService` / `BackgroundService`

`WebApplicationFactory` will start every registered hosted service. That's almost never what you want in a test process: timers fire, queues drain, outbox loops race against assertions.

**Do**
- **Disable hosted services by default in `Testing` env** and re-enable per test. The cleanest pattern is to remove all `IHostedService` registrations in `ConfigureTestServices` and add back only the one under test:
  ```csharp
  services.RemoveAll<IHostedService>();
  services.AddHostedService<OutboxProcessor>(); // only the SUT
  ```
- **Drive time with `FakeTimeProvider`.** Replace the registered `TimeProvider` with `FakeTimeProvider` and `Advance(...)` to step `PeriodicTimer`/`Task.Delay` loops deterministically — no `Thread.Sleep`, no flaky polling.
- **Coordinate with `TaskCompletionSource`** (constructed with `TaskCreationOptions.RunContinuationsAsynchronously`). Have the hosted service complete the TCS at a known checkpoint; `await` it in the test with a short timeout.
- **Assert graceful start/stop.** Call `host.StartAsync(ct)` then `StopAsync(ct)` and assert the service observed the cancellation token (didn't swallow it, didn't deadlock).

**Don't**
- Don't let the real `OutboxProcessor` run on a 5-second loop while your test asserts immediately and then exits. You'll get nondeterministic results and orphaned background tasks.
- Don't `Task.Delay` to "wait for the worker." Use the `FakeTimeProvider` + TCS combo.

### Decision table: which integration harness?

| You're testing | Use |
|---|---|
| A single ASP.NET Core service in-process; mock external HTTP, real DB via Testcontainers | `WebApplicationFactory<Program>` |
| Real infra (Postgres, Redis, RabbitMQ, Kafka, Azurite) for one service | Testcontainers (§6) |
| A multi-service Aspire app end-to-end, with the orchestrator wiring services together | **Aspire `DistributedApplicationTestingBuilder`** |
| Cross-service contract / API shape | Pact / Verify against captured responses |
| Live deployed environment | Smoke + critical journeys only (§1) |

`Aspire.Hosting.Testing.DistributedApplicationTestingBuilder` boots your `AppHost` project in-process for tests, hands you `HttpClient`s and connection strings keyed by resource name (`app.CreateHttpClient("api")`), and tears the whole app down at the end. By default it **disables the dashboard** and **randomizes proxied resource ports**, so multiple test runs can execute concurrently on the same machine without port collisions. It's the right answer for "I want to test the whole Aspire app as a closed box." For single-project in-memory testing, Microsoft explicitly points back to `WebApplicationFactory<T>`.

Sources:
- learn.microsoft.com — *Integration tests in ASP.NET Core* — https://learn.microsoft.com/aspnet/core/test/integration-tests
- learn.microsoft.com — *Implement background tasks in microservices with `IHostedService`* — https://learn.microsoft.com/dotnet/core/extensions/timer-service
- learn.microsoft.com — *.NET Aspire testing overview* — https://learn.microsoft.com/dotnet/aspire/testing/overview
- learn.microsoft.com — *Write your first .NET Aspire test* — https://learn.microsoft.com/dotnet/aspire/testing/write-your-first-test
- learn.microsoft.com — *Manage the app host in tests* — https://learn.microsoft.com/dotnet/aspire/testing/manage-app-host
- learn.microsoft.com — *FakeTimeProvider* — https://learn.microsoft.com/dotnet/api/microsoft.extensions.time.testing.faketimeprovider
- Andrew Lock — *Customising configuration sources in `WebApplicationFactory`* — https://andrewlock.net/converting-integration-tests-to-net-core-3/

---

## 6. Testcontainers

**Default: Testcontainers for anything with a driver — Postgres, SQL Server, Redis, RabbitMQ, Kafka, Azurite, MongoDB, Cosmos emulator.**
**Fallback: a provider-supplied local emulator** (Azurite, Cosmos DB Emulator, LocalStack) when no container image exists.
Docker is a test dependency; accept it.
The EF Core modeling, migration bundles, and provider-specific behavior these tests exercise are owned by [Chapter 03 — data](./03-data.md) — keep schema decisions there, not in the test fixture.

### Module packages

Prefer the **module packages** (`Testcontainers.PostgreSql`, `Testcontainers.MsSql`, `Testcontainers.Redis`, `Testcontainers.RabbitMq`, `Testcontainers.Kafka`, `Testcontainers.Azurite`, `Testcontainers.MongoDb`, …) over hand-rolled `ContainerBuilder` config. They ship sane defaults (image tags, wait strategies, connection-string accessors) and stay tracking upstream.

For xUnit v3 specifically, **`Testcontainers.XunitV3`** removes most fixture boilerplate by providing base classes that wire `IAsyncLifetime` for you.

### Lifetime

```csharp
public sealed class PostgresFixture : IAsyncLifetime
{
    public PostgreSqlContainer Container { get; } = new PostgreSqlBuilder()
        .WithImage("postgres:17.2-alpine")
        .Build();

    public Task InitializeAsync() => Container.StartAsync();
    public Task DisposeAsync()   => Container.DisposeAsync().AsTask();
}
```

### Sharing fixtures across tests (xUnit v3)

xUnit has **no `[CollectionFixture]` attribute**. Two correct patterns:

**One class shares the container** — `IClassFixture<T>`:

```csharp
public sealed class OrderRepoTests : IClassFixture<PostgresFixture>
{
    private readonly PostgresFixture _pg;
    public OrderRepoTests(PostgresFixture pg) => _pg = pg;
}
```

**Many classes share the container** — `ICollectionFixture<T>` on a definition + `[Collection]` on every participating class:

```csharp
[CollectionDefinition(nameof(PostgresCollection))]
public sealed class PostgresCollection : ICollectionFixture<PostgresFixture> { }

[Collection(nameof(PostgresCollection))]
public sealed class OrderRepoTests { /* ctor receives PostgresFixture */ }

[Collection(nameof(PostgresCollection))]
public sealed class CustomerRepoTests { /* same shared container */ }
```

**Do**
- Use `IClassFixture<T>` for one class, `ICollectionFixture<T>` + `[CollectionDefinition]`/`[Collection]` to share across many. Container startup is the slow part; reuse it.
- Pin image tags (`postgres:17.2-alpine`), not `latest`. Your CI should be reproducible.
- Enable **reuse** in local dev (`.WithReuse(true)`) — subsequent runs attach to the existing container. CI should *not* reuse.

**Don't**
- Don't write `[CollectionFixture]` — it doesn't exist. The fixture marker goes on the *definition* class via `ICollectionFixture<T>`; tests opt in via `[Collection("...")]`.
- Don't spin up a fresh container per test class. Your CI bill will show it.
- Don't use SQLite as a stand-in for SQL Server/Postgres. Different SQL dialects, different constraint semantics, different transaction isolation. You will find bugs in prod you didn't find in tests.
- Don't run `dotnet ef database update` in each test — snapshot the schema once, restore via backup or template DB.

### Per-test reset: pick the strategy that matches your harness

These are not interchangeable. Pick by *who owns the connection*.

| Test shape | Reset strategy |
|---|---|
| Narrow data-access test that owns the `DbConnection`/`DbContext` and the transaction | `BEGIN` in setup, `ROLLBACK` in teardown. Fast, perfectly isolated. |
| Full-stack `WebApplicationFactory` test that goes through HTTP — each request opens its own DI scope and connection from the pool | **Respawn** (or restore-from-template-DB). Truncate/delete known tables between tests. Transaction rollback **does not work** here because the test's transaction can't envelop separate connections opened inside the host. |
| Background-worker integration test where the SUT opens its own connections | Respawn / template-DB. Same reason. |

**Do**
- Use Respawn (or a restore-from-template-DB / `pg_restore` of a seeded snapshot) as the default for `WebApplicationFactory` and worker integration tests.
- Reserve transaction-rollback isolation for tight repository tests where the test code owns the connection and transaction end-to-end.

**Don't**
- Don't wrap a `WebApplicationFactory` request in `TransactionScope` and expect it to roll back the changes the API made — you'll get partial isolation and false greens.

### Performance vs SQLite

SQLite in-memory: ~5 ms setup, fake SQL. Postgres container: ~2 s startup, real SQL. Amortized over a full suite (container reused, Respawn for isolation), Postgres adds a few seconds of total CI time and eliminates an entire category of bugs. Worth it.

Sources:
- xUnit.net — *Shared context between tests* — https://xunit.net/docs/shared-context
- Testcontainers for .NET — *Modules* (MsSql, PostgreSql, Redis, …) — https://dotnet.testcontainers.org/modules/
- Testcontainers for .NET — *Testcontainers.XunitV3* — https://dotnet.testcontainers.org/test_frameworks/xunit_v3/
- Testcontainers for .NET — *Reuse containers* — https://dotnet.testcontainers.org/api/resource_reuse/
- testcontainers/testcontainers-dotnet — *Repository* — https://github.com/testcontainers/testcontainers-dotnet
- Respawn — https://github.com/jbogard/Respawn
- learn.microsoft.com — *Integration tests in ASP.NET Core* — https://learn.microsoft.com/aspnet/core/test/integration-tests

---

## 7. Snapshot / approval testing with Verify

`Verify.Xunit` (or `Verify.XunitV3`, `Verify.MSTest`, `Verify.NUnit`, `Verify.TUnit`) captures output to a `.verified.txt` file; each test run diffs against it.

**When to use**
- HTTP response bodies (`Verify.Http`).
- Generated SQL (`Verify.EntityFramework` — captures `DbCommand.CommandText`).
- Structured logs, OpenAPI docs, migration scripts, code generators.
- Anything where "does the output match?" is easier to eyeball than to assert field-by-field.

**Workflow**
1. First run creates `<TestName>.received.txt`. Review it.
2. Approve: rename to `.verified.txt`. Commit it.
3. Diffs on re-run open your diff tool automatically (Beyond Compare, Kaleidoscope, VS Code).

**Do**
- Scrub non-deterministic bits (`Guid`, `DateTime`, connection strings) via `VerifierSettings.ScrubInlineGuids()`.
- Commit `.verified.*` files. Gitignore `.received.*`.
- Configure a diff tool in `.editorconfig` or `ModuleInitializer`.

**Don't**
- Don't snapshot huge blobs you can't eyeball. Snapshots you can't read become rubber stamps.
- Don't snapshot prod secrets or PII. Use scrubbers.

Sources:
- VerifyTests/Verify — *Project README* — https://github.com/VerifyTests/Verify
- VerifyTests/Verify — *Scrubbers* — https://github.com/VerifyTests/Verify/blob/main/docs/scrubbers.md
- VerifyTests/Verify.EntityFramework — *Project README* — https://github.com/VerifyTests/Verify.EntityFramework
- VerifyTests/Verify.Http — *Project README* — https://github.com/VerifyTests/Verify.Http

---

## 8. Mutation testing with Stryker.NET

Mutation testing changes your source ("mutants") and reruns tests. If tests still pass, the mutant *survived* — your tests didn't actually cover that behavior.

**Do**
- Run **nightly** or on merge-to-main. Full runs take 10–60+ minutes.
- Target a **mutation score ≥ 70%** on domain/core assemblies. Ignore UI/glue.
- Read *escaped* mutants — they reveal weak assertions (`Assert.NotNull` where you meant `Assert.Equal(5, x)`), missing edge cases, dead branches.

**Don't**
- Don't run Stryker on every PR. It's too slow and produces noisy diffs.
- Don't chase 100%. Equivalent mutants (semantically identical) exist; diminishing returns after ~80%.

Sources:
- Stryker.NET — *Documentation* — https://stryker-mutator.io/docs/stryker-net/introduction/
- stryker-mutator/stryker-net — *Repository* — https://github.com/stryker-mutator/stryker-net

---

## 9. Architecture tests

Encode layering rules as tests. Cheap insurance against drift.

- **NetArchTest.Rules** — LINQ-style: `Types.InAssembly(...).That().ResideInNamespace("Domain").ShouldNot().HaveDependencyOn("Infrastructure")`.
- **ArchUnitNET** — richer model, port of Java ArchUnit.

**Do**
- Enforce: Domain has no dependencies on Infrastructure/Web. Controllers don't reference `DbContext` directly (go through MediatR/handlers). `public` types in `Internal` namespaces are banned.
- Put these in a single `ArchitectureTests` project that runs fast.

**Don't**
- Don't encode coding style here (use analyzers / `.editorconfig`).
- Don't make rules so strict every new feature requires updating them.

Sources:
- BenMorris/NetArchTest — *Repository* — https://github.com/BenMorris/NetArchTest
- TNG/ArchUnitNET — *Repository* — https://github.com/TNG/ArchUnitNET

---

## 10. Property-based tests

For invariants, not examples. Good for parsers, serializers, math, state machines, idempotency.

- **FsCheck** (F# origins, works great in C#) — mature.
- **CsCheck** — C#-first, actively developed, simple API, built-in shrinking.

Concrete example with **FsCheck.Xunit** (the integration package), surfacing the seed so CI failures are reproducible:

```csharp
using FsCheck;
using FsCheck.Xunit;

public class CodecProperties
{
    // Replay seeds from a previous failure: [Property(Replay = "(12345,67890)")]
    [Property(MaxTest = 500, QuietOnSuccess = true)]
    public Property Roundtrip(NonNull<string> input)
    {
        var s = input.Get;
        return (Decode(Encode(s)) == s).ToProperty();
    }
}
```

When FsCheck shrinks to a counterexample it prints the seed (e.g. `Replay = "(12345,67890)"`); paste that into `[Property(Replay = "...")]` to reproduce locally and in CI.

**Do**
- Seed deterministically in CI (print the seed on failure, paste it back via `Replay` in the next run).
- Use shrinking — the minimal failing input is the actual bug.

**Don't**
- Don't use property tests as a replacement for a well-chosen example test. Use both.

Sources:
- FsCheck — *Running tests* (xUnit integration, `Replay`) — https://fscheck.github.io/FsCheck/RunningTests.html
- fscheck/FsCheck — *Repository* — https://github.com/fscheck/FsCheck
- AnthonyLloyd/CsCheck — *Repository* — https://github.com/AnthonyLloyd/CsCheck

---

## 11. Performance tests in CI

**BenchmarkDotNet is not a test framework.** It's a measurement tool. Running it inside `dotnet test` in a tight loop produces garbage data.

**Do**
- Run BenchmarkDotNet in a **nightly** job on a dedicated runner (consistent hardware, no noisy neighbors).
- In PRs, run **allocation-only** assertions via `Microsoft.Extensions.Diagnostics.Testing.MetricCollector` or `Assert.Allocates`-style helpers. Allocation counts are deterministic and cheap.
- Export BDN results (`--exporters json`) and diff against a baseline committed to the repo.

**Don't**
- Don't gate PRs on wall-clock benchmarks in shared CI. Variance will kill you.
- Don't benchmark in Debug. BDN refuses anyway; heed it.

### Benchmarking against real infra (Testcontainers)

There is no official BDN ↔ Testcontainers integration. The standard idiom: start the container in `[GlobalSetup]` and dispose in `[GlobalCleanup]` so startup cost is excluded from measurements; expose the resolved connection string to `[Benchmark]` methods.

```csharp
[MemoryDiagnoser]
public class RepoBenchmarks
{
    private PostgreSqlContainer _pg = null!;
    private NpgsqlConnection _conn = null!;

    [GlobalSetup]
    public async Task Setup()
    {
        _pg = new PostgreSqlBuilder().WithImage("postgres:17.2-alpine").Build();
        await _pg.StartAsync();
        _conn = new NpgsqlConnection(_pg.GetConnectionString());
        await _conn.OpenAsync();
    }

    [GlobalCleanup]
    public async Task Cleanup()
    {
        await _conn.DisposeAsync();
        await _pg.DisposeAsync();
    }

    [Benchmark]
    public Task<int> Roundtrip() => _conn.ExecuteScalarAsync<int>("select 1");
}
```

Be aware: container startup is excluded, but Docker-host state (cache, page cache, neighboring containers) introduces variance the BDN warmup phase cannot iron out. Pin the image tag, reserve a dedicated runner, and treat results as a *trend* against a committed baseline — not as PR-gate truth.

Wall-clock benchmark methodology, allocation discipline, and BDN regression-gate policy live in [Chapter 05 — performance](./05-performance.md); this chapter only covers wiring BDN into the same Testcontainers fixtures the integration tier uses.

Sources:
- BenchmarkDotNet — *Setup and Cleanup* (`[GlobalSetup]`/`[GlobalCleanup]` lifecycle) — https://benchmarkdotnet.org/articles/features/setup-and-cleanup.html
- dotnet/BenchmarkDotNet — *Repository* — https://github.com/dotnet/BenchmarkDotNet
- Testcontainers for .NET — https://dotnet.testcontainers.org/

---

## 12. Coverage

Use **coverlet.collector** (MTP has first-party coverage too) + **ReportGenerator** for HTML.

**Do**
- Produce `cobertura` XML and upload to CI artifacts.
- Track coverage as a trend, not a gate. Gate on **no unexplained decrease** (e.g., `-5% fails`).
- Exclude generated code, migrations, `Program.cs` minimal hosting.

**Don't**
- Don't set a hard 80% gate. Teams write trivial tests to game it.
- Don't trust branch coverage on `async` methods — state machines distort numbers. Mutation testing is a better signal (§8).

Sources:
- coverlet-coverage/coverlet — *Repository* — https://github.com/coverlet-coverage/coverlet
- danielpalme/ReportGenerator — *Repository* — https://github.com/danielpalme/ReportGenerator
- learn.microsoft.com — *Use code coverage for unit testing* — https://learn.microsoft.com/dotnet/core/testing/unit-testing-code-coverage

---

## 13. Test data builders

Two patterns, both good:

- **Object Mother**: named factories for common personas. `Customers.Active()`, `Orders.Shipped()`.
- **Test Data Builder**: fluent, `new CustomerBuilder().WithEmail("x@y").Build()`.

**Do**
- Put builders in a shared test-support project (e.g., `Acme.Orders.Tests.Support`).
- Make builders produce **valid by default** objects. Tests only override what they care about.
- Seed deterministically (`Bogus.Faker` with a fixed seed).

**Don't**
- Don't use AutoFixture for domain aggregates with invariants. It sets properties via reflection and bypasses constructors.

Sources:
- AutoFixture/AutoFixture — *Repository* — https://github.com/AutoFixture/AutoFixture
- bchavez/Bogus — *Repository* — https://github.com/bchavez/Bogus

---

## 14. Time, randomness, I/O

### Time — use `TimeProvider` (.NET 8+)

`System.TimeProvider` is the canonical abstraction. Inject it; in tests use `Microsoft.Extensions.Time.Testing.FakeTimeProvider`:

```csharp
var time = new FakeTimeProvider(new DateTimeOffset(2025, 1, 1, 0, 0, 0, TimeSpan.Zero));
time.Advance(TimeSpan.FromMinutes(5));
```

**Do**
- Replace any `IClock` / `ISystemClock` / `INow` abstractions with `TimeProvider` in new code.
- Use `FakeTimeProvider.Advance` to drive `Timer`, `PeriodicTimer`, and `Task.Delay` deterministically.

**Don't**
- Don't call `DateTime.UtcNow` in production code. Ever. Inject `TimeProvider`.

### Randomness
Inject `Random` (or a seed). `new Random(seed)` is deterministic. For crypto, `RandomNumberGenerator` — but wrap behind a port you can fake.

### File system
Use `System.IO.Abstractions` + `MockFileSystem` for unit tests. For integration, use a `TempFolder` helper that cleans itself up.

Sources:
- learn.microsoft.com — *TimeProvider class* — https://learn.microsoft.com/dotnet/api/system.timeprovider
- learn.microsoft.com — *FakeTimeProvider* — https://learn.microsoft.com/dotnet/api/microsoft.extensions.time.testing.faketimeprovider
- TestableIO/System.IO.Abstractions — *Repository* — https://github.com/TestableIO/System.IO.Abstractions

---

## 15. Async test pitfalls

**Do**
- Return `Task` (or `ValueTask`) from every async test. **xUnit v3 does not support `async void` test methods** — they fail fast at discovery/runtime. (xUnit v2 quietly allowed it; v3 removed that.)
- Use `await Assert.ThrowsAsync<T>(() => sut.DoAsync())` — not `.Result`, not `.Wait()`.
- Use `TaskCompletionSource` (constructed with `TaskCreationOptions.RunContinuationsAsynchronously`) to deterministically coordinate when you need to assert "X happened before Y".
- Test **cancellation**: pass `TestContext.Current.CancellationToken` (xUnit v3) or a short `CancellationTokenSource(TimeSpan.FromSeconds(2))` and assert `OperationCanceledException`.

**Don't**
- Don't write `public async void Test()` — in xUnit v3 it's an error, not a warning. Use `async Task`.
- Don't `Thread.Sleep` in tests. Replace with `FakeTimeProvider.Advance` or a `TaskCompletionSource`.
- Don't `.GetAwaiter().GetResult()` inside a test — you will deadlock on some sync contexts (WASM, legacy hosts).
- Don't rely on "it's fast enough" timing. Flakes compound.

Sources:
- xUnit.net — *Migrating from v2 to v3* (async void no longer supported) — https://xunit.net/docs/getting-started/v3/migration
- xUnit.net — *What's New in v3* — https://xunit.net/docs/getting-started/v3/whats-new

---

## 16. CI discipline

**Do**
- **Fail fast**: `dotnet test --blame-hang-timeout 2m` and stop the job on first failure in PR builds (not nightly).
- **Parallel by default**: xUnit runs test classes in parallel. Mark genuinely non-parallel tests with `[CollectionDefinition(DisableParallelization = true)]`. Don't turn parallelism off globally.
- **Isolate flakes**: `[Trait("Category", "Flaky")]` and run them in a separate job that can be retried. Open an issue per flake; budget time to fix.
- **Categorize**: `[Trait("Category", "Integration")]`, filter with `--filter "Category=Unit"` on PRs; full matrix nightly.
- **Deterministic seeds** for property-based / fuzz tests. Print seed on failure, accept seed as env var.
- Publish **TRX + coverage + BDN artifacts** for every run.
- Use **retry** *only* for known-flaky integration layers (network blips); never for logic tests.

**Don't**
- Don't `Thread.Sleep`-then-retry in CI scripts. Fix the test.
- Don't run E2E on every PR if each run costs minutes. Tag and schedule.
- Don't mix test results across frameworks in one project (xUnit + NUnit in one csproj = pain).

Sources:
- xUnit.net — *Running tests in parallel* — https://xunit.net/docs/running-tests-in-parallel
- learn.microsoft.com — *Filter options for `dotnet test`* — https://learn.microsoft.com/dotnet/core/tools/dotnet-test#filter-tests

---

## TL;DR cheat sheet

- **xUnit v3 + MTP**, `OutputType=Exe`. Fallback: MSTest v3.
- **Shouldly** for new code; built-in `Assert` for zero deps; **AwesomeAssertions** if migrating off FA v8. Skip FA v8 unless you paid.
- **NSubstitute**, not Moq.
- **Testcontainers**, not SQLite-pretending-to-be-SQL-Server.
- **`WebApplicationFactory<Program>` + `TestAuthHandler`** for API tests; **`DistributedApplicationTestingBuilder`** for the whole Aspire graph.
- **Verify** for anything structured.
- **`TimeProvider` + `FakeTimeProvider`** — delete your homegrown `IClock`.
- **Stryker nightly**, coverage as trend, mutation score as signal.
- **Benchmarks nightly** on dedicated hardware; **allocation assertions** in PRs.
- **Property tests with deterministic seeds** (FsCheck or CsCheck).
- **Flakes are bugs**.

### Where to look next

- DI lifetimes for fakes / fixtures (Singleton vs Scoped vs Transient, captive deps) → [Chapter 01 — foundations](./01-foundations.md).
- The endpoint surface, JWT/Entra scheme, and ProblemDetails contracts that `WebApplicationFactory` exercises → [Chapter 02 — aspnetcore](./02-aspnetcore.md).
- EF Core schema, migrations, and the outbox pattern that integration fixtures target → [Chapter 03 — data](./03-data.md).
- Aspire AppHost composition that `DistributedApplicationTestingBuilder` boots, and `ServiceDefaults` it inherits → [Chapter 06 — cloud-native](./06-cloud-native.md).

---

## Sources

### Authoritative — frameworks
- xUnit.net — *What's New in v3* — https://xunit.net/docs/getting-started/v3/whats-new
- xUnit.net — *Microsoft.Testing.Platform support* — https://xunit.net/docs/getting-started/v3/microsoft-testing-platform
- xUnit.net — *Migrating from v2 to v3* — https://xunit.net/docs/getting-started/v3/migration
- xUnit.net — Release notes (v3 1.0.0 GA, 2024-12-16) — https://xunit.net/releases/v3/1.0.0
- xunit/xunit — *Repository* — https://github.com/xunit/xunit
- TUnit — *Project README* — https://github.com/thomhurst/TUnit
- Microsoft .NET Blog — *Microsoft.Testing.Platform: Now Supported by All Major .NET Test Frameworks* — https://devblogs.microsoft.com/dotnet/mtp-adoption-frameworks/
- learn.microsoft.com — *Microsoft.Testing.Platform overview* — https://learn.microsoft.com/dotnet/core/testing/microsoft-testing-platform-intro

### Authoritative — assertions, mocks, fakes
- Shouldly — *Documentation* — https://docs.shouldly.org/
- shouldly/shouldly — *Repository* — https://github.com/shouldly/shouldly
- AwesomeAssertions — *Repository* — https://github.com/AwesomeAssertions/AwesomeAssertions
- Verify — *Repository* — https://github.com/VerifyTests/Verify
- NSubstitute — *Documentation* — https://nsubstitute.github.io/
- nsubstitute/NSubstitute — *Repository* — https://github.com/nsubstitute/NSubstitute
- Xceed — *Fluent Assertions FAQ* (v8 licensing) — https://xceed.com/fluent-assertions-faq/
- devlooped/moq PR #1375 — *Remove SponsorLink since it breaks MacOS restore* — https://github.com/devlooped/moq/pull/1375
- devlooped/moq issue #1384 — *SponsorLink is now OSS too and no longer bundled* — https://github.com/devlooped/moq/issues/1384

### Authoritative — integration & infrastructure
- learn.microsoft.com — *Integration tests in ASP.NET Core* — https://learn.microsoft.com/aspnet/core/test/integration-tests
- Testcontainers for .NET — *Documentation* — https://dotnet.testcontainers.org/
- testcontainers/testcontainers-dotnet — *Repository* — https://github.com/testcontainers/testcontainers-dotnet
- Testcontainers for .NET — *Testcontainers.XunitV3* — https://dotnet.testcontainers.org/test_frameworks/xunit_v3/
- learn.microsoft.com — *.NET Aspire testing overview* — https://learn.microsoft.com/dotnet/aspire/testing/overview
- learn.microsoft.com — *Write your first .NET Aspire test* — https://learn.microsoft.com/dotnet/aspire/testing/write-your-first-test
- jbogard/Respawn — *Repository* — https://github.com/jbogard/Respawn

### Authoritative — time, properties, mutation, coverage
- learn.microsoft.com — *TimeProvider class* — https://learn.microsoft.com/dotnet/api/system.timeprovider
- learn.microsoft.com — *FakeTimeProvider* — https://learn.microsoft.com/dotnet/api/microsoft.extensions.time.testing.faketimeprovider
- fscheck/FsCheck — *Repository* — https://github.com/fscheck/FsCheck
- AnthonyLloyd/CsCheck — *Repository* — https://github.com/AnthonyLloyd/CsCheck
- Stryker.NET — *Documentation* — https://stryker-mutator.io/docs/stryker-net/introduction/
- stryker-mutator/stryker-net — *Repository* — https://github.com/stryker-mutator/stryker-net
- coverlet-coverage/coverlet — *Repository* — https://github.com/coverlet-coverage/coverlet
- danielpalme/ReportGenerator — *Repository* — https://github.com/danielpalme/ReportGenerator

### Community
- Khalid Abuhakmeh — testing & ASP.NET Core posts — https://khalidabuhakmeh.com/
- Andrew Lock — *.NET Escapades* (integration tests, WebApplicationFactory, Testcontainers) — https://andrewlock.net/
- Jimmy Bogard — *Los Techies* / blog (integration testing, Respawn author) — https://www.jimmybogard.com/
- Steve Smith (Ardalis) — testing & clean architecture — https://ardalis.com/
- Bradley Wells — xUnit v3 / MTP migration guides — https://bradwells.com/ (and https://wellsb.com)
- Meziantou — *Meziantou's blog* (deep .NET testing, analyzers, CI) — https://www.meziantou.net/
- Aaron Stannard — *.NET OSS Projects: Better to Re-license or Die?* (FA v8 context) — https://aaronstannard.com/relicense-or-die/
- Conrad Akunga — *There Be Dragons — FluentAssertions 8 New Licensing* — https://www.conradakunga.com/blog/there-be-dragons-fluentassertions-8-new-licensing/
