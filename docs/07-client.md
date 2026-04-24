# Client-side .NET 10: Blazor & MAUI — Opinionated Best Practices

> Target: .NET 10 LTS (released Nov 2025, C# 14). Applies to Blazor Web App (unified model) and .NET MAUI 10. Do / Don't. No fluff.

---

## TL;DR (read this first)

- **Blazor Web App is the default template** since .NET 8. Stop creating separate "Blazor Server" and "Blazor WASM" projects. Pick render modes per-component.
- **Interactive Auto** is the headline mode but the most complex. Default to **Static SSR + islands of Interactive Server** unless you have a reason not to.
- **MAUI 10 is an incremental quality release** — fewer bugs, faster startup (XAML source generators, experimental CoreCLR on Android), `TableView`/`MessagingCenter`/`Compatibility.Layout` out. Not a re-architecture.
- **CommunityToolkit.Mvvm is non-negotiable** for MAUI. Stop hand-rolling `INotifyPropertyChanged`.
- **Both stacks have real rough edges.** Section at the end is honest about when to pick React / SwiftUI / Kotlin / RN instead.

---

## Part 1 — Blazor (unified Web App model)

### 1. Render modes — pick deliberately, per component

Four modes in .NET 8+:

| Mode | Where runs | Interactive? | First paint | Payload | Notes |
|---|---|---|---|---|---|
| **Static SSR** | Server | No | Fastest | ~0 (HTML only) | Default. Use for marketing, content, forms-mostly pages. |
| **Interactive Server** | Server over SignalR | Yes | Fast | Small (~100KB blazor.web.js) | Every interaction is a WebSocket roundtrip. Latency-sensitive. |
| **Interactive WebAssembly** | Browser | Yes | Slow (download runtime) | Multi-MB (first load) | Offline-capable, zero server state per user. |
| **Interactive Auto** | Server → WASM | Yes | Fast via Server, then "upgrades" to WASM in background | Both costs eventually | Best UX, highest complexity. Requires code to run *in both environments*. |

**Do**
- Default new pages to **Static SSR**. Add `@rendermode InteractiveServer` only on the interactive island (a form, a chart).
- Set rendermode at the **component tag site** (`<Counter @rendermode="InteractiveServer" />`) when you want granular control, or with `@rendermode` inside the component when it's always interactive.
- For Interactive Auto, keep shared components in a **`.Client` RCL** — code must run on both server and WASM. Any server-only API (EF Core, file I/O, secrets) is forbidden there.
- Remember `prerender:false` on rendermode when prerendering causes double-fetch pain you can't solve with `[PersistentState]`.

**Don't**
- Don't set `@rendermode InteractiveServer` on `App.razor` or the layout unless you *want* a SignalR circuit for every page. This is the #1 Blazor perf mistake.
- Don't mix Interactive Server and Interactive WebAssembly inside the same interactive subtree — the rules are subtle and you'll get runtime errors.
- Don't reach for Interactive Auto unless your team can maintain strictly portable components.

### 2. Streaming rendering, enhanced navigation, enhanced forms

- **`@attribute [StreamRendering]`**: flush initial HTML with placeholders, then stream updates as async data resolves. Use on any Static SSR page whose main content needs >100ms of I/O. Net-zero cost if the page is fast; big perceived-perf win if it isn't.
- **Enhanced navigation** (default in Blazor Web App): intercepts `<a>`/forms, fetches via `fetch`, patches DOM. Feels SPA-like without any interactivity. Disable per-link with `data-enhance-nav="false"` when you need a full reload (auth redirects, cross-origin).
- **Enhanced form handling**: `<form data-enhance>` or `<EditForm Enhance>` — posts without full reload. Works on Static SSR. Use it; it's free.

**Don't** stream-render above-the-fold content the user immediately interacts with — jank.

### 3. State management — pragmatism over purity

Reach for, in order:

1. **Plain `[Parameter]` / `[CascadingParameter]`** — 80% of cases.
2. **Scoped DI service** holding state + `event Action StateChanged` that components subscribe to. Unsubscribe in `Dispose`. This is the idiomatic Blazor "store".
3. **`PersistentComponentState`** / `[PersistentState]` (new in .NET 10) — persist prerender state into interactive mode so you don't double-fetch.
4. **Fluxor / BlazorState** — only if you already know you want Redux semantics (time-travel debugging, strict unidirectional flow, large team needing ceremony). Otherwise skip. Most Blazor apps that adopt Fluxor regret the boilerplate.

**Do**: memorize **"scoped in Server = per-circuit; scoped in WASM = per-user-app lifetime; in Interactive Auto = both, and they aren't the same instance."** This trips everyone up.

**Don't**: store state in static fields. Don't assume `HttpContext` is usable outside Static SSR — it isn't in Interactive modes.

### 4. Component design

- **Parameters** for inputs. **`EventCallback<T>`** for outputs (handles `StateHasChanged` for you — don't use plain `Action`).
- **`RenderFragment`** / **`RenderFragment<T>`** for templating. Generic components (`@typeparam T`) for reusable lists, tables, pickers.
- **Cascading values** for ambient concerns (theme, current user, form context). Overuse = hidden coupling.
- Implement `IAsyncDisposable` when you subscribe to events, start timers, or hold `DotNetObjectReference`. Forgetting this leaks circuits in Interactive Server.
- `[StreamRendering]` on components with async data under Static SSR.
- `ShouldRender()` override — last resort. First try `@key`, `EventCallback`, and splitting components.

### 5. Auth

- **Server-rendered (Static SSR / Interactive Server)**: cookie auth via ASP.NET Core Identity or OIDC. `AuthenticationStateProvider` is wired up. Works normally.
- **Interactive WebAssembly**: use `Microsoft.AspNetCore.Components.WebAssembly.Authentication` + `AddMsalAuthentication` or OIDC. Persist state with `PersistentAuthenticationStateProvider` so prerender + interactive see the same user.
- **Interactive Auto**: both of the above — use the template; do not invent your own.
- **Token storage**: **do not put JWTs in `localStorage`** if you can avoid it (XSS = total account takeover). Preferred pattern: **BFF with HttpOnly, Secure, SameSite=Lax cookies**; the WASM app calls same-origin APIs, the server does token exchange. See `Microsoft.Identity.Web` + YARP or Duende.BFF.
- `<AuthorizeView>` for UI gating; `[Authorize]` on `@page` components; **always** re-check on the server — UI gating is not security.

### 6. Performance

- **`<Virtualize>`** for any list > ~100 rows. Pair with `ItemSize` when rows are uniform.
- **`@key`** on every list item bound to a stable ID. Without it, Blazor's diff rebinds wrong components and you'll chase phantom bugs.
- **Prerendering**: on by default; turn off per-component (`prerender:false`) when the prerender roundtrip is wasted (auth-gated content, user-specific data without cache).
- **WASM**: enable **AOT** for compute-heavy; expect 2–3× larger download. Use **lazy-loaded assemblies** (`BlazorWebAssemblyLazyLoad`) for rarely-used features. Enable **trimming** (default) and watch for reflection-based libs breaking.
- **Interactive Server**: the circuit is the perf budget — keep component state small, avoid giant parameter trees, don't re-render the world on every keystroke (debounce inputs).
- .NET 10 shipped the big WASM boot-time and JS payload reductions — take them by upgrading, not by clever code.

### 7. JS interop

- **`IJSRuntime.InvokeAsync<T>`** for one-offs. **JS module isolation** (`IJSRuntime.InvokeAsync<IJSObjectReference>("import", "./_content/MyLib/foo.js")`) for component-scoped JS — dispose the reference.
- **`DotNetObjectReference`** for JS→.NET callbacks; dispose in `IAsyncDisposable`.
- Interop crosses a marshalling boundary — each call has cost. **Batch**: one call returning an array is cheaper than 100 calls.
- If your component is 60% JS, **write it as a JS library and wrap it** — don't pretend it's .NET. Chart.js, Monaco, Leaflet all belong in JS-land.

### 8. Forms & validation

- `<EditForm Model>` + `<DataAnnotationsValidator />` + `<ValidationSummary/>` / `<ValidationMessage For>` is the baseline.
- For anything beyond trivial, **FluentValidation** via `Blazored.FluentValidation` (community) or the built-in `IValidator<T>` pattern. DataAnnotations doesn't express cross-field rules well.
- Use `[SupplyParameterFromForm]` + `[Bind]` on Static SSR forms — they work without any interactivity.
- .NET 10 adds per-component async validation hooks — use them for "is username available" style checks; don't hack it with `OnInput`.

### 9. Testing Blazor

- **bUnit** (Egil Hansen) — de facto unit-test framework. Great for: rendering, parameters, events, cascading values, fakes for `IJSRuntime`/`NavigationManager`/`AuthenticationStateProvider`.
- bUnit does **not** run a real browser — no real JS, no real CSS layout, no SignalR. Don't test rendermode-switching behavior in bUnit.
- For full E2E: **`WebApplicationFactory<TProgram>` + Playwright** (or `Microsoft.Playwright`). Test Interactive Server via real WebSocket; test WASM via real browser.
- Snapshot-test markup sparingly — Blazor's rendered HTML changes across versions.

---

## Part 2 — MAUI (.NET 10)

### 10. Architecture — MVVM with CommunityToolkit.Mvvm, always

```csharp
public partial class TodoViewModel : ObservableObject
{
    [ObservableProperty] private string _title = "";
    [ObservableProperty] private bool _isBusy;

    [RelayCommand(CanExecute = nameof(CanSave))]
    private async Task SaveAsync() { /* ... */ }

    private bool CanSave() => !IsBusy && !string.IsNullOrWhiteSpace(Title);
}
```

**Do**: use source-gen `[ObservableProperty]`, `[RelayCommand]`, `[ObservableObject]`. Zero boilerplate, no reflection, AOT-friendly. Use `WeakReferenceMessenger` (now that `MessagingCenter` is internal in .NET 10).

**Don't**: hand-roll `INotifyPropertyChanged`, don't use `SetProperty` manually, don't use `ICommand` implementations from blog posts circa 2015. Don't put logic in code-behind — ViewModel-first.

### 11. Shell & navigation

- **`Shell` is the default** for any app with tabs/flyout/hierarchical navigation. URL-based routing (`//home/details?id=42`) is the selling point.
- **`NavigationPage`** only for apps that are genuinely a simple stack (wizard, small utility).
- Register routes with `Routing.RegisterRoute("details", typeof(DetailsPage))`.
- Pass data via **`[QueryProperty]`** or Shell navigation parameters (`Dictionary<string, object>`) — **not** via constructor DI hacks or static singletons.
- Prefer typed navigation extension methods over raw strings; strings rot.

### 12. DI in MAUI — know the lifetime lies

`MauiAppBuilder.Services` is a standard `IServiceCollection`, **but**:

- **There is no per-request scope** (no HTTP pipeline). "Scoped" lifetime effectively behaves like singleton at app level unless you create your own scope via `IServiceScopeFactory`.
- For per-page VMs: register **transient**, resolve via constructor injection in the page.
- **Keyed services** (.NET 8+) work — useful for "give me the iOS impl of IPhotoPicker".
- Register `HttpClient` via `AddHttpClient<T>()`; don't `new HttpClient()` (socket exhaustion on Android is real).

### 13. Platform code

Preference order:

1. **Partial classes per platform** (`MyThing.cs` + `Platforms/Android/MyThing.android.cs`) — cleanest, best tooling.
2. **`OperatingSystem.IsAndroidVersionAtLeast(…)`** / `IsIOS()` runtime checks — fine for small divergences.
3. **`#if ANDROID`** conditional compilation — only as last resort; messes with IntelliSense and code coverage.

Write a platform-agnostic interface in shared code, inject the right impl via DI (keyed or per-platform registration in `MauiProgram`).

### 14. Performance

- MAUI is **handlers-based**, not renderers. If a tutorial mentions `CustomRenderer`, it's Xamarin-era — don't copy it.
- **`CollectionView`**, not `ListView`. `ListView` is legacy; `TableView` is **deprecated in .NET 10** (use `CollectionView` with grouping).
- Virtualize: `CollectionView` does it; set `ItemsUpdatingScrollMode` correctly for chat-style UIs.
- Images: use `FileImageSource`/`UriImageSource` with caching enabled (default true), keep source sizes reasonable — MAUI will not magically downsample your 4K PNG.
- **AOT / Full AOT / NativeAOT**: iOS always AOT (Apple mandate). Android .NET 10 has **experimental CoreCLR** runtime option — faster but not production-default. Use `PublishAot` for iOS `Release`; measure before enabling everywhere.
- Startup: .NET 10's **XAML source generators** compile XAML at build, not runtime — ensure they're enabled (default in new templates). If you migrated from 8/9, verify your csproj isn't pinned to runtime XAML.

### 15. State & data

- **`sqlite-net-pcl`** — the pragmatic choice for on-device storage. Small, works, zero ceremony.
- **EF Core works on MAUI** (SQLite provider). Use **`IDbContextFactory<T>`**, not singleton `DbContext` — `DbContext` is not thread-safe and mobile apps are concurrent. Expect larger app size and slower cold start than sqlite-net; choose accordingly.
- **`SecureStorage`** for tokens/secrets — Keychain on iOS, Keystore on Android. Never `Preferences` for secrets.
- **MSAL.NET** for Entra ID. Enable the **broker** (`WithBroker(true)`) on Android/iOS — uses Microsoft Authenticator / Company Portal, supports conditional access and device-based policies. Without broker, you'll fail CA requirements in enterprise tenants.
- Tokens go in MSAL's cache (backed by SecureStorage via the platform-specific extensions); **don't roll your own**.

### 16. Lifecycle

- **App lifecycle**: `CreateMauiApp` → `App.OnStart` / `OnSleep` / `OnResume`. Window events (`Activated`, `Deactivated`, `Stopped`, `Resumed`) on `Window`.
- **Page lifecycle**: `OnAppearing`, `OnDisappearing`, `OnNavigatedTo`, `OnNavigatedFrom`. Fire order is not always what you expect across platforms — test on both.
- **Background execution** on mobile is **limited and platform-specific**. MAUI has no cross-platform background-task API; use `Platforms/Android/...` for `WorkManager`, `Platforms/iOS/...` for `BGTaskScheduler`. Don't promise "runs every 15 minutes" — iOS will laugh at you.
- Push: use **Firebase** (Android) + **APNs** (iOS) directly, or a service like Azure Notification Hubs. There is no built-in cross-platform push in MAUI.

### 17. Testing MAUI

- **ViewModels are plain C#** — unit-test them with xUnit/NUnit. If your VM can't be tested without a `Page`, refactor.
- **UI tests**: `.NET MAUI UITest` (Appium-based, the Xamarin.UITest successor, maintained by the MAUI team) or raw **Appium**. Slow, flaky like all mobile E2E — use sparingly for critical flows (login, checkout).
- **Snapshot/visual testing**: no first-party story. Third-party (e.g., `PlatformSpecific.Snapshot`) exists but is niche.
- CI: run VM tests on every PR; run UITest on a nightly schedule against real devices or a device farm.

### 18. CI/CD

- **Visual Studio App Center is retired (March 2025).** Don't start new projects on it.
- Practical current options:
  - **Azure DevOps Pipelines** or **GitHub Actions** for build + test + sign. Microsoft-hosted macOS runners for iOS builds.
  - **fastlane** for the actual store-submission dance (match for certs, pilot for TestFlight, supply for Play). Still the gold standard.
  - **Codemagic** / **Bitrise** as hosted alternatives if you don't want to run your own macOS agents.
  - **App Center Diagnostics/Analytics replacements**: **App Center Analytics → App Insights**; **Diagnostics (crashes) → Sentry, Firebase Crashlytics, or Visual Studio App Center's successor features folded into Azure Monitor**. Pick Sentry if you want one tool for mobile + web.
- **Code signing**: store certs/keys in Key Vault or fastlane match (encrypted git repo). Never commit `.p12` files. Use per-environment signing identities.
- **Store submission**: automate with fastlane (`pilot`, `supply`) or Azure DevOps App Store/Google Play tasks. Manual submission from a developer laptop is a compliance smell.

---

## Honest section — rough edges, and when to pick something else

### Blazor rough edges (call them out)

- **Interactive Auto is conceptually elegant, operationally tricky.** You have *two* runtimes executing the same component, *two* DI scopes, and state that has to survive the handoff. Debugging "why does this only break after the Auto upgrade?" is a specific kind of pain.
- **SignalR circuits are stateful.** Scale-out requires sticky sessions + Redis backplane; a memory leak in a component leaks until the circuit disconnects. Horizontal scaling is not free.
- **WASM cold start is still measured in seconds** on a cold cache, even after .NET 10's wins. For public-facing consumer sites where bounce rate matters, this is real.
- **Ecosystem depth.** Compared to React, the component ecosystem (especially free/OSS) is thin. Paid component suites (Telerik, Syncfusion, MudBlazor for free) dominate.
- **Mobile performance of Blazor Hybrid/`BlazorWebView`** is "good enough" for LOB — don't ship a consumer game in it.

**Reach for React/Vue/Svelte instead when**:
- You need a huge existing JS component ecosystem (charts, editors, 3D, design systems).
- Your team is already JS-native; forcing them into C# has a real productivity cost.
- You need SSR + edge deployment (Cloudflare/Vercel-style); Blazor SSR assumes an ASP.NET Core server.
- Bundle size / cold start is a hard business metric.

### MAUI rough edges (call them out)

- **.NET 10 is "quality-focused"** because .NET 8 and 9 had stability issues. Some corners (`CollectionView` layout on iOS, keyboard handling, focus) were historically buggy. MAUI 10 is the first release most people will call "solid" — but verify on your own device matrix.
- **Hot Reload** is better than it was, still not as smooth as SwiftUI Preview or Flutter hot reload.
- **Single-window assumption** is deeply baked in; multi-window (iPad, desktop) works but is second-class.
- **Community size** is smaller than Flutter / React Native. Stack Overflow answers are often Xamarin.Forms-era and subtly wrong.
- **Android build times** can be brutal; CoreCLR-on-Android (experimental in .NET 10) may help eventually.

**Reach for native (SwiftUI / Jetpack Compose) instead when**:
- You're building one platform only (obviously).
- You need bleeding-edge OS features the day they ship (Live Activities, widgets with full fidelity, CarPlay, WearOS).
- UX polish is the product — pixel-perfect, platform-idiomatic animations.

**Reach for Flutter / React Native instead when**:
- Your team isn't .NET-native and has no reason to become so.
- You want the largest cross-platform mobile community + plugin ecosystem (Flutter/RN both dwarf MAUI).
- You're OK with "not quite native" look-and-feel in exchange for faster velocity.

**MAUI is genuinely a good fit when**:
- You're a .NET shop with an existing ASP.NET Core backend and shared domain code (DTOs, validation, business logic in netstandard/.NET 10 libraries).
- You're building internal / LOB apps where productivity > pixel-perfection.
- You need WinUI + mobile from one codebase (MAUI's Windows story is unique).

---

## Sources

### Authoritative
- **Microsoft Learn — Blazor** (render modes, rendering, auth): <https://learn.microsoft.com/aspnet/core/blazor/> (target `?view=aspnetcore-10.0`)
- **Microsoft Learn — Blazor render modes**: <https://learn.microsoft.com/aspnet/core/blazor/components/render-modes?view=aspnetcore-10.0>
- **Microsoft Learn — .NET MAUI**: <https://learn.microsoft.com/dotnet/maui/>
- **What's new in .NET MAUI for .NET 10**: <https://learn.microsoft.com/dotnet/maui/whats-new/dotnet-10>
- **What's new in ASP.NET Core 10.0**: <https://learn.microsoft.com/aspnet/core/release-notes/aspnetcore-10.0>
- **dotnet/aspnetcore** (source + issues): <https://github.com/dotnet/aspnetcore>
- **dotnet/maui** (source + issues + release notes): <https://github.com/dotnet/maui>
- **CommunityToolkit.Mvvm**: <https://learn.microsoft.com/dotnet/communitytoolkit/mvvm/>
- **.NET Blog** (team announcements): <https://devblogs.microsoft.com/dotnet/>
- **Microsoft Identity (MSAL.NET) docs** — broker, mobile: <https://learn.microsoft.com/entra/msal/dotnet/>

### Community / practitioners
- **Steve Sanderson** — Blazor creator; blog <https://blog.stevensanderson.com/> and conference talks (NDC, .NET Conf).
- **Daniel Roth** — ASP.NET PM; announcements and deep-dives on the .NET Blog.
- **Chris Sainty** — Blazor author/community; <https://chrissainty.com/>; creator of `Blazored.*` (FluentValidation, Toast, LocalStorage).
- **Egil Hansen** — creator of **bUnit**: <https://bunit.dev/>; GitHub <https://github.com/bUnit-dev/bUnit>.
- **Khalid Abuhakmeh** (JetBrains) — Blazor / ASP.NET Core how-tos: <https://khalidabuhakmeh.com/>.
- **Jonathan Peppers** — MAUI performance / Android internals; <https://github.com/jonathanpeppers>, .NET Blog posts on MAUI startup and size.
- **Gerald Versluis** — MAUI advocate; YouTube (deep-dive videos), <https://blog.verslu.is/>.
- **James Montemagno** — Xamarin/MAUI; <https://montemagno.com/>; maintains MAUI community toolkits and plugins.
- **David Ortinau** — MAUI PM; .NET Blog MAUI release posts, conference demos.
- **.NET MAUI Community Toolkit**: <https://github.com/CommunityToolkit/Maui>.
- **MudBlazor** (OSS Blazor component lib): <https://mudblazor.com/>.

Cross-reference all version-specific claims against the Microsoft Learn pages pinned to `view=aspnetcore-10.0` / the .NET 10 MAUI "what's new" doc — preview-era blog posts (early/mid 2025) drift from the final API.
