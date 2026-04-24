# .NET 10 / C# 13 — Monorepo strategy, anti-patterns, and patterns

Opinionated companion to `foundations.md`, `aspnetcore.md`, `data.md`. This page
covers four things sibling docs intentionally skip: how to lay out a multi-team
.NET monorepo, the anti-patterns we reject in code review, the positive patterns
that should replace them, and a short GoF refresher framed for idiomatic .NET 10.

> Severity legend (used inline): 🔴 hard fail in review, 🟡 needs justification,
> 🟢 recommended.

---

## 1. .NET monorepo strategy

### 1.1 When a monorepo is the right call

A monorepo earns its keep when (a) you ship a coherent product whose pieces
release together, (b) cross-cutting refactors (e.g. an interface change in a
shared library) need to land atomically across services, or (c) you want one
toolchain, one analyzer ruleset, one CI definition. Polyrepo + a private NuGet
feed is the right answer when teams release on independent cadences, when the
shared code is genuinely a stable SDK, or when blast-radius isolation is more
valuable than atomic refactors.

The [ThoughtWorks Tech Radar entry on "monorepo for cross-team
collaboration"](https://www.thoughtworks.com/radar/techniques/monorepo-with-a-suitable-build-system)
sits in **Trial**, but with the explicit caveat that it requires a *suitable
build system* (graph-aware, incremental, with change-impact analysis). Without
it you get the worst of both worlds: a giant repo *and* full-rebuild CI.

Microsoft's [.NET Microservices Architecture
eBook](https://learn.microsoft.com/dotnet/architecture/microservices/) is
explicit: prefer a single repo per bounded context (or product) and *don't*
shard a single bounded context across repos just to mimic Netflix. Conversely,
the [.NET Application Architecture
guides](https://dotnet.microsoft.com/learn/dotnet/architecture-guides) treat
"monorepo with multiple deployable hosts" as the default for greenfield
ASP.NET Core systems.

Decision rule we use:

| Question | Monorepo | Polyrepo + private feed |
| --- | --- | --- |
| Teams release on the same train? | ✅ | ❌ |
| Shared code changes weekly? | ✅ | ❌ |
| Shared code changes quarterly with SemVer? | ❌ | ✅ |
| Need to enforce one analyzer ruleset? | ✅ | painful |
| Independent SLAs, independent on-call? | ❌ | ✅ |
| Open-source library you want consumers to fork? | ❌ | ✅ |

### 1.2 Repo layout

```
/
├── global.json                # SDK pin
├── nuget.config               # feeds + package source mapping
├── .editorconfig              # style + analyzer severities
├── Directory.Build.props      # MSBuild defaults injected into every project
├── Directory.Build.targets    # MSBuild targets injected after every project
├── Directory.Packages.props   # CPM versions
├── EntraAuthPatterns.slnx     # the one solution (or per-slice .slnf)
├── src/                       # production assemblies
├── tests/                     # *.Tests projects (mirror src tree)
├── samples/                   # runnable demos; never referenced by src
├── tools/                     # internal CLIs, code generators, dotnet-tool sources
├── eng/                       # build infra: targets, scripts, pipeline yaml
├── docs/
└── .github/workflows/         # CI
```

The inheritance order is the part people get wrong. MSBuild walks **upwards**
from each `.csproj` looking for `Directory.Build.props` (imported *before* the
SDK) and `Directory.Build.targets` (imported *after*). It stops at the first hit
unless you re-import the parent explicitly. See
[Customize your build —
`Directory.Build.props`](https://learn.microsoft.com/visualstudio/msbuild/customize-by-directory).
If you put a `Directory.Build.props` in `src/` to override repo-root settings,
re-import the root with:

```xml
<Project>
  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />
  <PropertyGroup>
    <IsPackable>true</IsPackable>
  </PropertyGroup>
</Project>
```

`nuget.config` likewise composes from machine → user → repo. Use [package
source mapping](https://learn.microsoft.com/nuget/consume-packages/package-source-mapping)
in the repo-root `nuget.config` so internal packages can *only* be resolved
from the internal feed. This is the single most effective defence against
[dependency confusion](https://learn.microsoft.com/nuget/concepts/security-best-practices#enable-package-source-mapping)
attacks, and it is mandatory in any monorepo that mixes nuget.org with a
corporate feed.

```xml
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="contoso"   value="https://pkgs.dev.azure.com/contoso/_packaging/internal/nuget/v3/index.json" />
  </packageSources>
  <packageSourceMapping>
    <packageSource key="nuget.org">
      <package pattern="*" />
    </packageSource>
    <packageSource key="contoso">
      <package pattern="Contoso.*" />
    </packageSource>
  </packageSourceMapping>
</configuration>
```

### 1.3 Solution files: `.slnx` vs `.sln` vs `.slnf` vs traversal

[`.slnx`](https://devblogs.microsoft.com/visualstudio/new-simpler-solution-file-format/)
is the XML solution format, GA in VS 17.14 / .NET SDK 9.0.200+. It is now the
default for new repos: human-readable, mergeable, and supported by `dotnet sln`,
MSBuild, Rider, and VS. `dotnet sln migrate ./Foo.sln` converts in place.

For **one big solution vs many**, the practical break-even is around 60–80
projects on a developer workstation — beyond that the IDE indexer and Roslyn
service start dragging. Above that threshold use [solution filters
(`.slnf`)](https://learn.microsoft.com/visualstudio/ide/filtered-solutions)
keyed to dev workflows: `Backend.slnf`, `Workers.slnf`, `Web.slnf`. CI still
loads the whole `.slnx` so consistency holds.

For headless build orchestration, prefer [MSBuild traversal
projects](https://github.com/microsoft/MSBuildSdks/tree/main/src/Traversal)
(`Microsoft.Build.Traversal`) over `dotnet sln`-driven scripts:

```xml
<!-- eng/dirs.proj -->
<Project Sdk="Microsoft.Build.Traversal/4.1.0">
  <ItemGroup>
    <ProjectReference Include="..\src\**\*.csproj" />
    <ProjectReference Include="..\tests\**\*.csproj" />
  </ItemGroup>
</Project>
```

`dotnet build eng/dirs.proj /graph:true /restore` then evaluates the full
project graph in one MSBuild process — much faster than iterating `.sln`
entries. `dotnet build` directly on a folder works for trivial cases but does
not give you graph evaluation.

### 1.4 Central Package Management — beyond the basics

[CPM](https://learn.microsoft.com/nuget/consume-packages/central-package-management)
is non-negotiable in a monorepo. The minimal `Directory.Packages.props`:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Microsoft.Extensions.Hosting" Version="10.0.0" />
    <PackageVersion Include="Serilog.AspNetCore"          Version="9.0.0" />
  </ItemGroup>
  <ItemGroup>
    <!-- analyzers everywhere, never per-csproj -->
    <GlobalPackageReference Include="Meziantou.Analyzer"             Version="2.0.196" />
    <GlobalPackageReference Include="Microsoft.CodeAnalysis.PublicApiAnalyzers" Version="3.11.0-beta1.24658.1" />
  </ItemGroup>
</Project>
```

Key facts:

- **`CentralPackageTransitivePinningEnabled=true`** pins transitive versions
  too. Without it, a transitive package can float and break reproducibility.
  See [the spec
  PR](https://github.com/NuGet/Home/blob/dev/proposed/2022/CentralPackageManagement-TransitiveDependencies.md).
- **`VersionOverride`** lets a single project upgrade a single package without
  forking the whole repo — use it in a PR that is explicitly the "trial
  upgrade" PR, then promote to `Directory.Packages.props` once green:
  ```xml
  <PackageReference Include="Npgsql" VersionOverride="9.0.2" />
  ```
- **`GlobalPackageReference`** propagates the reference to every project
  in the cone and is the *only* correct way to apply analyzers in CPM —
  putting them in `<PackageVersion>` does nothing on its own.
- **`<PackagePrune>` (NET 10)** lets you exclude a package from the resolution
  graph that would otherwise be pulled in by a framework alias. See
  [the .NET 10 SDK release
  notes](https://github.com/dotnet/core/blob/main/release-notes/10.0/preview/preview6/sdk.md#package-pruning)
  — useful when the BCL ships a feature in-box and you want to drop the
  legacy compat package the way `Microsoft.Bcl.AsyncInterfaces` was historically
  pulled in.
- **Prereleases**: keep them out of the root file; gate behind a `*-Preview.props`
  imported only from projects that opt in (`<Import Project="..\Preview.props"
  Condition="'$(UsePreviewPackages)'=='true'"/>`). This stops a single drive-by
  PR from globally adopting a preview.
- **Trial upgrades**: branch → bump *one* `PackageVersion` → CI runs the full
  monorepo build + tests. Don't combine upgrades.

### 1.5 ProjectReference vs PackageReference — the internal NuGet boundary

Inside the monorepo, default to `ProjectReference`. Switch to a `PackageReference`
against your internal feed when **both** are true:

1. The component has its own SemVer cadence (you tag releases for it).
2. Consumers want to hold an old version while you ship a new one (i.e. you
   need decoupled deploys).

Otherwise `ProjectReference` is faster to build, gives you go-to-definition
across the monorepo, and keeps refactors atomic. The pattern of "everything is
a NuGet, even within the same repo" — call it the *internal NuGet boundary
anti-pattern* — is regularly cited by [Andrew
Lock](https://andrewlock.net/) as a source of needless friction; only adopt it
at real architectural seams.

When you *do* publish internally, set in the producing project:

```xml
<IsPackable>true</IsPackable>
<PackageId>Contoso.Auth.Abstractions</PackageId>
<EnablePackageValidation>true</EnablePackageValidation>
<PackageValidationBaselineVersion>1.4.0</PackageValidationBaselineVersion>
```

[`Microsoft.DotNet.PackageValidation`](https://learn.microsoft.com/dotnet/fundamentals/apicompat/package-validation/overview)
will fail the build on accidental binary-breaking changes.

### 1.6 Multi-targeting matrix

Multi-target only when you actually ship to multiple TFMs (libraries on nuget.org,
shared SDKs, Roslyn analyzers). Apps target one TFM.

```xml
<PropertyGroup>
  <TargetFrameworks>net10.0;net8.0</TargetFrameworks>
</PropertyGroup>

<ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
  <PackageReference Include="Microsoft.Bcl.TimeProvider" />
</ItemGroup>

<ItemGroup Condition="'$(TargetFramework)' == 'net10.0'">
  <!-- analyzers that require Roslyn 4.12+ -->
  <PackageReference Include="Meziantou.Analyzer" />
</ItemGroup>
```

Conditional `PackageVersion` lives in `Directory.Packages.props`:

```xml
<ItemGroup Condition="'$(TargetFramework)'=='net10.0'">
  <PackageVersion Include="System.Text.Json" Version="10.0.0" />
</ItemGroup>
<ItemGroup Condition="'$(TargetFramework)'=='net8.0'">
  <PackageVersion Include="System.Text.Json" Version="8.0.5" />
</ItemGroup>
```

### 1.7 Build performance

Monorepo build time is the #1 reason teams abandon them. The levers, in order
of payoff:

1. **MSBuild static graph** — `dotnet build /graph:true /restore` evaluates
   the project graph once and parallelises across the topology.
   [Docs](https://learn.microsoft.com/visualstudio/msbuild/static-graph). Pair
   with `RestoreUseStaticGraphEvaluation=true` in CI.
2. **Parallelism** — `dotnet build /m:N`. On CI agents with N cores, set N
   explicitly; defaults can be conservative.
3. **NuGet lock files** — `<RestorePackagesWithLockFile>true</...>` and
   `dotnet restore --locked-mode` in CI. Reproducible restores, faster cold
   restore (graph is precomputed).
   [NuGet docs](https://learn.microsoft.com/nuget/consume-packages/package-references-in-project-files#locking-dependencies).
4. **HTTP cache + global packages folder** — cache `~/.nuget/packages` and
   `~/.nuget/http-cache` between CI runs; cache key = hash of all
   `packages.lock.json` and `Directory.Packages.props`. See [`actions/cache`
   recipe in the .NET docs](https://learn.microsoft.com/dotnet/devops/dotnet-test-github-action#cache-nuget-packages).
5. **Incremental build cache** — MSBuild already does input/output timestamp
   caching at the target level; the new opt-in
   [build cache + replay
   (`BuildCacheReplay`)](https://github.com/dotnet/msbuild/issues/8531)
   in MSBuild 17.13+ allows distributed cache reuse. Currently most useful
   inside Azure DevOps with `MSBuildBinaryLog` artifacts.
6. **`--use-current-runtime` for `dotnet test/run`** — skips RID-specific
   restore overhead when you're only running the host's RID locally.
7. **`obj/` cache** — restoring `obj/` between CI runs reuses the asset file
   (`project.assets.json`) and skips the `dotnet restore` walk entirely. Key
   on `Directory.Packages.props` + all `.csproj` hashes.

A realistic CI delta: a 200-project repo cold-builds in ~7 minutes; with the
above it warms to ~90s.

### 1.8 Testing in monorepos

- **Discovery**: glob `tests/**/*.Tests.csproj` from a traversal project.
- **Sharding**: split by directory (`tests/Auth/**`, `tests/Billing/**`) into
  matrix jobs in CI; each job restores the same packages so the cache is hot.
- **Filter**: `dotnet test --filter "Category=Fast&FullyQualifiedName!~Slow"`
  or `--filter Trait=Integration` to slice by xUnit/NUnit traits.
- **Microsoft.Testing.Platform** is the new test host (replaces VSTest);
  [docs](https://learn.microsoft.com/dotnet/core/testing/microsoft-testing-platform-intro).
  Opt in with `<UseMicrosoftTestingPlatformRunner>true</...>`. Faster startup,
  better CI exit codes, native `--report-trx` support.
- **xUnit v3** parallel collections: by default tests in different *classes*
  run in parallel within an assembly; opt out per-collection with
  `[Collection("Db")]`.
- **Deterministic seeds**: every randomised test takes a `seed` from
  `TEST_SEED` env or a fixed default; print it on failure. Don't use
  `DateTime.Now`; inject `TimeProvider`.
- **Snapshots**: [Verify](https://github.com/VerifyTests/Verify) — commit
  `*.verified.txt`, fail on mismatch with a CI-friendly diff.
- **Coverage**: `coverlet.collector` per-project + a single
  [`reportgenerator`](https://github.com/danielpalme/ReportGenerator) merge
  step (`reportgenerator -reports:**/coverage.cobertura.xml
  -targetdir:./coverage -reporttypes:Cobertura;Html`). Don't try to compute
  monorepo-wide coverage from a single test run; merge afterwards.

### 1.9 Versioning + release

Two viable models:

- **Single-version monorepo** — one `Version.props` in `eng/`, every package
  ships `1.42.0`. Simple, atomic. Best when you don't expose libraries to the
  outside world.
- **Per-project SemVer** — each packable project owns its version, computed by
  [MinVer](https://github.com/adamralph/minver) (Git-tag based) or
  [Nerdbank.GitVersioning](https://github.com/dotnet/Nerdbank.GitVersioning)
  (`version.json` per directory, height-based). Required when consumers depend
  on individual packages.

MinVer is the simpler default:

```xml
<PackageReference Include="MinVer" Version="6.0.0" PrivateAssets="all" />
<PropertyGroup>
  <MinVerTagPrefix>v</MinVerTagPrefix>
  <MinVerDefaultPreReleaseIdentifiers>preview.0</MinVerDefaultPreReleaseIdentifiers>
</PropertyGroup>
```

Enforce SemVer on packable projects with
[`Microsoft.CodeAnalysis.PublicApiAnalyzers`](https://github.com/dotnet/roslyn-analyzers/blob/main/src/PublicApiAnalyzers/PublicApiAnalyzers.Help.md):
maintain `PublicAPI.Shipped.txt` and `PublicAPI.Unshipped.txt` per project; the
analyzer flags any addition/removal not reflected in those files. Combined with
`Microsoft.DotNet.PackageValidation`, you get build-time SemVer enforcement
without anyone reading a checklist.

### 1.10 CI/CD

**Change detection.** Don't run the whole monorepo on every PR. Two options:

1. **Path filters in GitHub Actions** — coarse, fine for a small repo:
   ```yaml
   on:
     pull_request:
       paths: ['src/Auth/**', 'tests/Auth/**', 'Directory.*.props']
   ```
2. **Project-graph diff** — compute reverse dependencies from a base SHA. The
   `dotnet-affected` tool ([github.com/leonardochaia/dotnet-affected](https://github.com/leonardochaia/dotnet-affected))
   does this:
   ```bash
   dotnet affected -p eng/dirs.proj --from origin/main --format text
   ```
   Feed the output into a matrix.

**Containers without Dockerfiles.** [`Microsoft.NET.Build.Containers`](https://learn.microsoft.com/dotnet/core/docker/publish-as-container)
is in-box in .NET 8+:

```bash
dotnet publish src/Web -c Release -t:PublishContainer \
  /p:ContainerRepository=contoso/web \
  /p:ContainerImageTag=$(git rev-parse --short HEAD)
```

It produces a chiselled, distroless-style image with no Dockerfile to maintain.
Pair with `<ContainerFamily>noble-chiseled</ContainerFamily>` for the smallest
runtime image. See [the team's
post](https://devblogs.microsoft.com/dotnet/announcing-builtin-container-support-for-the-dotnet-sdk/).

**Publish profiles** keep environment-specific publish settings out of CI YAML:
`dotnet publish /p:PublishProfile=Properties/PublishProfiles/AksProd.pubxml`.

### 1.11 Code ownership and architectural enforcement

- `CODEOWNERS` per directory:
  ```
  /src/Auth/**          @contoso/auth-team
  /src/Billing/**       @contoso/billing-team
  Directory.*.props     @contoso/platform @contoso/eng-leads
  ```
- Architecture tests:
  [NetArchTest](https://github.com/BenMorris/NetArchTest) or
  [ArchUnitNET](https://github.com/TNG/ArchUnitNET) — run as xUnit tests in CI.
  Enforce "no `Auth.*` may reference `Billing.*`", "domain has no MVC
  reference", etc.
- Banned APIs:
  [`Microsoft.CodeAnalysis.BannedApiAnalyzers`](https://github.com/dotnet/roslyn-analyzers/blob/main/src/Microsoft.CodeAnalysis.BannedApiAnalyzers/BannedApiAnalyzers.Help.md)
  with a checked-in `BannedSymbols.txt`:
  ```
  T:System.DateTime;Use TimeProvider.GetUtcNow() instead.
  M:System.Net.Http.HttpClient.#ctor;Use IHttpClientFactory.
  ```

### 1.12 Refactoring at scale

- `dotnet format` (with `--severity error --verify-no-changes` in CI) keeps
  whitespace and `using`s normalised; combined with
  `EnforceCodeStyleInBuild=true` you can rely on consistent diffs.
- Roslyn analyzers + code fixes are the right vehicle for *semantic* refactors
  spanning the whole repo (e.g. "rename `IFoo` and update all impls" beyond what
  IDE rename can do across multi-targeting). Ship the analyzer in `tools/` as
  an internal source generator project; run via `dotnet build /p:RunAnalyzers=true`.
- For one-shot mass edits prefer
  [Roslynator's CLI](https://josefpihrt.github.io/docs/roslynator/cli) over
  `sed` — it understands C# syntax and won't munge strings.

### 1.13 Trade-offs (be honest)

- **Restore time** scales with package count, not project count. CPM helps
  because every project resolves the same `Directory.Packages.props` graph.
- **PR diff size** balloons when a `Directory.Build.props` change touches
  every project — keep such PRs separate, with a single owner.
- **Blast radius**: a typo in `Directory.Build.props` breaks 200 projects.
  Mitigate with a `dotnet build eng/dirs.proj /graph:true` PR check.
- **IDE memory**: VS / Rider with 200 projects loaded eats ~6 GB. Solution
  filters are not optional past ~80 projects.
- **CI cost**: change-detection is the only way to keep CI bills sane. If you
  can't do it on day one, plan it for week three.

---

## 2. Anti-patterns that create chaos in .NET codebases

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
`foundations.md` §3.

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

## 3. Patterns to follow

### Composition root + DI graph testing

Compose at the host (`Program.cs`); test the graph:

```csharp
[Fact]
public void Container_resolves_all_services()
{
    using var app = new WebApplicationFactory<Program>();
    using var scope = app.Services.CreateScope();
    foreach (var sd in app.Services.GetRequiredService<IServiceCollection>()
                            .Where(s => s.ServiceType.IsPublic && !s.ServiceType.IsGenericTypeDefinition))
    {
        _ = scope.ServiceProvider.GetRequiredService(sd.ServiceType);
    }
}
```

(In practice you snapshot the registered `IServiceCollection` at host build
time via a test seam.)

### Options pattern, validated, fail-fast

```csharp
services.AddOptions<GraphOptions>()
        .Bind(config.GetSection("Graph"))
        .ValidateDataAnnotations()
        .Validate(o => o.TenantId != Guid.Empty, "TenantId required")
        .ValidateOnStart();
```

Combine with the [`[OptionsValidator]` source
generator](https://learn.microsoft.com/dotnet/core/extensions/options-library-authors#validateoptionssourcegen)
to avoid reflection at startup.

### Source generators for hot paths

```csharp
[LoggerMessage(Level = LogLevel.Information, Message = "Token issued for {ClientId}")]
public static partial void TokenIssued(this ILogger logger, string clientId);

[JsonSerializable(typeof(TokenResponse))]
internal partial class AuthJsonContext : JsonSerializerContext;

[GeneratedRegex("^[A-Z]{2}[0-9]{6}$", RegexOptions.CultureInvariant)]
private static partial Regex AccountIdRegex();
```

ASP.NET Core also ships a [Configuration binder source
generator](https://learn.microsoft.com/dotnet/core/extensions/configuration-generator)
and an OpenAPI source generator (`Microsoft.AspNetCore.OpenApi` in .NET 9+) —
all eliminate reflection at startup, all are AOT-safe.

### `TimeProvider` for testable time

```csharp
public sealed class Clock(TimeProvider time)
{
    public DateTimeOffset Now => time.GetUtcNow();
}
```

Test with `FakeTimeProvider`:

```csharp
var time = new FakeTimeProvider(DateTimeOffset.Parse("2025-01-01Z"));
time.Advance(TimeSpan.FromHours(2));
```

### Outbox + idempotent handlers

For reliable messaging, never publish from inside `SaveChangesAsync` — write
to an outbox table in the same transaction, dispatch from a relay.
[Jimmy Bogard, *Reliable messaging without distributed
transactions*](https://www.jimmybogard.com/refactoring-towards-resilience-evaluating-coupling/).
Wolverine and MassTransit both ship transactional-outbox EF Core integrations
out of the box ([Wolverine
docs](https://wolverinefx.net/guide/durability/),
[MassTransit
docs](https://masstransit.io/documentation/configuration/middleware/outbox)).

Idempotency keys on every external command: hash `(message-id, handler)` and
short-circuit re-deliveries — exactly-once is a fairy tale, idempotent
at-least-once is the achievable goal.

### Vertical slice vs Clean Architecture

- **Vertical slice** ([Jimmy
  Bogard](https://www.jimmybogard.com/vertical-slice-architecture/)): organize
  by feature folder, each slice has its own request/handler/validator/dto;
  shared abstractions only when *real* duplication appears. Best for systems
  where features evolve independently.
- **Clean Architecture** ([Steve Smith's
  template](https://github.com/ardalis/CleanArchitecture)): organize by layer
  (Domain / Application / Infrastructure / Web). Best for stable, complex
  domains with many cross-cutting policies.

Hybrid is fine: Clean at the assembly level, vertical-slice within
`Application`. Pick the *primary* axis.

### CQRS only where asymmetry exists

If your reads and writes have the same shape, MediatR + handlers *is* CQRS
theatre — Khorikov's
[*CQRS without 
DTOs*](https://enterprisecraftsmanship.com/posts/types-of-cqrs/). Adopt
read-model projections only when the asymmetry is real: different scaling,
different storage, different consistency.

### Domain events vs integration events

Domain events fire inside the aggregate boundary, dispatched in-process after
`SaveChanges`. Integration events leave the process via the outbox. Don't
conflate; the failure modes differ.

### Result types for expected failures

```csharp
public Result<Order, OrderError> Place(Cart cart) =>
    cart.Items.Count == 0
        ? OrderError.Empty
        : new Order(cart);
```

Libraries: [`OneOf`](https://github.com/mcintyre321/OneOf),
[`ErrorOr`](https://github.com/amantinband/error-or),
[`FluentResults`](https://github.com/altmann/FluentResults). Use exceptions
for *exceptional* (programmer error, infrastructure failure), results for
*expected* (validation, business rule). Khorikov,
[*Functional C#: Handling failures, input
errors*](https://enterprisecraftsmanship.com/posts/functional-c-handling-failures-input-errors/).

### Strongly-typed IDs

```csharp
[ValueObject<Guid>] public partial struct OrderId;
```

[Vogen](https://github.com/SteveDunn/Vogen) and
[StronglyTypedId](https://github.com/andrewlock/StronglyTypedId) both source-
generate the boilerplate. Eliminates `new OrderRepository().Get(customerId)`-style
bugs.

### Value objects + `record struct`

For tiny immutable values that get passed around hot loops, `readonly record
struct Money(decimal Amount, string Currency)` gives you value equality with
no heap allocation.

### Specification pattern

[Ardalis.Specification](https://github.com/ardalis/Specification) for
reusable EF query criteria; only adopt when you have ≥3 places asking the
same question. One-off queries belong inline.

### Repository ONLY for aggregate roots

Per Eric Evans' DDD book and Vaughn Vernon's IDDD: one repository per
*aggregate root*, not per table. `IOrderRepository`, not `IOrderLineRepository`.

### Unit of Work via EF Core

`DbContext` is the unit of work. Don't wrap it. If you need cross-repo
transactional behaviour, share the `DbContext` (it's already scoped). For
domain events, use [SaveChanges
interceptors](https://learn.microsoft.com/ef/core/logging-events-diagnostics/interceptors#savechanges-interceptor)
to dispatch after a successful commit.

### `BackgroundService` + `Channel<T>`

```csharp
public sealed class EmailQueue
{
    private readonly Channel<EmailJob> _ch = Channel.CreateBounded<EmailJob>(
        new BoundedChannelOptions(1024) { FullMode = BoundedChannelFullMode.Wait });
    public ValueTask EnqueueAsync(EmailJob j, CancellationToken ct) => _ch.Writer.WriteAsync(j, ct);
    public IAsyncEnumerable<EmailJob> ReadAllAsync(CancellationToken ct) => _ch.Reader.ReadAllAsync(ct);
}

public sealed class EmailWorker(EmailQueue queue, IEmailSender sender) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stop)
    {
        await foreach (var j in queue.ReadAllAsync(stop))
            await sender.SendAsync(j, stop);
    }
}
```

`BoundedChannelOptions.FullMode = Wait` gives you backpressure for free.

### Graceful shutdown

`IHostApplicationLifetime.ApplicationStopping` fires before `StopAsync`;
register cleanup there. Configure `HostOptions.ShutdownTimeout` to a value
larger than your longest in-flight request — default is 30s.

### `ProblemDetails` + `IExceptionHandler`

```csharp
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<DomainExceptionHandler>();
app.UseExceptionHandler();
```

[`IExceptionHandler` (NET 8+)](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling#iexceptionhandler)
is the new chain — register multiple, return `true` to mark handled.

### `WebApplicationFactory` integration tests

Use a `TestAuthHandler` that issues a synthetic claims principal so
`[Authorize]` routes are exercised; replace `IExternalApi` with a
`WireMock.Net` fixture. See
[Integration tests in ASP.NET
Core](https://learn.microsoft.com/aspnet/core/test/integration-tests).

### Testcontainers

[`Testcontainers for .NET`](https://dotnet.testcontainers.org/) — real
Postgres/SQL Server/Redis in Docker per test class via `[CollectionFixture]`.
Faster and more correct than in-memory providers.

### Snapshot testing

```csharp
await Verify(response).ScrubLinesContaining("traceparent");
```

Verify integrates with xUnit/NUnit/MSTest; commit the `*.verified.txt`.

### Architecture tests

```csharp
var result = Types.InAssembly(typeof(OrderHandler).Assembly)
    .That().ResideInNamespace("Contoso.Domain")
    .Should().NotHaveDependencyOn("Microsoft.EntityFrameworkCore")
    .GetResult();
Assert.True(result.IsSuccessful, string.Join(",", result.FailingTypeNames ?? []));
```

### Public-API tracking

Per packable project: `PublicAPI.Shipped.txt` + `PublicAPI.Unshipped.txt`.
Combined with `EnablePackageValidation`, accidental breaks fail the build.

### `IAsyncDisposable` discipline

```csharp
await using var conn = await _factory.CreateAsync(ct);
```

Both `using` and `await using` on the same scope are fine; don't mix synchronous
disposal with async resources or you block on shutdown.

### Telemetry-by-construction

Every cross-boundary type takes `ILogger<T>`, owns an `ActivitySource`, and
emits a `Meter`. Bake it in at construction so you don't retrofit it after
incidents:

```csharp
public sealed class GraphClient(HttpClient http, ILogger<GraphClient> log)
{
    private static readonly ActivitySource Activity = new("Contoso.Graph");
    private static readonly Meter Meter = new("Contoso.Graph");
    private static readonly Counter<long> CallCount = Meter.CreateCounter<long>("graph.calls");
    // ...
}
```

OpenTelemetry [docs for
.NET](https://opentelemetry.io/docs/instrumentation/net/).

---

## 4. GoF + .NET-idiomatic patterns

For each: when to use → 5–15 line C# → "in .NET, prefer X built-in over hand-roll".

### Strategy
Pick an algorithm at runtime. Resolve via DI; don't build your own factory.
```csharp
public interface IHasher { string Hash(string s); }
public sealed class Sha256Hasher : IHasher { /* ... */ }
public sealed class Argon2Hasher : IHasher { /* ... */ }

services.AddKeyedSingleton<IHasher, Sha256Hasher>("sha256");
services.AddKeyedSingleton<IHasher, Argon2Hasher>("argon2");

public sealed class Login([FromKeyedServices("argon2")] IHasher hasher);
```
Prefer `IServiceProvider.GetRequiredKeyedService<T>` over `Func<string, IHasher>` factories.

### Decorator
Wrap an interface to add cross-cutting behaviour.
```csharp
services.AddScoped<IOrderService, OrderService>();
services.Decorate<IOrderService, LoggingOrderDecorator>();   // Scrutor
services.Decorate<IOrderService, CachingOrderDecorator>();
```
Prefer [Scrutor](https://github.com/khellang/Scrutor) `Decorate` over hand-rolled wrappers; for AOT-safe alternatives use a source generator (e.g. [DispatchProxy](https://learn.microsoft.com/dotnet/api/system.reflection.dispatchproxy) is not AOT).

### Adapter
Translate one interface to another at a boundary.
```csharp
public sealed class GraphUserAdapter(GraphClient g) : IUserDirectory
{
    public async Task<User?> FindAsync(string upn, CancellationToken ct)
        => (await g.GetUserAsync(upn, ct))?.ToUser();
}
```
Prefer hand-written adapters over reflection mappers at boundaries — they're the documentation of the seam.

### Factory / Abstract Factory
Defer construction.
```csharp
services.AddScoped<Func<TenantId, ITenantContext>>(sp =>
    id => ActivatorUtilities.CreateInstance<TenantContext>(sp, id));
```
For most cases, `IServiceProvider.GetRequiredKeyedService<T>(key)` replaces the factory entirely.

### Builder
Step-wise construction of complex objects.
```csharp
var req = new HttpRequestMessage(HttpMethod.Post, url);
req.Headers.Authorization = new("Bearer", token);
req.Content = JsonContent.Create(payload, options: AuthJsonContext.Default.Options);
```
The BCL already gives you fluent builders for the common cases (`HttpRequestMessage`, `WebApplicationBuilder`, `HostBuilder`, `BlobClientOptionsBuilder`); only hand-roll a builder for *your* domain-specific construction.

### Singleton
A single shared instance.
```csharp
services.AddSingleton<IClock, SystemClock>();
```
Always via the DI container, never `static readonly Foo Instance = new();` — the latter is untestable and forces lifetime decisions on consumers.

### Observer
Push notifications to subscribers.
```csharp
var ch = Channel.CreateUnbounded<OrderPlaced>();
_ = Task.Run(async () => { await foreach (var e in ch.Reader.ReadAllAsync()) await Handle(e); });
await ch.Writer.WriteAsync(new OrderPlaced(id));
```
For in-proc events use `Channel<T>` (backpressure!) or MediatR notifications. Skip `IObservable<T>` unless you genuinely need Rx operators.

### Mediator
Decouple senders from handlers.
```csharp
public record PlaceOrder(CartId Cart) : IRequest<OrderId>;
public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrder, OrderId> { /* ... */ }
var orderId = await mediator.Send(new PlaceOrder(cart));
```
[MediatR](https://github.com/jbogard/MediatR) for in-proc; [Wolverine](https://wolverinefx.net/) when you also want messaging + outbox in the same primitive. Don't use mediator as a hand-rolled service-locator.

### Command
Encapsulate a request as an object — the C in CQRS.
Same shape as Mediator above; commands are the *write* side, return `OrderId` or `Result<…>`, never query data back.

### Chain of Responsibility
Pass a request through a chain.
```csharp
app.UseExceptionHandler();
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
```
ASP.NET Core middleware *is* CoR. For exceptions specifically, register multiple `IExceptionHandler`s — the framework chains them.

### Template Method
Algorithm skeleton with overridable steps.
```csharp
public abstract class HostedJob : BackgroundService
{
    protected sealed override async Task ExecuteAsync(CancellationToken stop)
    {
        await OnStartAsync(stop);
        while (!stop.IsCancellationRequested) await TickAsync(stop);
    }
    protected abstract Task TickAsync(CancellationToken ct);
    protected virtual Task OnStartAsync(CancellationToken ct) => Task.CompletedTask;
}
```
`sealed override` on the framework method, `abstract` on the customisation point — communicates the contract.

### Visitor
Operate on a closed hierarchy.
```csharp
decimal Total(LineItem item) => item switch
{
    Product p   => p.Price * p.Qty,
    Discount d  => -d.Amount,
    Shipping s  => s.Cost,
    _ => throw new UnreachableException()
};
```
C# pattern matching on sealed hierarchies is the idiomatic Visitor. Mark the base type `abstract` and *all* derivations `sealed`; the compiler then warns on missing arms.

### Specification
Encapsulate a query predicate.
```csharp
public sealed class ActiveOrdersOver(decimal min) : Specification<Order>
{
    public ActiveOrdersOver() { Query.Where(o => o.IsActive && o.Total >= min); }
}
```
Use [Ardalis.Specification](https://github.com/ardalis/Specification); plays nicely with EF Core projections.

### Null Object
Avoid null branches with a do-nothing implementation.
```csharp
public static class NullClock { public static readonly IClock Instance = new Impl(); /* ... */ }
```
The BCL ships [`NullLogger<T>.Instance`](https://learn.microsoft.com/dotnet/api/microsoft.extensions.logging.abstractions.nulllogger-1) as the canonical example.

### Memento
Capture state for undo.
```csharp
var snapshot = doc.SaveSnapshot();
doc.Mutate();
if (cancel) doc.Restore(snapshot);
```
For larger graphs: serialise via STJ source-gen so the snapshot is a `byte[]`.

### Composite
Treat one and many uniformly.
```csharp
public sealed class CompositeHealthCheck(IEnumerable<IHealthCheck> inner) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(...)
        => /* aggregate */;
}
```
Most BCL composites are already provided: `IConfiguration` chains providers, `LoggerProvider` chains writers, `IFileProvider` composes via `CompositeFileProvider`.

### Proxy
Stand-in object controlling access.
For cross-cutting at runtime, [Castle DynamicProxy](https://github.com/castleproject/Core) is fine on JIT but not AOT. AOT-safe alternative: a Roslyn source generator that produces a sealed wrapper class. Decorator via Scrutor covers most "I want logging around this interface" cases without proxies at all.

### Flyweight
Share immutable state to save memory.
```csharp
ReadOnlySpan<byte> data = ...;
var rented = ArrayPool<byte>.Shared.Rent(8192);
try { /* use rented */ } finally { ArrayPool<byte>.Shared.Return(rented); }
```
BCL: `ArrayPool<T>`, `MemoryPool<T>`, `string.Intern`/`StringPool` (CommunityToolkit). Hand-roll only after a profiler shows allocation pressure.

### Bridge
Separate abstraction from implementation so each varies independently.
`ILogger` (abstraction) ↔ `ILoggerProvider` (impl) is the canonical .NET example: Serilog, Console, EventSource all plug in without consumers changing.

### Iterator
Lazy traversal.
```csharp
public async IAsyncEnumerable<Page> PagesAsync([EnumeratorCancellation] CancellationToken ct)
{
    string? next = null;
    do { var p = await Fetch(next, ct); yield return p; next = p.NextLink; } while (next is not null);
}
```
`yield return` for sync, `IAsyncEnumerable<T>` + `[EnumeratorCancellation]` for async. Don't materialise to `List<T>` unless the caller needs random access.

### State
Behaviour varies with state.
For non-trivial workflows use [Stateless](https://github.com/dotnet-state-machine/stateless); for simple toggles, an `enum` + `switch` is enough — don't introduce a state-machine library for two states.

### Interpreter
Evaluate sentences in a language.
You almost never write this by hand in .NET — Roslyn (C#), `System.Linq.Expressions`, EF Core's LINQ provider, and [Sprache](https://github.com/sprache/Sprache) cover the realistic cases. If you find yourself building one, you probably want a config DSL — consider YAML/JSON + a typed binding instead.

---

## 5. References

### Microsoft — Docs

- [.NET Application Architecture Guides (hub)](https://dotnet.microsoft.com/learn/dotnet/architecture-guides)
- [.NET Microservices: Architecture for Containerized .NET Applications (eBook)](https://learn.microsoft.com/dotnet/architecture/microservices/)
- [Architecting Modern Web Applications with ASP.NET Core (eBook)](https://learn.microsoft.com/dotnet/architecture/modern-web-apps-azure/)
- [.NET Microservices ebook GitHub repo](https://github.com/dotnet-architecture/eShopOnContainers)
- [Customize your build — `Directory.Build.props`/`.targets`](https://learn.microsoft.com/visualstudio/msbuild/customize-by-directory)
- [`.slnx` solution file format](https://devblogs.microsoft.com/visualstudio/new-simpler-solution-file-format/)
- [Solution filters (`.slnf`)](https://learn.microsoft.com/visualstudio/ide/filtered-solutions)
- [MSBuild static graph](https://learn.microsoft.com/visualstudio/msbuild/static-graph)
- [Microsoft.Build.Traversal SDK](https://github.com/microsoft/MSBuildSdks/tree/main/src/Traversal)
- [Central Package Management](https://learn.microsoft.com/nuget/consume-packages/central-package-management)
- [NuGet package source mapping](https://learn.microsoft.com/nuget/consume-packages/package-source-mapping)
- [NuGet lock files](https://learn.microsoft.com/nuget/consume-packages/package-references-in-project-files#locking-dependencies)
- [NuGet auditing](https://learn.microsoft.com/nuget/concepts/auditing-packages)
- [NuGet warning NU1010](https://learn.microsoft.com/nuget/reference/errors-and-warnings/nu1010)
- [Package validation](https://learn.microsoft.com/dotnet/fundamentals/apicompat/package-validation/overview)
- [`Microsoft.NET.Build.Containers`](https://learn.microsoft.com/dotnet/core/docker/publish-as-container)
- [Microsoft Testing Platform](https://learn.microsoft.com/dotnet/core/testing/microsoft-testing-platform-intro)
- [Integration tests in ASP.NET Core](https://learn.microsoft.com/aspnet/core/test/integration-tests)
- [`IExceptionHandler` (NET 8+)](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling#iexceptionhandler)
- [HybridCache](https://learn.microsoft.com/aspnet/core/performance/caching/hybrid)
- [`TimeProvider` overview](https://learn.microsoft.com/dotnet/standard/datetime/timeprovider-overview)
- [Options pattern](https://learn.microsoft.com/dotnet/core/extensions/options)
- [`[OptionsValidator]` source generator](https://learn.microsoft.com/dotnet/core/extensions/options-library-authors#validateoptionssourcegen)
- [Configuration binder source generator](https://learn.microsoft.com/dotnet/core/extensions/configuration-generator)
- [`LoggerMessage` source generator](https://learn.microsoft.com/dotnet/core/extensions/logger-message-generator)
- [`System.Text.Json` source generation](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation)
- [`GeneratedRegex`](https://learn.microsoft.com/dotnet/standard/base-types/regular-expression-source-generators)
- [DI guidelines / pitfalls](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines)
- [`IHttpClientFactory`](https://learn.microsoft.com/dotnet/core/extensions/httpclient-factory)
- [`Microsoft.Extensions.Http.Resilience`](https://learn.microsoft.com/dotnet/core/resilience/http-resilience)
- [.NET upgrade assistant](https://learn.microsoft.com/dotnet/core/porting/upgrade-assistant-overview)
- [App secrets in development](https://learn.microsoft.com/aspnet/core/security/app-secrets)
- [`dotnet test` GitHub Action](https://learn.microsoft.com/dotnet/devops/dotnet-test-github-action)

### Microsoft — Team blogs

- [.NET Blog — built-in container support](https://devblogs.microsoft.com/dotnet/announcing-builtin-container-support-for-the-dotnet-sdk/)
- [.NET Blog — `ValueTask` whys, whats, whens (Stephen Toub)](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)
- [Visual Studio Blog — `.slnx`](https://devblogs.microsoft.com/visualstudio/new-simpler-solution-file-format/)
- [.NET 10 SDK release notes](https://github.com/dotnet/core/blob/main/release-notes/10.0/preview/preview6/sdk.md)
- [.NET Conf playlists (YouTube)](https://www.youtube.com/@dotnet)
- [Donovan Brown — DevOps on .NET](https://www.donovanbrown.com/)
- [Abel Wang — Azure DevOps + .NET (archive)](https://devblogs.microsoft.com/devops/author/abelwang/)

### Community thought-leaders

- [Stephen Cleary — *Don't Block on Async Code*](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html)
- [Stephen Cleary — *Async/Await Best Practices*](https://learn.microsoft.com/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming)
- [Stephen Cleary — blog](https://blog.stephencleary.com/)
- [Jimmy Bogard — *Vertical Slice Architecture*](https://www.jimmybogard.com/vertical-slice-architecture/)
- [Jimmy Bogard — *Favour query objects over repositories*](https://www.jimmybogard.com/favor-query-objects-over-repositories/)
- [Jimmy Bogard — blog](https://www.jimmybogard.com/)
- [Andrew Lock — *.NET Escapades*](https://andrewlock.net/)
- [Andrew Lock — *Adding validation to strongly-typed configuration objects*](https://andrewlock.net/adding-validation-to-strongly-typed-configuration-objects-in-asp-net-core/)
- [David Fowler — AspNetCoreDiagnosticScenarios](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/main/AsyncGuidance.md)
- [David Fowler — Twitter / X feed (commentary)](https://twitter.com/davidfowl)
- [Steve Smith (Ardalis) — Clean Architecture template](https://github.com/ardalis/CleanArchitecture)
- [Steve Smith — blog](https://ardalis.com/blog/)
- [Mark Seemann — *Service Locator is an anti-pattern*](https://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/)
- [Mark Seemann — blog (Ploeh)](https://blog.ploeh.dk/)
- [Vladimir Khorikov — Enterprise Craftsmanship](https://enterprisecraftsmanship.com/)
- [Vladimir Khorikov — *Types of CQRS*](https://enterprisecraftsmanship.com/posts/types-of-cqrs/)
- [Vladimir Khorikov — *Functional C#: Handling failures*](https://enterprisecraftsmanship.com/posts/functional-c-handling-failures-input-errors/)
- [Oren Eini (Ayende) — blog](https://ayende.com/blog/)
- [Damian Edwards — talks (.NET Conf)](https://www.youtube.com/results?search_query=damian+edwards+dotnet+conf)
- [Nick Chapsas — YouTube](https://www.youtube.com/@nickchapsas)
- [Khalid Abuhakmeh — blog](https://khalidabuhakmeh.com/)
- [Khalid Abuhakmeh — *Records and EF Core*](https://khalidabuhakmeh.com/dotnet-records-and-entity-framework-core)
- [Steve Gordon — blog](https://www.stevejgordon.co.uk/)
- [Maarten Balliauw — blog](https://blog.maartenballiauw.be/)
- [Eric Lippert — *Fabulous Adventures in Coding*](https://ericlippert.com/)
- [Meziantou — blog](https://www.meziantou.net/)
- [Roslynator](https://github.com/dotnet/roslynator)
- [The Pragmatic Engineer — Monorepo and Microservices](https://blog.pragmaticengineer.com/monorepo-vs-multi-repo/)
- [ThoughtWorks Tech Radar — Monorepo with a suitable build system](https://www.thoughtworks.com/radar/techniques/monorepo-with-a-suitable-build-system)

### Tooling

- [NuGet Home (specs, issues)](https://github.com/NuGet/Home)
- [MSBuild docs](https://learn.microsoft.com/visualstudio/msbuild/msbuild)
- [`dotnet/sdk` issues](https://github.com/dotnet/sdk/issues)
- [`dotnet/runtime` issues](https://github.com/dotnet/runtime/issues)
- [`dotnet/aspnetcore` issues](https://github.com/dotnet/aspnetcore/issues)
- [`dotnet-affected`](https://github.com/leonardochaia/dotnet-affected)
- [MinVer](https://github.com/adamralph/minver)
- [Nerdbank.GitVersioning](https://github.com/dotnet/Nerdbank.GitVersioning)
- [Microsoft.CodeAnalysis.PublicApiAnalyzers](https://github.com/dotnet/roslyn-analyzers/blob/main/src/PublicApiAnalyzers/PublicApiAnalyzers.Help.md)
- [BannedApiAnalyzers](https://github.com/dotnet/roslyn-analyzers/blob/main/src/Microsoft.CodeAnalysis.BannedApiAnalyzers/BannedApiAnalyzers.Help.md)
- [Scrutor](https://github.com/khellang/Scrutor)
- [Mapperly](https://github.com/riok/mapperly)
- [Vogen](https://github.com/SteveDunn/Vogen)
- [StronglyTypedId](https://github.com/andrewlock/StronglyTypedId)
- [OneOf](https://github.com/mcintyre321/OneOf) · [ErrorOr](https://github.com/amantinband/error-or) · [FluentResults](https://github.com/altmann/FluentResults)
- [Ardalis.Specification](https://github.com/ardalis/Specification)
- [NetArchTest](https://github.com/BenMorris/NetArchTest) · [ArchUnitNET](https://github.com/TNG/ArchUnitNET)
- [Verify](https://github.com/VerifyTests/Verify)
- [Testcontainers for .NET](https://dotnet.testcontainers.org/)
- [coverlet](https://github.com/coverlet-coverage/coverlet) · [reportgenerator](https://github.com/danielpalme/ReportGenerator)
- [BenchmarkDotNet](https://benchmarkdotnet.org/)
- [Polly v8](https://www.pollydocs.org/) · [`Microsoft.Extensions.Resilience`](https://learn.microsoft.com/dotnet/core/resilience/)
- [Wolverine](https://wolverinefx.net/) · [MassTransit](https://masstransit.io/) · [NServiceBus](https://docs.particular.net/)
- [EF Core docs](https://learn.microsoft.com/ef/core/)
- [OpenTelemetry .NET](https://opentelemetry.io/docs/instrumentation/net/)
- [Stateless](https://github.com/dotnet-state-machine/stateless)
- [Sprache](https://github.com/sprache/Sprache)
- [Husky.Net](https://github.com/alirezanet/Husky.Net)

### Talks (NDC / .NET Conf)

- [NDC Conferences — YouTube channel](https://www.youtube.com/@NDC)
- [.NET Conf — YouTube playlists](https://www.youtube.com/@dotnet/playlists)
- [Damian Edwards & David Fowler — keynotes (.NET Conf)](https://www.youtube.com/results?search_query=damian+edwards+david+fowler+dotnet+conf)
- [Jimmy Bogard — *Vertical Slice Architecture* talk (NDC)](https://www.youtube.com/results?search_query=jimmy+bogard+vertical+slice+architecture)
- [Vladimir Khorikov — *Unit Testing Principles* (NDC)](https://www.youtube.com/results?search_query=vladimir+khorikov+unit+testing+ndc)
- [Mark Seemann — *Functional architecture: the pits of success* (NDC)](https://www.youtube.com/results?search_query=mark+seemann+functional+architecture+pits+of+success)
