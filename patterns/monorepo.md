# .NET monorepo strategy

How to lay out a multi-team .NET monorepo without paying the chaos tax.

> Severity legend (used inline): 🔴 hard fail in review, 🟡 needs justification,
> 🟢 recommended.

> Companion pages: [`anti-patterns.md`](./anti-patterns.md),
> [`patterns.md`](./patterns.md), shared [`references.md`](./references.md).

---

## 1.1 When a monorepo is the right call

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

## 1.2 Repo layout

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

## 1.3 Solution files: `.slnx` vs `.sln` vs `.slnf` vs traversal

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

## 1.4 Central Package Management — beyond the basics

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

## 1.5 ProjectReference vs PackageReference — the internal NuGet boundary

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

## 1.6 Multi-targeting matrix

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

## 1.7 Build performance

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

## 1.8 Testing in monorepos

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

## 1.9 Versioning + release

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

## 1.10 CI/CD

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

## 1.11 Code ownership and architectural enforcement

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

## 1.12 Refactoring at scale

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

## 1.13 Trade-offs (be honest)

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

---

See [`references.md`](./references.md) for the curated source list shared across the patterns/ pages.
