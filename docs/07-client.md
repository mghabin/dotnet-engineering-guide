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
| **Interactive Auto** | Server first, WASM later | Yes | Fast via Interactive Server on the *first* visit | Both costs eventually | First visit runs interactively on the server while the WebAssembly bundle downloads in the background; **subsequent visits** (after the bundle is cached) run on WebAssembly. There is **no in-place handoff inside the same request**. Requires code to run *in both environments*. |

**Do**
- Default new pages to **Static SSR**. Add `@rendermode InteractiveServer` only on the interactive island (a form, a chart).
- Set rendermode at the **component tag site** (`<Counter @rendermode="InteractiveServer" />`) when you want granular control, or with `@rendermode` inside the component when it's always interactive.
- For Interactive Auto, keep shared components in a **`.Client` RCL** — code must run on both server and WASM. Any server-only API (EF Core, file I/O, secrets) is forbidden there.
- Disable prerendering with the constructor argument (`new InteractiveServerRenderMode(prerender: false)` / `new InteractiveWebAssemblyRenderMode(prerender: false)` / `new InteractiveAutoRenderMode(prerender: false)`) when prerendering causes double-fetch pain you can't solve with `[PersistentState]`.
- Set a global render mode by applying `@rendermode` to the `Routes` and `HeadOutlet` components in `App.razor` — that's the supported "everything interactive" switch.

**Don't**
- Don't set `@rendermode InteractiveServer` on `App.razor` itself or its `<HeadContent>` — applying a render mode to the **App root component** is explicitly **not supported**; use `Routes` + `HeadOutlet` instead. (And avoid making the whole app interactive unless you really want a SignalR circuit for every page — it's the #1 Blazor perf mistake.)
- Don't try to use render modes in a **standalone Blazor WebAssembly** app — render modes are a Blazor Web App feature only; standalone WASM is always client-rendered.
- Don't mix Interactive Server and Interactive WebAssembly inside the same interactive subtree — the rules are subtle and you'll get runtime errors.
- Don't reach for Interactive Auto unless your team can maintain strictly portable components.
- Don't describe Interactive Auto as a same-request server→WASM "upgrade." On the first visit the server keeps rendering interactively; only later visits actually execute on WebAssembly.

#### Decision table — which Blazor render mode?

| Need | Static SSR | Interactive Server | Interactive WebAssembly | Interactive Auto |
|---|---|---|---|---|
| Lowest first paint, SEO content | ✅ | ⚠️ (extra JS) | ❌ (multi-MB) | ⚠️ first visit OK, later visits like WASM |
| Real-time UI without per-keystroke latency | ❌ | ⚠️ network-bound | ✅ | ✅ once on WASM |
| Offline / poor network | ❌ | ❌ (circuit dies) | ✅ | ⚠️ only after bundle cached |
| No per-user server state | ✅ | ❌ (circuit per tab) | ✅ | ⚠️ server state on first visit |
| Access to server resources (EF Core, secrets) directly in component | ✅ | ✅ | ❌ (call API) | ❌ in shared `.Client` RCL |
| Authorization enforced by component code | ❌ — never | ❌ — never | ❌ — never | ❌ — never (server/API only) |
| Operational complexity | Low | Medium (sticky sessions, backplane) | Medium (publish bundle, version cache) | **High** (two runtimes, two DI scopes) |

**Sources:** Microsoft Learn — [ASP.NET Core Blazor render modes](https://learn.microsoft.com/aspnet/core/blazor/components/render-modes?view=aspnetcore-10.0); [Prerender ASP.NET Core Razor components](https://learn.microsoft.com/aspnet/core/blazor/components/prerender?view=aspnetcore-10.0).

### 2. Streaming rendering, enhanced navigation, enhanced forms

- **`@attribute [StreamRendering]`**: flush initial HTML with placeholders, then stream updates as async data resolves. Use on any Static SSR page whose main content needs >100ms of I/O. Net-zero cost if the page is fast; big perceived-perf win if it isn't.
- **Enhanced navigation** (default in Blazor Web App): intercepts `<a>`/forms, fetches via `fetch`, patches DOM. Feels SPA-like without any interactivity. Requires the page to be served by a Blazor endpoint (`MapRazorComponents<App>()`) **and** `blazor.web.js` to be on the page. Opt out per-link with `data-enhance-nav="false"`, per-form with `data-enhance="false"`, or globally via `Blazor.disableDomPreservation = true`. Internal SPA-style transitions skip prerendering.
- **Enhanced form handling**: `<form data-enhance>` or `<EditForm Enhance>` — posts without full reload, patches the DOM. Works on Static SSR — but with caveats:
  - Only works when the form posts to a **Blazor endpoint** (a routable Razor component handling the POST). Plain MVC/minimal-API endpoints don't get the enhanced behavior.
  - Every Static SSR form needs a unique **`FormName`** (or `@formname` on a plain `<form>`) so the model binder knows which form posted.
  - **Antiforgery**: `<EditForm>` injects the antiforgery token automatically. A plain `<form>` must include `<AntiforgeryToken />` explicitly. `AddRazorComponents` auto-registers antiforgery services; the template's `Program.cs` calls `UseAntiforgery()` **after** `UseHttpsRedirection` (and after `UseAuthentication`/`UseAuthorization`) — keep that order. Missing tokens = 400. To attach the token to a same-origin web API call from an interactive component, inject `AntiforgeryStateProvider` and read `GetAntiforgeryToken()`.

**Don't** stream-render above-the-fold content the user immediately interacts with — jank.

**Sources:** Microsoft Learn — [Blazor static server-side rendering / streaming](https://learn.microsoft.com/aspnet/core/blazor/components/render-modes?view=aspnetcore-10.0#static-server-side-rendering-static-ssr); [Enhanced navigation and form handling](https://learn.microsoft.com/aspnet/core/blazor/fundamentals/routing?view=aspnetcore-10.0#enhanced-navigation-and-form-handling); [ASP.NET Core Blazor forms overview](https://learn.microsoft.com/aspnet/core/blazor/forms/?view=aspnetcore-10.0).

### 3. State management — pragmatism over purity

Reach for, in order:

1. **Plain `[Parameter]` / `[CascadingParameter]`** — 80% of cases for *passing data down a known tree*.
2. **Scoped DI service** holding state + `event Action StateChanged` that components subscribe to. Unsubscribe in `Dispose`. This is the idiomatic Blazor "store" for cross-component mutable state.
3. **`PersistentComponentState`** / **`[PersistentState]`** — persist prerender state into interactive mode so you don't double-fetch. The `[PersistentState]` attribute (.NET 9) decorates a **public** property and replaces the manual `RegisterOnPersisting` + `TryTakeFromJson` pattern; the property must be public because the framework uses reflection (trimming/source-gen implication). Default serialization is `System.Text.Json`. **.NET 10** adds `AllowUpdates` (whether the parameter receives updates beyond the initial restored value) and `RestoreBehavior` (controls how/when data is restored, e.g., across enhanced-navigation transitions).
4. **Fluxor / BlazorState** — only if you already know you want Redux semantics (time-travel debugging, strict unidirectional flow, large team needing ceremony). Otherwise skip. Most Blazor apps that adopt Fluxor regret the boilerplate.

**Cascading values are for ambient context, not a global store.** Use them for theme, auth snapshot, current `EditContext`, request culture — values that are *read-mostly* and naturally apply to a subtree. Do **not** use cascading parameters as a general-purpose mutable application store: every change re-renders every consumer, dependencies become invisible, and testing degrades. Route mutable cross-component state through a scoped store service (option 2) or an explicit state container (option 4).

**Do**: memorize **"scoped in Server = per-circuit; scoped in WASM = per-user-app lifetime; in Interactive Auto = both, and they aren't the same instance."** This trips everyone up.

**Don't**: store state in static fields. Don't assume `HttpContext` is usable outside Static SSR — it isn't in Interactive modes.

> **⚠️ Per-circuit hazards (Interactive Server)**
> - **One circuit per browser tab.** Open the same app in two tabs = two scoped service instances, two copies of state, no automatic sync.
> - **Refresh or new tab = brand-new scoped store.** Anything not persisted (URL, server DB, browser storage, `PersistentComponentState`) is gone.
> - **Disconnect/reconnect**: brief network drops attempt reconnection to the same circuit; longer drops kill it and the user gets a fresh circuit (and fresh state) on next interaction. Default circuit retention is short — tune with `CircuitOptions` if needed, but don't rely on it for durability.
> - **Decision rule**: if state must survive refresh, route, or device → persist it. URL/query string for shareable view state; server DB for user data; `ProtectedBrowserStorage` (local/session) for client-only preferences; `PersistentComponentState` for prerender → interactive handoff.

**Sources:** Microsoft Learn — [ASP.NET Core Blazor state management](https://learn.microsoft.com/aspnet/core/blazor/state-management?view=aspnetcore-10.0); [Prerendered state persistence](https://learn.microsoft.com/aspnet/core/blazor/components/prerender?view=aspnetcore-9.0#persist-prerendered-state); [SignalR guidance for Blazor — circuit lifetime](https://learn.microsoft.com/aspnet/core/blazor/fundamentals/signalr?view=aspnetcore-10.0).

### 4. Component design

- **Parameters** for inputs. **`EventCallback<T>`** for outputs (handles `StateHasChanged` for you — don't use plain `Action`).
- **`RenderFragment`** / **`RenderFragment<T>`** for templating. Generic components (`@typeparam T`) for reusable lists, tables, pickers.
- **Cascading values** for *ambient* concerns only (theme, current user snapshot, `EditContext`). Not a global mutable store — see §3.
- Implement `IAsyncDisposable` when you subscribe to events, start timers, or hold `DotNetObjectReference`. Forgetting this leaks circuits in Interactive Server.
- `[StreamRendering]` on components with async data under Static SSR.
- `ShouldRender()` override — last resort. First try `@key`, `EventCallback`, and splitting components.

### 5. Auth

> **🚨 Security boundary — read this before writing a single `<AuthorizeView>`.**
> In **Interactive WebAssembly** and **Interactive Auto**, the browser holds a *projection* of the authentication state. The user controls the browser. They can edit memory, swap providers, and lie about claims. **Every authorization decision that matters MUST be enforced on the server or the API.** `<AuthorizeView>`, `[Authorize]` on a Razor component, and any `AuthenticationStateProvider` check on the client are **UI gating only** — they hide buttons, they do not protect data.

- **Server-rendered (Static SSR / Interactive Server)**: cookie auth via ASP.NET Core Identity or OIDC. `AuthenticationStateProvider` is wired up by the template. Works normally because rendering happens on the server.
- **Interactive WebAssembly (standalone)**: use `Microsoft.AspNetCore.Components.WebAssembly.Authentication` + `AddMsalAuthentication` or OIDC. The WASM app calls APIs with bearer tokens.
- **Blazor Web App with WebAssembly or Auto interactivity (.NET 9+)**: serialize the server's auth state down to the WebAssembly client so both runtimes see the same user without an extra round-trip.

  ```csharp
  // Server Program.cs
  builder.Services.AddRazorComponents()
      .AddInteractiveServerComponents()
      .AddInteractiveWebAssemblyComponents()
      .AddAuthenticationStateSerialization();   // .NET 9+
  ```

  ```csharp
  // .Client Program.cs
  builder.Services.AddAuthorizationCore();
  builder.Services.AddCascadingAuthenticationState();
  builder.Services.AddAuthenticationStateDeserialization(); // .NET 9+
  ```

  This pair replaces the older hand-rolled `PersistentAuthenticationStateProvider` glue. Use the Blazor Web App template's "Individual Accounts" option to scaffold it correctly. **By default only the `name` and `role` claims are serialized** to the WASM client — opt in to additional claims via the `AddAuthenticationStateSerialization` options (e.g., `SerializeAllClaims = true`). Don't blindly serialize everything: anything you serialize is visible to the browser.
- **Token storage**: **do not put JWTs in `localStorage`** if you can avoid it (XSS = total account takeover). Preferred pattern: **BFF with HttpOnly, Secure, SameSite=Lax cookies**; the WASM app calls same-origin APIs, the server does token exchange. See `Microsoft.Identity.Web` + YARP or Duende.BFF.
- `<AuthorizeView>` for UI gating; `[Authorize]` on `@page` components for client routing; **always** enforce again on the server/API. UI gating is not security.

**Sources:** Microsoft Learn — [ASP.NET Core Blazor authentication and authorization](https://learn.microsoft.com/aspnet/core/blazor/security/?view=aspnetcore-10.0); [Server-side and WebAssembly authentication state with Blazor Web App](https://learn.microsoft.com/aspnet/core/blazor/security/?view=aspnetcore-10.0#manage-authentication-state-in-blazor-web-apps); [Threat mitigation guidance for ASP.NET Core Blazor interactive server-side rendering](https://learn.microsoft.com/aspnet/core/blazor/security/interactive-server-side-rendering?view=aspnetcore-10.0).

### 6. Performance

- **`<Virtualize>`** for any list > ~100 rows. Pair with `ItemSize` when rows are uniform.
- **`@key`** on every list item bound to a stable ID. Without it, Blazor's diff rebinds wrong components and you'll chase phantom bugs.
- **Prerendering**: on by default; turn off per-component (`prerender:false`) when the prerender roundtrip is wasted (auth-gated content, user-specific data without cache).
- **WASM**: enable **AOT** for compute-heavy; expect 2–3× larger download. Use **lazy-loaded assemblies** (`BlazorWebAssemblyLazyLoad`) for rarely-used features. Enable **trimming** (default) and watch for reflection-based libs breaking.
- **Interactive Server**: the circuit is the perf budget — keep component state small, avoid giant parameter trees, don't re-render the world on every keystroke (debounce inputs).
- .NET 10 shipped the big WASM boot-time and JS payload reductions — take them by upgrading, not by clever code.

#### .NET 10 Blazor changes worth knowing

- **Blazor script is now a static web asset** (`blazor.web.js` / `blazor.server.js`) with automatic compression + fingerprinting (was an embedded resource). If a project has no `.razor` files but still needs the script (e.g., an MVC host loading Razor components), set `<RequiresAspNetWebAssets>true</RequiresAspNetWebAssets>`.
- **BREAKING (WASM): `HttpClient` response streaming is on by default.** `HttpResponseMessage.Content.ReadAsStreamAsync()` returns a `BrowserHttpReadStream` instead of a buffered `MemoryStream` — synchronous `Stream.Read` will throw. Switch to `ReadAsync` / `CopyToAsync`, or opt out via `<WasmEnableStreamingResponse>false</WasmEnableStreamingResponse>` while you migrate consumers.
- **`NavLinkMatch.All` now ignores query string + fragment by default** — links no longer go inactive when only the query/hash changes. Opt out with the `Microsoft.AspNetCore.Components.Routing.NavLink.IgnoreQueryAndFragmentInMatchAll` `AppContext` switch if you depended on the old behavior.
- **`NavigationManager.NavigateTo` no longer scrolls to top on same-page navigation.** Scroll explicitly when you need it.
- **`ReconnectModal`** component now ships in the Blazor Web App template — CSP-friendly, exposes a `components-reconnect-state-changed` event and a "retrying" state. Replace the inline reconnect HTML in upgraded apps.

**Sources:** Microsoft Learn — [What's new in ASP.NET Core 10.0](https://learn.microsoft.com/aspnet/core/release-notes/aspnetcore-10.0?view=aspnetcore-10.0); [Blazor WebAssembly response streaming](https://learn.microsoft.com/aspnet/core/blazor/call-web-api?view=aspnetcore-10.0).

### 7. JS interop

- **`IJSRuntime.InvokeAsync<T>`** for one-offs. **JS module isolation** (`IJSRuntime.InvokeAsync<IJSObjectReference>("import", "./_content/MyLib/foo.js")`) for component-scoped JS — dispose the reference.
- **`DotNetObjectReference`** for JS→.NET callbacks; dispose in `IAsyncDisposable`.
- Interop crosses a marshalling boundary — each call has cost. **Batch**: one call returning an array is cheaper than 100 calls.
- If your component is 60% JS, **write it as a JS library and wrap it** — don't pretend it's .NET. Chart.js, Monaco, Leaflet all belong in JS-land.

### 8. Forms & validation

- `<EditForm Model>` + `<DataAnnotationsValidator />` + `<ValidationSummary/>` / `<ValidationMessage For>` is the baseline. `EditForm` automatically renders an antiforgery token on Static SSR.
- For anything beyond trivial, **FluentValidation** via `Blazored.FluentValidation` (community) or the built-in `IValidator<T>` pattern. DataAnnotations doesn't express cross-field rules well.
- **Static SSR forms** — pick one of:
  - **`<EditForm Model="Model" FormName="addProduct" OnValidSubmit="Submit">`** — preferred. Antiforgery, validation, and model binding all wired up. `FormName` is required and **must be unique per page** so the binder knows which form posted. Initialize the model defensively in `OnInitialized` (`Model ??= new();`) so prerender + post round-trip works.
  - Or a plain `<form method="post" @onsubmit="Submit" @formname="addProduct">` with `[SupplyParameterFromForm] public ProductModel? Product { get; set; }` on the component, plus `<AntiforgeryToken />` inside the form. Razor components do form model binding through `[SupplyParameterFromForm]`; **`[Bind]` (the MVC attribute) is not the SSR forms pattern** — don't reach for it.
- .NET 10 adds per-component async validation hooks — use them for "is username available" style checks; don't hack it with `OnInput`.

**Sources:** Microsoft Learn — [ASP.NET Core Blazor forms overview](https://learn.microsoft.com/aspnet/core/blazor/forms/?view=aspnetcore-10.0); [Blazor forms binding](https://learn.microsoft.com/aspnet/core/blazor/forms/binding?view=aspnetcore-10.0); [Blazor forms validation](https://learn.microsoft.com/aspnet/core/blazor/forms/validation?view=aspnetcore-10.0).

### 9. Testing Blazor

- **bUnit** (Egil Hansen) — de facto unit-test framework. Great for: rendering, parameters, events, cascading values, fakes for `IJSRuntime`/`NavigationManager`/`AuthenticationStateProvider`.
- bUnit does **not** run a real browser — no real JS, no real CSS layout, no SignalR. Don't test rendermode-switching behavior in bUnit.
- For full E2E: **`WebApplicationFactory<TProgram>` + Playwright** (or `Microsoft.Playwright`). Test Interactive Server via real WebSocket; test WASM via real browser.
- Snapshot-test markup sparingly — Blazor's rendered HTML changes across versions.

### 9a. PWA (Progressive Web App)

- **PWA support is a Blazor WebAssembly feature** — `dotnet new blazorwasm --pwa` (or the "Progressive Web Application" checkbox in Visual Studio) scaffolds `manifest.webmanifest`, icons, `service-worker.js` (dev), and `service-worker.published.js` (publish-time, asset-hash-aware via the `ServiceWorkerAssetsManifest` MSBuild property + `ServiceWorker` item).
- **A Blazor Web App's server project cannot itself be a PWA** — the offline/installable story requires a WASM client. The Blazor Web App template has **no built-in PWA toggle**; if you need PWA in a Web App solution, convert the `.Client` (WASM) project manually following the same template steps (manifest, icons, `service-worker.published.js`, MSBuild items).
- All PWA capabilities (offline cache, install prompt, push, background sync) come from **standard browser APIs** (Web Manifest, Service Worker, Push) — there is no Blazor abstraction over them. The `service-worker.published.js` template is the only Blazor-specific piece, and it exists to do **asset-hash-based cache invalidation** so users actually get new bits after a publish.

**Sources:** Microsoft Learn — [Progressive Web Application support in Blazor](https://learn.microsoft.com/aspnet/core/blazor/progressive-web-app?view=aspnetcore-10.0); [Blazor WebAssembly templates](https://learn.microsoft.com/aspnet/core/blazor/tooling?view=aspnetcore-10.0).

### 9b. Blazor Hybrid (boundary note)

- **Blazor Hybrid** renders Razor components **natively in an embedded WebView** — *not* WebAssembly, *not* a browser tab. Hosts: **.NET MAUI** (`Microsoft.AspNetCore.Components.WebView.Maui` → `BlazorWebView`), **WPF** (`...WebView.Wpf`), and **Windows Forms** (`...WebView.WindowsForms`). Components have full .NET access to native device APIs; `BlazorWebViewInitializing` / `BlazorWebViewInitialized` are the WebView config hooks.
- Use it when you want to **share Razor component code between a Blazor Web App and a MAUI app**. Full Hybrid coverage (BlazorWebView APIs, root component config, JS interop quirks) lives in the dedicated Blazor Hybrid docs — out of scope for this chapter.

**Sources:** Microsoft Learn — [ASP.NET Core Blazor Hybrid](https://learn.microsoft.com/aspnet/core/blazor/hybrid/?view=aspnetcore-10.0).

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

- **`Shell` is the default** for any app with tabs/flyout/hierarchical navigation. URL-based routing (`//home/details?id=42`), built-in flyout, tabs, and search handler are the selling points.
- **`NavigationPage`** only for apps that are genuinely a simple stack (wizard, small utility).
- **`FlyoutPage`** exists for *legacy* / non-Shell apps that need a master-detail flyout. **Do not mix `FlyoutPage` with `Shell`** — Shell apps already provide a flyout, and Microsoft documents that combining them is unsupported and will throw at runtime. New apps should use Shell's flyout; only keep `FlyoutPage` when migrating an older Forms/MAUI app where Shell adoption isn't yet feasible.
- Register routes with `Routing.RegisterRoute("details", typeof(DetailsPage))`.
- Pass data via **`[QueryProperty]`** or Shell navigation parameters (`Dictionary<string, object>`) — **not** via constructor DI hacks or static singletons.
- Prefer typed navigation extension methods over raw strings; strings rot.

| Need | `Shell` | `NavigationPage` | `FlyoutPage` |
|---|---|---|---|
| Tabs + flyout + URL routing | ✅ | ❌ | ❌ |
| Simple linear stack only | ⚠️ overkill | ✅ | ❌ |
| Legacy master-detail without Shell | ❌ (don't mix) | ❌ | ✅ legacy only |
| Recommended for new apps (.NET 10) | ✅ | ⚠️ niche | ❌ |

**Sources:** Microsoft Learn — [.NET MAUI Shell overview](https://learn.microsoft.com/dotnet/maui/fundamentals/shell/); [.NET MAUI FlyoutPage](https://learn.microsoft.com/dotnet/maui/user-interface/pages/flyoutpage); [Shell navigation](https://learn.microsoft.com/dotnet/maui/fundamentals/shell/navigation).

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

- **Window lifecycle is the modern API — lead here.** Subscribe to events on the `Window` you create in `App.CreateWindow` (or override the matching `On{EventName}` virtual methods on `Window` for handler-free hooks):
  - **`Created`** — window object exists, before it's shown. Wire up services that need a window handle.
  - **`Activated`** — window has focus / is foregrounded. Resume animations, refresh time-sensitive data.
  - **`Deactivated`** — lost focus but still visible (e.g., user opened another app on desktop, or pulled down notification center on iOS). Pause heavy work, *don't* tear down state.
  - **`Stopped`** — window is no longer visible (backgrounded). Persist state, release expensive resources, stop timers. On mobile, treat this as "the OS may kill us soon."
  - **`Resumed`** — coming back from `Stopped`. Re-acquire resources, reload anything you released.
  - **`Destroying`** — window is going away. Final cleanup.
  - **`Backgrounding`** — **iOS / Mac Catalyst only**. Fires before the OS suspends the app and gives you `BackgroundingEventArgs.State` (a `string`) to stash a small resume payload that survives suspension.
  - **`SizeChanged`** — **desktop only** (Windows + Mac Catalyst). Don't wire it up on mobile.
  - Order on launch: `Created → Activated`. Background round-trip: `Deactivated → Stopped → Resumed → Activated`. Close: `Destroying`.
- **Page lifecycle**: `OnAppearing`, `OnDisappearing`, `OnNavigatedTo`, `OnNavigatedFrom`. Fire order is not always what you expect across platforms — test on both.
- **Legacy `App.OnStart` / `OnSleep` / `OnResume`** still exist for back-compat with Xamarin.Forms-era code, but Microsoft's current MAUI lifecycle guidance is built around **Window** events. Prefer `Window` for any new code; treat the `App`-level methods as a secondary, app-wide hook (e.g., one-time startup telemetry).
- **Platform-specific lifecycle**: when you need APIs the cross-platform events don't expose (e.g., iOS scene delegates, Android `onSaveInstanceState`), use `ConfigureLifecycleEvents` in `MauiProgram` with the platform-specific delegate hooks.
- **Background execution** on mobile is **limited and platform-specific**. MAUI has no cross-platform background-task API; use `Platforms/Android/...` for `WorkManager`, `Platforms/iOS/...` for `BGTaskScheduler`. Don't promise "runs every 15 minutes" — iOS will laugh at you.
- Push: use **Firebase** (Android) + **APNs** (iOS) directly, or a service like Azure Notification Hubs. There is no built-in cross-platform push in MAUI.

**Sources:** Microsoft Learn — [.NET MAUI app lifecycle](https://learn.microsoft.com/dotnet/maui/fundamentals/app-lifecycle); [Window class](https://learn.microsoft.com/dotnet/maui/fundamentals/windows); [Platform lifecycle events](https://learn.microsoft.com/dotnet/maui/platform-integration/configure-lifecycle-events).

### 17. Testing MAUI

- **ViewModels are plain C#** — unit-test them with xUnit/NUnit. If your VM can't be tested without a `Page`, refactor.
- **UI tests**: `.NET MAUI UITest` (Appium-based, the Xamarin.UITest successor, maintained by the MAUI team) or raw **Appium**. Slow, flaky like all mobile E2E — use sparingly for critical flows (login, checkout).
- **Snapshot/visual testing**: no first-party story. Third-party (e.g., `PlatformSpecific.Snapshot`) exists but is niche.
- CI: run VM tests on every PR; run UITest on a nightly schedule against real devices or a device farm.

### 17a. MAUI 9 → MAUI 10 — what actually changed

**Added in .NET MAUI 9** (worth knowing if you're upgrading from 8):
- **`HybridWebView`** control — host arbitrary HTML/JS/CSS with C#↔JS interop. Enables embedding React/Vue widgets inside a MAUI app without going full Blazor Hybrid.
- **`TitleBar`** control — custom title bars on Mac Catalyst + Windows via `Window.TitleBar`.
- **`CollectionView` / `CarouselView` opt-in handlers on iOS / Mac Catalyst** (perf + stability).
- **Minimum targets raised**: iOS 12.2, Mac Catalyst 15.0 (macOS 12.0). Requires **Xcode 16**.
- Workload + per-package NuGet model preserved (you can still pin individual MAUI packages).

**Changed / removed in .NET MAUI 10**:
- **.NET Aspire integration** — `MauiAppBuilder.AddServiceDefaults()` wires up OpenTelemetry + service discovery + `HttpClient` defaults the same way ASP.NET Core projects do.
- **Animation methods renamed to `*Async`** (`FadeToAsync`, `RotateToAsync`, `ScaleToAsync`, …). Old non-async overloads are deprecated — migrate.
- **`CollectionView` / `CarouselView` iOS + Mac Catalyst handlers are now the default** (the .NET 9 opt-in is gone — opt out only if you hit a regression).
- **Removed**: `Accelerator` (use `KeyboardAccelerator`), `ClickGestureRecognizer` (use `TapGestureRecognizer`), `Compatibility.Layout`. `TableView` / `MessagingCenter` are gone in this release too — see §10 / §14.
- **Editor / Entry on Android** now use `MauiAppCompatEditText` natively (adds `SelectionChanged`).
- **`HybridWebView`** gains an `InvokeJavaScriptAsync` overload without a return type and propagates JS exceptions back into .NET.

**Sources:** Microsoft Learn — [What's new in .NET MAUI for .NET 9](https://learn.microsoft.com/dotnet/maui/whats-new/dotnet-9?view=net-maui-9.0); [What's new in .NET MAUI for .NET 10](https://learn.microsoft.com/dotnet/maui/whats-new/dotnet-10?view=net-maui-9.0).

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

### Decision table — MAUI vs PWA vs Avalonia vs Uno

| Concern | **.NET MAUI** | **PWA (Blazor WASM / JS)** | **Avalonia** | **Uno Platform** |
|---|---|---|---|---|
| Target platforms | iOS, Android, Windows (WinUI), macOS (Catalyst), Tizen | Anywhere with a modern browser; "installable" on most OSes | Windows, macOS, Linux, iOS, Android, browser (WASM) | iOS, Android, Windows, macOS, Linux, browser (WASM) |
| Native API access | ✅ first-class (Essentials, platform partials) | ❌ browser sandbox only (limited Web APIs) | ⚠️ desktop-good; mobile via bindings | ✅ broad, via platform projections |
| Offline | ✅ | ⚠️ service-worker dependent | ✅ | ✅ |
| Desktop depth (Linux first-class?) | ❌ no Linux | ⚠️ runs in browser | ✅ best-in-class for Linux desktop | ✅ |
| UI consistency across platforms | ⚠️ uses native controls (looks native, behaves differently) | ✅ web is web | ✅ pixel-identical custom rendering | ✅ pixel-identical (or native via WinUI) |
| Reuse existing WinUI/UWP XAML | ⚠️ partial | ❌ | ⚠️ different XAML dialect | ✅ designed for it |
| Team skill required | C# + XAML + per-platform quirks | C# + HTML/CSS (or JS) | C# + Avalonia XAML | C# + WinUI/XAML |
| Distribution | App stores (signing, review) | URL + install prompt | Installer / store / package mgr | Store + web |
| Mature for production | ✅ MAUI 10 is the first "solid" release | ✅ for the right use case | ✅ desktop; mobile newer | ✅ |

Rules of thumb:
- **Need real native APIs on iOS/Android + Windows desktop, .NET shop?** → MAUI.
- **Cross-platform reach matters more than native APIs, you can live with the browser sandbox?** → Blazor PWA.
- **Linux desktop is a first-class target, or you want one pixel-identical UI everywhere?** → Avalonia.
- **You have an existing WinUI/UWP codebase you want on the web and mobile?** → Uno.

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
