# Testing .NET 10 Apps — Opinionated Best Practices

This is a dense, opinionated guide. It assumes `net10.0`, `Microsoft.Testing.Platform` (MTP), and xUnit v3 GA (shipped Dec 16, 2024). Do/Don't notation is intentional. If you disagree, write your own doc.

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

---

## 2. Frameworks: xUnit v3, MSTest, NUnit

**Use xUnit v3.** It is GA (1.0.0, Dec 2024) and is the canonical choice for new .NET code. MSTest and NUnit are fine if you already use them — don't migrate for its own sake.

### xUnit v3: what actually changed from v2

- **Tests run in a dedicated process.** v2 ran tests in-process with the runner (reflection-only loading of your assembly). v3 compiles each test project into an **executable** that the runner launches. Consequences:
  - `Main` entry point exists; you can debug a test project with F5.
  - Static state no longer leaks between unrelated test runs on the same machine.
  - `AppDomain` tricks, `ILogger` capture via shared statics, and module initializers behave like a real app.
- **Async lifetime for fixtures.** `IAsyncLifetime` is first-class for `IClassFixture<T>` and `ICollectionFixture<T>` — stop doing sync-over-async in `ctor`/`Dispose`.
- **`Assert.Equivalent(expected, actual)`** — deep structural equality built-in. Kills a big reason to reach for FluentAssertions.
- **`[Theory]` data via `TheoryData<>`** is strongly typed; prefer it over `object[][]`.
- **Cancellation tokens** are available to tests via `TestContext.Current.CancellationToken`. Wire long-running tests to honor it.
- **First-class `Microsoft.Testing.Platform` support.** `dotnet test` works; so does running the test `.exe` directly.

### Microsoft.Testing.Platform (MTP) replaces VSTest

MTP is the modern runner. By early 2025 all major frameworks (xUnit v3, MSTest, NUnit, TUnit) support it. It is not a new test framework — it's a host.

**Do**
- Enable MTP in csproj: `<EnableMSTestRunner>true</EnableMSTestRunner>` / `<UseMicrosoftTestingPlatformRunner>true</UseMicrosoftTestingPlatformRunner>`, and `<OutputType>Exe</OutputType>`.
- Use MTP's trx/coverage/retry extensions instead of VSTest logger packages where available.

**Don't**
- Don't mix `Microsoft.NET.Test.Sdk` (VSTest) and MTP in the same project. Pick one per project.
- Don't block on VSTest-only tooling — MTP is the direction of travel.

### MSTest / NUnit

- MSTest v3 has MTP integration, dynamic data sources, and is perfectly reasonable.
- NUnit is fine, but `[SetUp]`/`[TearDown]` per-test ceremony tempts shared mutable state. xUnit's constructor-per-test model is better for isolation.

---

## 3. Assertions

**Recommendation: built-in `Assert` (xUnit v3) + `Shouldly` for readability + `Verify` for structural/snapshot.**

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

---

## 4. Mocks and fakes

**Prefer NSubstitute. Avoid Moq.**

### Moq — SponsorLink fallout
In Aug 2023, Moq 4.20 shipped `SponsorLink`, a closed-source analyzer that read the developer's git email, hashed it (SHA-256, not salted), and phoned home. It was removed under pressure, but trust is gone. Many orgs banned Moq outright or pinned to 4.18.4.

**Don't**
- Don't take new dependencies on Moq in 2025+ code.
- If you must use Moq, pin tightly and audit each bump.

### NSubstitute
Cleaner syntax (no `.Object`), no lambda trees for setups, MIT licensed.

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

---

## 5. `WebApplicationFactory` for ASP.NET Core

The standard way to test a real request pipeline in-process. Fast (no network), realistic (full middleware), overrideable (real DI container).

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

`TestAuthHandler` returns a `ClaimsPrincipal` built from request headers (`X-Test-User`, `X-Test-Roles`). Each test sets headers to simulate a user — no tokens, no IdP round-trips.

**Do**
- Keep the `TestAuthHandler` in the test project, not in production code.
- Assert on authorization policies (403 vs 401) with the same test harness — policies are part of your contract.

**Don't**
- Don't disable authentication in tests. Authz bugs hide there.
- Don't hardcode bearer tokens from a real IdP — they expire, leak, and slow CI.

---

## 6. Testcontainers

**Use Testcontainers for anything with a driver: Postgres, SQL Server, Redis, RabbitMQ, Kafka, Azurite, MongoDB.** Docker is a test dependency; accept it.

### Lifetime

```csharp
public sealed class PostgresFixture : IAsyncLifetime
{
    public PostgreSqlContainer Container { get; } = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine")
        .Build();

    public Task InitializeAsync() => Container.StartAsync();
    public Task DisposeAsync()   => Container.DisposeAsync().AsTask();
}
```

**Do**
- Share containers across tests with `[CollectionFixture]` / `[ICollectionFixture<T>]`. Container startup is the slow part; reuse it.
- Reset state per test via **transactions** (`Respawn` or `BEGIN/ROLLBACK`) — far faster than re-creating the container.
- Pin image tags (`postgres:17.2-alpine`), not `latest`. Your CI should be reproducible.
- Enable **reuse** in local dev (`.WithReuse(true)`) — subsequent runs attach to the existing container. CI should *not* reuse.

**Don't**
- Don't spin up a fresh container per test class. Your CI bill will show it.
- Don't use SQLite as a stand-in for SQL Server/Postgres. Different SQL dialects, different constraint semantics, different transaction isolation. You will find bugs in prod you didn't find in tests.
- Don't run `dotnet ef database update` in each test — snapshot the schema once, restore via backup or template DB.

### Performance vs SQLite

SQLite in-memory: ~5 ms setup, fake SQL. Postgres container: ~2 s startup, real SQL. Amortized over a full suite (container reused, transactions for isolation), Postgres adds a few seconds of total CI time and eliminates an entire category of bugs. Worth it.

---

## 7. Snapshot / approval testing with Verify

`Verify.Xunit` (or `Verify.MSTest`, `Verify.NUnit`) captures output to a `.verified.txt` file; each test run diffs against it.

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

---

## 10. Property-based tests

For invariants, not examples. Good for parsers, serializers, math, state machines, idempotency.

- **FsCheck** (F# origins, works great in C#) — mature.
- **CsCheck** — C#-first, actively developed, simple API, built-in shrinking.

```csharp
[Fact] public void Roundtrip()
    => Gen.String.Sample(s => Decode(Encode(s)).ShouldBe(s));
```

**Do**
- Seed deterministically in CI (print the seed on failure, reuse it in the next run).
- Use shrinking — the minimal failing input is the actual bug.

**Don't**
- Don't use property tests as a replacement for a well-chosen example test. Use both.

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

---

## 15. Async test pitfalls

**Do**
- Return `Task`. Never `async void` in tests — xUnit supports it but exceptions escape to the runtime.
- Use `await Assert.ThrowsAsync<T>(() => sut.DoAsync())` — not `.Result`, not `.Wait()`.
- Use `TaskCompletionSource` (with `RunContinuationsAsynchronously`) to deterministically coordinate when you need to assert "X happened before Y".
- Test **cancellation**: pass `TestContext.Current.CancellationToken` (xUnit v3) or a short `CancellationTokenSource(TimeSpan.FromSeconds(2))` and assert `OperationCanceledException`.

**Don't**
- Don't `Thread.Sleep` in tests. Replace with `FakeTimeProvider.Advance` or a `TaskCompletionSource`.
- Don't `.GetAwaiter().GetResult()` inside a test — you will deadlock on some sync contexts (WASM, legacy hosts).
- Don't rely on "it's fast enough" timing. Flakes compound.

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

---

## TL;DR cheat sheet

- **xUnit v3 + MTP**, `OutputType=Exe`.
- **Shouldly or built-in `Assert.Equivalent`**. Skip FA v8 unless you paid.
- **NSubstitute**, not Moq.
- **Testcontainers**, not SQLite-pretending-to-be-SQL-Server.
- **`WebApplicationFactory` + `TestAuthHandler`** for API tests.
- **Verify** for anything structured.
- **`TimeProvider` + `FakeTimeProvider`** — delete your homegrown `IClock`.
- **Stryker nightly**, coverage as trend, mutation score as signal.
- **Benchmarks nightly** on dedicated hardware; **allocation assertions** in PRs.
- **Property tests with deterministic seeds**.
- **Flakes are bugs**.

---

## Sources

### Authoritative
- xUnit.net — *What's New in v3* — https://xunit.net/docs/getting-started/v3/whats-new
- xUnit.net — Release notes (v3 1.0.0 GA, 2024-12-16) — https://xunit.net/releases/v3/1.0.0
- Microsoft .NET Blog — *Microsoft.Testing.Platform: Now Supported by All Major .NET Test Frameworks* — https://devblogs.microsoft.com/dotnet/mtp-adoption-frameworks/
- learn.microsoft.com — *Microsoft.Testing.Platform overview* — https://learn.microsoft.com/dotnet/core/testing/microsoft-testing-platform-intro
- learn.microsoft.com — *Integration tests in ASP.NET Core* — https://learn.microsoft.com/aspnet/core/test/integration-tests
- learn.microsoft.com — *TimeProvider class* — https://learn.microsoft.com/dotnet/api/system.timeprovider
- learn.microsoft.com — *FakeTimeProvider* — https://learn.microsoft.com/dotnet/api/microsoft.extensions.time.testing.faketimeprovider
- Testcontainers for .NET — https://dotnet.testcontainers.org/
- Verify — https://github.com/VerifyTests/Verify
- Stryker.NET — https://stryker-mutator.io/docs/stryker-net/introduction/
- NSubstitute — https://nsubstitute.github.io/
- Shouldly — https://docs.shouldly.org/
- Xceed — *Fluent Assertions FAQ* (v8 licensing) — https://xceed.com/fluent-assertions-faq/
- devlooped/moq issue #1384 — *SponsorLink is now OSS too and no longer bundled* — https://github.com/devlooped/moq/issues/1384

### Community
- Khalid Abuhakmeh — testing & ASP.NET Core posts — https://khalidabuhakmeh.com/
- Andrew Lock — *.NET Escapades* (integration tests, WebApplicationFactory, Testcontainers) — https://andrewlock.net/
- Nick Chapsas — YouTube deep-dives on xUnit v3, NSubstitute, Verify — https://www.youtube.com/@nickchapsas
- Jimmy Bogard — *Los Techies* / blog (integration testing, Respawn author) — https://www.jimmybogard.com/
- Steve Smith (Ardalis) — testing & clean architecture — https://ardalis.com/
- Bradley Wells — xUnit v3 / MTP migration guides — https://bradwells.com/ (and https://wellsb.com)
- Meziantou — *Meziantou's blog* (deep .NET testing, analyzers, CI) — https://www.meziantou.net/
- Aaron Stannard — *.NET OSS Projects: Better to Re-license or Die?* (FA v8 context) — https://aaronstannard.com/relicense-or-die/
- Conrad Akunga — *There Be Dragons — FluentAssertions 8 New Licensing* — https://www.conradakunga.com/blog/there-be-dragons-fluentassertions-8-new-licensing/
