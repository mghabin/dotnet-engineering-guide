# .NET 10 Performance — Opinionated Best Practices

Dense, opinionated, sourced. Targets **.NET 10 (LTS, Nov 2025)**. Assumes ASP.NET Core / library / service workloads. Read Stephen Toub's "Performance Improvements in .NET 10" before arguing with anything here.

> **The single rule:** measure, don't guess. Every "do" below has a "don't" because someone shipped the don't and regretted it.

---

## 1. Methodology — measure first, always

**Do**
- Start from a **macro metric** that matters: p50/p95/p99 latency, RPS, allocations/sec, CPU%, GC pause time, working set. If you can't tie a micro-win to a macro metric, you're polishing brass.
- For micro-benchmarks: **BenchmarkDotNet (BDN)**. Always. It handles warmup, pilot, JIT tier promotion, statistical analysis, and process isolation. Anything you write with `Stopwatch` in a `for` loop is wrong by default (cold JIT, no warmup, dead-code elimination, tier-0 code).
- For live processes: **`dotnet-counters`** (cheap, EventCounters/Meter), **`dotnet-trace`** (sampled CPU/events, cross-platform), **`dotnet-gcdump`**, **`dotnet-dump`**, **PerfView** (Windows ETW, deepest), **`perf`/LTTng** on Linux, **EventPipe** as the cross-platform substrate. **Visual Studio Profiler** and **JetBrains dotTrace/dotMemory** for GUI-driven analysis.
- **Instrument before you optimize.** Tools above are *consumers*; the signals must exist first. Emit app-level metrics with **`System.Diagnostics.Metrics.Meter`** (OpenTelemetry-compatible) or a custom **`EventSource`** / **`EventCounters`** for: queue length, payload size, allocations per request, cache hit rate, per-stage latency (auth, deserialize, handler, serialize, downstream). `dotnet-counters` and OTel collectors then scrape them; without instrumentation you're profiling blind.
- **Cold-start vs steady-state are different benchmarks.** BDN warms the JIT and reports steady-state — that's the wrong tool for measuring R2R / NativeAOT / source-generator wins, which mostly move *startup*. For startup, run the published binary as a fresh process N times (e.g. 30+ launches), no warmup, and measure time-to-first-request, time-to-`Main`-exit, or time-to-`Application started`. Use `dotnet-trace collect --providers Microsoft-Windows-DotNETRuntime` on the launches and aggregate. Keep both numbers; one win can hide the other regression.
- Isolate variables. One change per benchmark run. Pin CPU governors (`cpupower frequency-set -g performance`), disable turbo or accept the noise, run on quiet hardware, kill antivirus.
- Understand **tiered JIT**: tier-0 = fast compile / slow code; tier-1 = optimized after N calls (default 30). Without warmup you measure tier-0. **Dynamic PGO** (on-by-default since .NET 8, sharper in .NET 9/10) reshuffles tier-1 based on tier-0 instrumentation — so the *shape* of warmup matters.
- Reproduce production: same TFM, same RID, same GC mode (Server vs Workstation), same container limits.

**Don't**
- Don't benchmark in `Debug`. Don't benchmark with a debugger attached. Don't benchmark on a laptop on battery.
- Don't trust a single run. BDN's confidence intervals exist for a reason.
- Don't optimize before you've profiled. ~80% of "obvious" wins move nothing because they're off the hot path.
- Don't report a NativeAOT/R2R/source-gen win using a steady-state BDN number. Show the cold-start histogram.

**Sources:**
- Stephen Toub, "Performance Improvements in .NET 10" — https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/
- BenchmarkDotNet — how it works: https://benchmarkdotnet.org/articles/guides/how-it-works.html
- .NET diagnostic tools overview: https://learn.microsoft.com/dotnet/core/diagnostics/
- `System.Diagnostics.Metrics` & OpenTelemetry .NET: https://learn.microsoft.com/dotnet/core/diagnostics/metrics

---

## 2. Allocations — the silent latency tax

GC time is allocation time deferred. Every `new` on the hot path is a future Gen0 (cheap), Gen1 (less cheap), or Gen2 (expensive, blocking on Workstation GC) collection. Objects ≥ **85,000 bytes** go on the **Large Object Heap (LOH)** — Gen2-collected, not compacted by default (opt-in via `GCSettings.LargeObjectHeapCompactionMode`). LOH fragmentation is a classic memory leak that isn't a leak.

**Do**
- Use **`Span<T>` / `ReadOnlySpan<T>`** for stack-only slicing of arrays, strings, native memory. Zero-alloc, bounds-checked, JIT loves them.
- Use **`Memory<T>` / `ReadOnlyMemory<T>`** when the buffer must escape a frame (async, fields).
- **`stackalloc Span<T>`** for small, bounded buffers (≤ ~1 KB rule of thumb; never inside a loop without bounds; never escape it). Gate larger sizes with `if (size <= threshold) stackalloc … else ArrayPool`.
- **`ArrayPool<T>.Shared.Rent` / `Return`** for transient buffers. **Always** `Return` in `finally`. Pass `clearArray: true` only if it held secrets. **Honor the returned length:** `Rent(n)` may return an array *larger* than `n` — slice via `buffer.AsSpan(0, n)` and never assume `buffer.Length == n`.
- **`MemoryPool<T>`** when the consumer wants `IMemoryOwner<T>`.
- **`ObjectPool<T>`** (`Microsoft.Extensions.ObjectPool`) for expensive-to-construct mutable objects (StringBuilders, parsers). Use `DefaultObjectPoolProvider` and write a `PooledObjectPolicy<T>` with a `Reset`.
- **`Microsoft.IO.RecyclableMemoryStream`** instead of `new MemoryStream()` for serialization/proxy/streaming. It pools blocks and avoids the LOH for large payloads. Create **one static `RecyclableMemoryStreamManager` per process** (the manager is thread-safe; individual streams are not) and obtain streams from it.
- Prefer **`struct`** for small (≤ 16 bytes), short-lived value types. Mark `readonly struct` and `readonly` members to let the JIT skip defensive copies.
- **Span/Memory + `Stream` interop** before reaching for `Pipelines`. Modern `Stream` exposes `Read(Span<byte>)`, `Write(ReadOnlySpan<byte>)`, `ReadAsync(Memory<byte>, …)`, `WriteAsync(ReadOnlyMemory<byte>, …)` — pair these with a rented `ArrayPool<byte>` buffer and you've covered most "stream a payload through" paths without changing the I/O model. Escalate to **`PipeReader`/`PipeWriter`** only when you need backpressure, multi-segment `ReadOnlySequence<byte>` parsing, or true zero-copy framing.

**Pooling correctness — the rule that makes the perf real**

Pooling that forgets to release is *retained* memory, not *recycled* memory. Every pool API has an explicit return contract; violate it and you've shipped a leak that profilers will blame on something else.

- **`ArrayPool<T>.Shared.Rent(n)`** → `try { … } finally { ArrayPool<T>.Shared.Return(buffer, clearArray: containsSecrets); }`. Never `await` between rent and return without a `try/finally` — an exception silently leaks the buffer.
- **`MemoryPool<T>.Shared.Rent(n)`** returns **`IMemoryOwner<T>`** — wrap in `using` so `Dispose()` returns the buffer to the pool. Storing an `IMemoryOwner<T>` in a field means *you* own the disposal lifecycle; document it.
- **`ObjectPool<T>`** → `var obj = pool.Get(); try { … } finally { pool.Return(obj); }`. The `PooledObjectPolicy<T>.Return` override must reset the object (clear `StringBuilder`, drain collections); skipping the reset poisons the pool with stale state.
- **`RecyclableMemoryStream`** is `IDisposable`; `using` it. Without `Dispose()` the underlying blocks stay rented forever — a textbook "we pooled it and our RSS doubled" outage.
- Verify with `[MemoryDiagnoser]` plus a long-running soak: rent rate ≈ return rate, RSS flat after warmup. If they diverge, you have a missing return path.

**Don't**

**Don't**
- Don't `ToArray()` / `ToList()` on hot paths just to "materialize." Iterate the source.
- Don't capture closures in hot lambdas — each capture allocates a display class.
- Don't pass `params object[]` (boxes value types). Use generic overloads or interpolated handlers (§3).
- Don't `stackalloc` unbounded sizes (StackOverflowException = process death; no catch).
- Don't pool things that aren't expensive. Pooling a 24-byte POCO loses to the GC every time.
- Don't rent without a `try/finally` (or `using` for `IMemoryOwner<T>` / `RecyclableMemoryStream`). A leaked rent is a leak that compounds under load.

**Sources:**
- Stephen Toub on pooling and `Span<T>`: https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/
- `ArrayPool<T>` / `MemoryPool<T>` / `IMemoryOwner<T>`: https://learn.microsoft.com/dotnet/api/system.buffers.arraypool-1, https://learn.microsoft.com/dotnet/api/system.buffers.memorypool-1
- `Microsoft.IO.RecyclableMemoryStream`: https://github.com/microsoft/Microsoft.IO.RecyclableMemoryStream
- `System.IO.Pipelines`: https://learn.microsoft.com/dotnet/standard/io/pipelines
- Konrad Kokosa, *Pro .NET Memory Management* — LOH, pooling, retention semantics.

---

## 3. Strings — the most allocated type

**Do**
- **Interpolated strings** (`$"..."`) are usually fine in C# 10+: the compiler lowers them through **`DefaultInterpolatedStringHandler`** (a `ref struct` using pooled buffers). On hot paths, write **custom `[InterpolatedStringHandler]`** types (the pattern `Debug.Assert`, `ILogger`, `StringBuilder.Append` use) so arguments are formatted only when needed.
- **`StringBuilder`** for unbounded concatenation; pool it (`ObjectPool<StringBuilder>` via `StringBuilderPooledObjectPolicy`).
- **`string.Create(length, state, span => …)`** when you know the final length and can write in place. Zero intermediate allocations.
- **`ReadOnlySpan<char>`** for parsing — `int.Parse(span)`, `span.Split(…)` (.NET 8+), `MemoryExtensions.*`. Avoid `Substring`.
- **UTF-8 literals**: `"text"u8` returns `ReadOnlySpan<byte>` baked into the assembly. Use everywhere you write to a UTF-8 sink (`Utf8JsonWriter`, `PipeWriter`, sockets, HTTP/2 frames). No transcoding, no allocation.
- **`Utf8.TryWrite` / `Utf8Formatter`** for direct UTF-8 formatting.
- Use `string.Equals(a, b, StringComparison.Ordinal)` on hot paths; `Ordinal` is the fastest comparer. `OrdinalIgnoreCase` is next. Culture-aware comparisons are 10–100× slower.

**Don't**
- Don't `+` in a loop. Don't `string.Format` on hot paths.
- Don't `.ToLower()` to compare — use `StringComparison.OrdinalIgnoreCase`.
- Don't transcode UTF-16 ⇄ UTF-8 repeatedly. Pick one for the boundary and stay there.

**Sources:**
- Stephen Toub, "Performance Improvements in .NET 10" (string / interpolation / UTF-8 sections) — https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/
- Interpolated string handlers: https://learn.microsoft.com/dotnet/csharp/whats-new/tutorials/interpolated-string-handler
- UTF-8 string literals: https://learn.microsoft.com/dotnet/csharp/language-reference/proposals/csharp-11.0/utf8-string-literals

---

## 4. Collections — pick the right one, size it

**Do**
- **Pre-size** every `List<T>`, `Dictionary<,>`, `HashSet<T>`, `StringBuilder` you can. Resizing copies and re-hashes.
- For tight numeric loops over a `List<T>`, take a span: `CollectionsMarshal.AsSpan(list)`. Drops bounds-check overhead, removes the indexer call.
- **`Dictionary<TKey,TValue>`**: use `TryGetValue`, not `ContainsKey` + indexer (double lookup). For mutate-or-add use **`CollectionsMarshal.GetValueRefOrAddDefault`** — single lookup, by-ref slot. For struct values, this avoids a copy.
- **Read-mostly, build-once** lookups: **`FrozenDictionary<TKey,TValue>` / `FrozenSet<T>`** (`System.Collections.Frozen`). Slower to build, faster to read than `Dictionary`/`HashSet`. Use for routing tables, enum maps, config caches.
- **`ImmutableArray<T>`** (a wrapper around `T[]`) for small read-only fields — zero indirection. **`ImmutableList<T>`** is a tree; only use when you need cheap copy-on-write.
- **`ArrayPool<T>` + `Span<T>`** beats `List<T>` when size is bounded and known.

**Don't**
- Don't use `LinkedList<T>`. Almost never the right answer (cache-hostile, allocates per node).
- Don't use `ConcurrentBag<T>` unless your access pattern is thread-local-then-steal. `ConcurrentQueue<T>` is usually what you want.
- Don't use `ImmutableDictionary` as a hot read path. `FrozenDictionary` exists.

**Sources:**
- `System.Collections.Frozen`: https://learn.microsoft.com/dotnet/api/system.collections.frozen
- `CollectionsMarshal`: https://learn.microsoft.com/dotnet/api/system.runtime.interopservices.collectionsmarshal
- Stephen Toub on collections improvements (annual perf posts): https://devblogs.microsoft.com/dotnet/author/toub/

---

## 5. JSON — `System.Text.Json`, source-generated

**Do**
- **Source-generation** (`JsonSerializerContext`) for everything. Eliminates startup reflection, enables trimming and NativeAOT, faster steady-state.
  ```csharp
  [JsonSerializable(typeof(MyDto))]
  internal partial class AppJsonContext : JsonSerializerContext;
  // JsonSerializer.Serialize(value, AppJsonContext.Default.MyDto);
  ```
- Configure `JsonSerializerOptions` **once** as a static singleton. Re-creating per call destroys perf (each instance JIT-compiles converters).
- For polymorphism use `[JsonDerivedType(typeof(Sub), "sub")]` on the base — works under source-gen and AOT.
- Hot paths / streaming: drop down to **`Utf8JsonReader`** and **`Utf8JsonWriter`** with a pooled buffer or `IBufferWriter<byte>` (e.g. `PipeWriter`). Skip the model entirely.
- Prefer `SerializeAsync(Stream, …)` / `DeserializeAsync` with `PipeReader`/`PipeWriter` for HTTP. ASP.NET Core's input/output formatters already do this.

**Don't**
- Don't use `Newtonsoft.Json` on new hot paths in 2025. (Fine for compatibility, slower and allocates more.)
- Don't deserialize to `JsonDocument`/`JsonNode` if you know the shape — full POCO + source-gen is faster and lower-alloc.
- Don't enable `WriteIndented` in production.

**Sources:**
- `System.Text.Json` source generation: https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation
- `Utf8JsonReader` / `Utf8JsonWriter`: https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/use-utf8jsonwriter
- Stephen Toub on STJ improvements (annual posts): https://devblogs.microsoft.com/dotnet/author/toub/

---

## 6. Reflection alternatives — source generators win

Reflection is allocation, slow startup, and AOT-hostile. Most reasons to use it have a generator now.

**Do**
- **`LoggerMessage` source generator** (§12).
- **Options validation source generator** (`[OptionsValidator]`).
- **Regex source generator** (`[GeneratedRegex("…")]`) — compiles to IL at build time, zero startup cost, AOT-safe.
- **JSON source generator** (§5).
- **ASP.NET Core**: `RequestDelegateGenerator` (Minimal APIs) and the **MVC source generator** remove per-request reflection and enable AOT.
- **`ConfigurationBinder` source generator** (`[ConfigurationKeyName]` etc.) for AOT-friendly options binding.
- **gRPC** and **SignalR** AOT generators where applicable.

**When you must do dynamic dispatch:**
- Cache `MethodInfo`/`PropertyInfo` lookups; never do them per call.
- Compile to a delegate via **`Expression<T>.Compile()`** or **`DynamicMethod`/IL emit** once, invoke many. ~100× faster than `MethodInfo.Invoke`.
- For property get/set on POCOs, prefer source-gen (`Microsoft.Extensions.Configuration.Binder` SG, JSON SG) over hand-rolled emit.

**Don't**
- Don't `Activator.CreateInstance` on hot paths.
- Don't ship reflection-based DI containers and then ask why startup is 4 s. (`Microsoft.Extensions.DependencyInjection` is fine; it caches.)

**Sources:**
- `LoggerMessage` source generator: https://learn.microsoft.com/dotnet/core/extensions/logger-message-generator
- Regex source generator: https://learn.microsoft.com/dotnet/standard/base-types/regular-expression-source-generators
- Configuration binder source generator: https://learn.microsoft.com/dotnet/core/extensions/configuration-generator
- Trimming-warning fixes & source generators overview: https://learn.microsoft.com/dotnet/core/deploying/trimming/

---

## 7. Async perf

**Do**
- Default to `Task` / `Task<T>`. Switch to **`ValueTask` / `ValueTask<T>` only when measured**: methods that frequently complete synchronously and are called millions of times/sec (e.g. cache hits, pipe reads). **A `ValueTask` may only be awaited once** — that constraint is real and easy to violate.
- **`IAsyncEnumerable<T>`** for streaming results; supports `await foreach`, `WithCancellation`, `ConfigureAwait`. The C# compiler emits a state machine that can be `[AsyncMethodBuilder]`-customized — `System.Threading.Channels` and `System.Linq.Async` make this ergonomic. **Hot-path caveat:** every `MoveNextAsync` is an async state-machine step plus a cancellation/disposal hook — fine for latency-sensitive streaming (SSE, gRPC server streams, paged enumeration), often beaten on raw throughput by **`Channel<T>`** or chunked `ReadOnlyMemory<T>` batches that amortize one async hop over many elements. Watch for accidental multi-enumeration (`IAsyncEnumerable` is single-shot for most producers) and remember the implicit `await DisposeAsync()` that `await foreach` injects. Benchmark per-element vs per-batch before defaulting either way.
- If the method body is fully synchronous, **don't mark it `async`** — return `Task.FromResult` / `ValueTask.FromResult` (or expose a sync API). The async state machine costs allocation and overhead.
- In **library code**, write `.ConfigureAwait(false)` everywhere. In ASP.NET Core app code it doesn't matter (no sync context), but library code may run under WinForms/WPF/Xamarin where it does. ConfigureAwait analyzers (`Microsoft.VisualStudio.Threading.Analyzers`, `ConfigureAwaitChecker`) help.
- Cancel via `CancellationToken` — pass it through, register lightly, and for very hot paths consider `CancellationToken.UnsafeRegister` to skip ExecutionContext capture.
- Use **`Task.WhenAll` / `Task.WhenAny`** for fan-out; pre-size the input list.
- For producer/consumer, use **`System.Threading.Channels`** (zero-alloc steady state, much faster than `BlockingCollection`).
- **Pooled async state machines**: enable with `<TieredCompilation>true</TieredCompilation>` (default) plus `[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder))]` for `ValueTask`-returning methods. Or set `DOTNET_SYSTEM_THREADING_POOLASYNCVALUETASKS=1` process-wide. Measure first.

**Don't**
- Don't `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` — sync-over-async deadlocks and starves the threadpool.
- Don't `async void` except for top-level event handlers. Exceptions crash the process.
- Don't allocate a `CancellationTokenSource` per request when one is already supplied.
- Don't `Task.Run(() => syncMethod())` to "make it async" in ASP.NET Core. It steals threadpool threads from real work.

**Sources:**
- Stephen Toub, "ValueTask restrictions": https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/
- `System.Threading.Channels` overview: https://learn.microsoft.com/dotnet/core/extensions/channels
- `IAsyncEnumerable<T>` patterns: https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/generate-consume-asynchronous-stream
- Stephen Cleary on async semantics: https://blog.stephencleary.com/

---

## 8. JIT & AOT

**Tiered compilation** (default-on since .NET Core 3.0; `TieredCompilation`, `TieredCompilation.QuickJit`): tier-0 compiles fast, runs unoptimized; hot methods promote to tier-1 (optimized) using **dynamic PGO** instrumentation. **.NET 8 made dynamic PGO default** (`TieredPGO=1` is now redundant) and shipped **AVX-512 / `Vector512<T>`** plus physical struct promotion. **.NET 9** sharpened DPGO further (better devirtualization, Arm64 vectorization, loop opts) and made **DATAS** the default GC mode (see §9). **.NET 10** added **physical promotion of struct args into shared registers**, **improved loop inversion**, more DPGO/stack-allocation work, and AVX10/512 codegen. You generally do nothing — but understand it when reading benchmarks.

**ReadyToRun (R2R / crossgen2)**: AOT-compiles framework + your assemblies to native stubs that the JIT can replace. Use for **startup-sensitive** services. `dotnet publish -p:PublishReadyToRun=true`. Pair with `PublishReadyToRunComposite=true` for one-big-image, smaller disk, faster cold start; but updates require re-publish.

**NativeAOT** (`<PublishAot>true</PublishAot>`): full ahead-of-time compilation, no JIT, no tiered comp, no runtime codegen. **ASP.NET Core support shipped in .NET 8**; .NET 9/10 expanded it (smaller images, better library compatibility, EF/reflection improvements) so it's production-ready for many workloads, including ASP.NET Core Minimal APIs, gRPC, and console tools.
- **Wins**: 60–90% faster cold start, ~50–70% smaller images, lower steady-state working set, no JIT memory cost, fewer attack surfaces.
- **Costs (give-ups)**: no `Reflection.Emit`, no `Assembly.LoadFrom` of arbitrary IL, restricted reflection (must be analyzable / `[DynamicallyAccessedMembers]`), no plug-in / dynamic loading of new managed code, some libraries unsupported. Trimming is mandatory and aggressive.
- **Targets**: cloud-native APIs, CLI tools, Lambda/Functions, serverless/cold-start-critical, sidecars.

**Do NOT choose NativeAOT when any of the following is true:**
- You use **Entity Framework Core** with lazy-loading proxies, runtime model discovery, dynamic LINQ providers, or providers that haven't published an AOT-compatibility statement. EF Core's AOT story has improved but still relies on expression compilation paths that trim/AOT will silently mis-handle.
- You **load assemblies at runtime** (`Assembly.LoadFrom`, `AssemblyLoadContext`, MEF, plug-in hosts, scripting via Roslyn/CSharpScript).
- You depend on **runtime codegen** — `Reflection.Emit`, `DynamicMethod`, `Expression<T>.Compile()` on shapes not known at publish time, dynamic proxy frameworks (Castle DynamicProxy, Moq for production code).
- Your stack is **reflection-heavy** without source-generator equivalents — older serializers (Newtonsoft.Json with `[JsonExtensionData]` on unknown types), AutoMapper without source-gen profiles, hand-rolled DI scanners.
- You consume **NuGet packages without verified AOT compatibility**. Audit each dependency against its `IsAotCompatible`/trim warnings; one library that emits IL takes the whole process down.
- Validate by publishing with `<IsAotCompatible>true</IsAotCompatible>` and `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` in CI; trim/AOT warnings *are* the bug list.

**Trimming** (`<PublishTrimmed>true</PublishTrimmed>`, modes `partial` / `full`): removes unreferenced IL. Required for AOT. Annotate with `[DynamicallyAccessedMembers]` / `[RequiresUnreferencedCode]`; treat trim warnings as errors (`<TrimmerSingleWarn>false</TrimmerSingleWarn>`, `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`).

**Sources:**
- ReadyToRun & crossgen2: https://learn.microsoft.com/dotnet/core/deploying/ready-to-run
- NativeAOT deployment & limitations: https://learn.microsoft.com/dotnet/core/deploying/native-aot/
- Trimming overview: https://learn.microsoft.com/dotnet/core/deploying/trimming/
- Tiered compilation / Quick JIT / DPGO config: https://learn.microsoft.com/dotnet/core/runtime-config/compilation
- What's new in the .NET 8 runtime (DPGO default-on, AVX-512, physical promotion): https://learn.microsoft.com/dotnet/core/whats-new/dotnet-8/runtime
- What's new in the .NET 10 runtime (struct-arg promotion, loop inversion, DPGO): https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/runtime
- Stephen Toub, "Performance Improvements in .NET 10" — https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/
- Stephen Toub, "Performance Improvements in .NET 9" — https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-9/
- Stephen Toub, "Performance Improvements in .NET 8" — https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/

---

## 9. GC

**Modes**:
- **Server GC** (`<ServerGarbageCollection>true</ServerGarbageCollection>` / `System.GC.Server`): one heap and one GC thread per logical CPU, parallel collection, larger segments. Higher throughput, higher memory floor. **The runtime default is Workstation GC**; ASP.NET Core's host opts the process into Server GC. **On a single-logical-CPU machine (or a container with cgroup CPU < 1) the runtime forces Workstation GC regardless of config** — relevant when you tightly CPU-limit a sidecar.
- **Workstation GC**: one heap, one GC thread, smaller footprint. Default for desktop / single-CPU. Lower throughput.
- **Concurrent / background GC** (`<ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>`, default): Gen2 collection runs concurrently with mutator, dramatically reducing pause times.

**Generations**: Gen0 (nursery, ~cheap), Gen1 (survivors), Gen2 (long-lived, blocking on Workstation, background on Server), **LOH** (≥ 85 KB, Gen2). .NET 10 includes write-barrier improvements that further reduce GC pause time on Arm64 (8–20% in Microsoft's numbers).

**Containers** (heuristics, not commandments):
- **Set memory limits at the container level** so the runtime knows the budget. With Server GC + DATAS, .NET reads the cgroup limit and sizes heap heuristically (commonly around ~75% of the limit, but the exact target is implementation-defined and version-dependent — don't memorize the number).
- **`DOTNET_GCHeapHardLimit`** / **`DOTNET_GCHeapHardLimitPercent`** are *override* knobs, not defaults. Reach for them in **multi-tenant pods**, **co-located sidecars**, or **tightly budgeted memory envs** where you'd rather GC more aggressively than risk an OOMKill. Trade-off: a hard cap forces more frequent / longer collections — measure pause time and throughput before pinning a value.
- **`DOTNET_gcServer=1`**, **`DOTNET_gcConcurrent=1`** are typically already on for ASP.NET Core; verify in your image rather than re-asserting them everywhere.
- **DATAS** (Dynamically Adapting To Application Sizes) shrinks heap when idle. **Introduced as opt-in in .NET 8 (`System.GC.DynamicAdaptationMode=1` / `DOTNET_GCDynamicAdaptationMode=1`); made the default Server-GC mode in .NET 9.** Disable only with reason and a benchmark.

**Latency knobs (situational, not defaults)**
- **`GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency`** — suppresses Gen2 (full blocking) collections for the scope, allowing only Gen0/Gen1/background work. Useful for **bounded** critical sections (trading windows, frame loops, request bursts with known duration). Costs: heap grows, fragmentation rises, and you *must* return to `Interactive` afterwards. Not a process-wide default.
- **`GCSettings.LatencyMode = GCLatencyMode.LowLatency`** — weaker, Workstation-only, similar caveats.
- **`GC.TryStartNoGCRegion(size)`** — blocks GC entirely until you exit the region or exceed `size`. Almost always wrong outside trading systems / game frames; if you need it, you have an allocation problem to fix instead.
- **`DOTNET_GCRetainVM=1`** — tells the GC to retain (rather than release back to the OS) virtual memory segments freed during collection, so subsequent allocations skip the syscall round-trip. Helps **steady high-throughput** services that allocate in waves; hurts **memory-budgeted multi-tenant** hosts because RSS won't shrink between bursts. Decide by workload, not by default.

**Diagnose**: `dotnet-counters monitor System.Runtime` (gen sizes, % time in GC, alloc rate), `dotnet-gcdump`, PerfView GCStats, the GC ETW/EventPipe events.

**Sources:**
- GC fundamentals: https://learn.microsoft.com/dotnet/standard/garbage-collection/fundamentals
- Workstation vs Server GC (defaults, single-CPU rule): https://learn.microsoft.com/dotnet/standard/garbage-collection/workstation-server-gc
- Latency modes: https://learn.microsoft.com/dotnet/standard/garbage-collection/latency
- What's new in the .NET 9 runtime (DATAS as default): https://learn.microsoft.com/dotnet/core/whats-new/dotnet-9/runtime
- GC configuration knobs (`Server`, `DynamicAdaptationMode`, `HeapHardLimit*`, `RetainVM`): https://learn.microsoft.com/dotnet/core/runtime-config/garbage-collector
- Stephen Toub, "Performance Improvements in .NET 9" (DATAS explainer) — https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-9/
- Konrad Kokosa, *Pro .NET Memory Management* (Apress, 2018) — https://prodotnetmemory.com/
- Maoni Stephens on GC internals: https://maoni0.medium.com/

---

## 10. HTTP & networking

**Do**
- **Reuse `HttpClient`**. Use **`IHttpClientFactory`** (`AddHttpClient`) — it manages handler rotation so DNS changes aren't ignored, and lets you compose with Polly / resilience handlers.
- Tune **`SocketsHttpHandler`** explicitly:
  - `PooledConnectionLifetime` — cap connection age (e.g. `TimeSpan.FromMinutes(2)`) so DNS/load-balancer changes propagate.
  - `PooledConnectionIdleTimeout`.
  - `MaxConnectionsPerServer` — default is `int.MaxValue`; set this for upstreams you don't want to hammer.
  - `EnableMultipleHttp2Connections = true` for high-fan-out HTTP/2.
  - `AutomaticDecompression = DecompressionMethods.All`.
- Prefer **HTTP/2** for multiplexed loads; **HTTP/3 (QUIC)** in .NET 8+ for high-latency / lossy networks (`HttpVersion = HttpVersion.Version30`, `HttpVersionPolicy.RequestVersionOrLower`). HTTP/3 is GA in Kestrel.
- **gRPC** over HTTP/2 with `protobuf-net` or built-in code-gen. Use `grpc-dotnet` `MaxReceiveMessageSize` and `ResponseCompressionLevel`. Streaming over unary when bidirectional.
- For hot in-memory parsing of network bytes, use **`System.IO.Pipelines`** (`PipeReader`, `PipeWriter`) — backpressure, zero-copy, `ReadOnlySequence<byte>`.

**Don't**
- Don't `new HttpClient()` per request (socket exhaustion + DNS pinning).
- Don't disable TLS resumption / set silly timeouts because someone copy-pasted from Stack Overflow.
- Don't ignore `MaxConnectionsPerServer` and then wonder why your downstream falls over.

**Sources:**
- `IHttpClientFactory`: https://learn.microsoft.com/aspnet/core/fundamentals/http-requests
- `SocketsHttpHandler`: https://learn.microsoft.com/dotnet/api/system.net.http.socketshttphandler
- `System.IO.Pipelines`: https://learn.microsoft.com/dotnet/standard/io/pipelines
- HTTP/3 in Kestrel: https://learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel/http3

---

## 11. Containers — what to set

There is **no universal "perf-tuned" env-var set**. The runtime defaults are good. Each tunable below moves a specific axis (startup vs steady-state vs memory) and can regress the others. Decide per service, then bake into the image. (**Env-var prefix:** `DOTNET_` is the standardized prefix since .NET 6; the old `COMPlus_` names still work but new code should use `DOTNET_`.)

**Minimal Dockerfile (no tuning, just sane base):**

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled AS runtime
ENV ASPNETCORE_URLS=http://+:8080
# Server GC + concurrent GC are already on for ASP.NET Core images; verify, don't re-assert.
```

**Startup-sensitive profile** — short-lived processes, serverless, scale-to-zero, frequent rollouts:

```dockerfile
ENV DOTNET_TieredPGO=1 \
    DOTNET_ReadyToRun=1 \
    DOTNET_TC_QuickJitForLoops=1
# Pair with: dotnet publish -p:PublishReadyToRun=true (or PublishAot=true).
```

- `DOTNET_TieredPGO=1` — dynamic PGO is already on by default in .NET 8+; explicit `=1` is a no-op-or-affirmation, harmless to set.
- `DOTNET_ReadyToRun=1` — only meaningful if the assemblies were actually published with R2R; otherwise inert.
- `DOTNET_TC_QuickJitForLoops=1` — lets methods with loops stay at tier-0 longer (faster startup, slower steady-state until promoted). **Can regress throughput** for long-running services with hot loops; benchmark before defaulting on.

**Throughput-sensitive profile** — long-lived APIs, workers, anything where steady-state RPS dominates startup:

```dockerfile
# Leave defaults. Don't set TC_QuickJitForLoops; let tier-1 kick in promptly.
# Don't set ReadyToRun unless you've measured a startup win that matters.
```

**Memory-budgeted profile** — multi-tenant pods, sidecars, tight cgroups:

```dockerfile
ENV DOTNET_GCHeapHardLimitPercent=75
# Or DOTNET_GCHeapHardLimit=<bytes> for an absolute cap.
# Trade-off: more frequent / longer GCs. Measure pause time.
```

- Don't set `GCHeapHardLimit*` "just in case" — it forces tighter GC behavior that costs throughput. The default container heuristic (DATAS + cgroup-aware sizing) is fine for the common case.

**General hygiene (apply broadly):**

- **Base images**: prefer **chiseled Ubuntu** (`*-noble-chiseled`) — distroless, ~100 MB, no shell, smaller CVE surface. **Azure Linux** (formerly Mariner) for Microsoft-supported distro. **Alpine** (musl) for size, but watch for native interop quirks.
- **Build images on .NET SDK 10**, run on `aspnet:10.0-*` (or `runtime:10.0-*` for non-web).
- **Container publish**: `dotnet publish -t:PublishContainer -p:ContainerBaseImage=mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled -p:ContainerRepository=myapp` — no Dockerfile needed; uses the SDK's container support. Pair with `-p:PublishReadyToRun=true` *only if you've decided startup matters*.
- For NativeAOT: `dotnet publish -p:PublishAot=true -r linux-x64 -c Release` then COPY into a `runtime-deps:10.0-*-chiseled-extra` (no managed runtime) base.
- Set `ASPNETCORE_HTTP_PORTS=8080` (8080 since .NET 8) and run as non-root (chiseled images do this by default, UID 1654).

**Sources:**
- .NET runtime configuration knobs: https://learn.microsoft.com/dotnet/core/runtime-config/
- Tiered compilation & PGO: https://learn.microsoft.com/dotnet/core/runtime-config/compilation
- Chiseled containers: https://devblogs.microsoft.com/dotnet/dotnet-6-is-now-in-ubuntu-2204/ (and follow-ups), https://learn.microsoft.com/dotnet/core/docker/container-images
- SDK container publish: https://learn.microsoft.com/dotnet/core/docker/publish-as-container

---

## 12. Logging perf

`ILogger.LogInformation("user {UserId} did {Action}", id, action)` already avoids string formatting if the level is disabled — but only if the args don't allocate (boxing, `ToString()` calls, captured closures).

**Do**
- Use **`LoggerMessage` source generator**:
  ```csharp
  internal static partial class Log
  {
      [LoggerMessage(EventId = 1001, Level = LogLevel.Information,
                     Message = "User {UserId} did {Action}")]
      public static partial void UserDidAction(ILogger logger, long userId, string action);
  }
  ```
  Zero boxing for value types, pre-computed `EventId`, level gating compiled in, AOT-safe. ~5–10× cheaper than the extension methods on hot paths.
- For very hot paths gate with **`if (logger.IsEnabled(LogLevel.Debug))`** before constructing arguments.
- Pass **structured arguments** (`{UserId}`), never interpolate (`$"user {id}"`) — interpolation defeats the structured pipeline and forces formatting even if disabled.
- Configure sinks to **batch** and write off the request thread.

**Don't**
- Don't `LogInformation($"…")` (interpolation).
- Don't `.ToString()` complex objects in log args; let the sink do it lazily, or pre-extract fields.
- Don't log inside tight loops at Information.

**Sources:**
- `LoggerMessage` source generator: https://learn.microsoft.com/dotnet/core/extensions/logger-message-generator
- High-performance logging: https://learn.microsoft.com/dotnet/core/extensions/high-performance-logging

---

## 13. Common micro-optimizations (use surgically)

- **`CollectionsMarshal.AsSpan(list)`** and **`CollectionsMarshal.GetValueRefOrAddDefault(dict, key, out exists)`** — eliminate copies and double lookups.
- **`Span<T>.Sort`** — in-place, generic, zero-alloc.
- **`MemoryMarshal.Cast<TFrom,TTo>(span)`** — reinterpret without copy.
- **`Unsafe.As<TFrom,TTo>(ref x)`** — last-resort reinterpret. You are now responsible for correctness.
- **`ImmutableArray<T>`** in fields where you want value-semantics with zero indirection (it's a `struct` wrapper around `T[]`).
- **Pattern matching** (`is`, `switch` expressions) is generally as cheap as hand-written `if` chains; the compiler emits balanced dispatch. Prefer it for clarity.
- **`SearchValues<T>`** (.NET 8+) for "does this byte/char appear in this set" — replaces `IndexOfAny(charArray)` with a vectorized lookup.
- **SIMD**: `Vector<T>`, `Vector128/256/512<T>`, hardware-intrinsic namespaces (`System.Runtime.Intrinsics.X86/Arm`). **`Vector512<T>` and the `Avx512*` intrinsic classes shipped in .NET 8.** .NET 10 added AVX10.2 and Arm64 SVE intrinsics. For numeric kernels only; the LINQ/Span methods already use SIMD internally.
  - **AVX-512 / `Vector512<T>` portability:** wider isn't automatically faster. Probe **`Vector512.IsHardwareAccelerated`** (runtime auto-detection of AVX-512F support) and the specific `Avx512F.IsSupported` (etc.) at runtime; without hardware support `Vector512<T>` operations fall back to software emulation. Cloud VMs vary by SKU and frequency-throttle under sustained AVX-512 load. Benchmark on the **production hardware family**, not your dev laptop — a Vector512 win on Sapphire Rapids can be a Vector256 loss on an older Skylake worker. Keep a `Vector256` fallback path.
- **`[SkipLocalsInit]`** on methods that `stackalloc` large buffers and immediately overwrite — skips `init`-zero of locals. Measure; sometimes negligible.

**Sources:**
- `System.Runtime.Intrinsics`: https://learn.microsoft.com/dotnet/api/system.runtime.intrinsics
- `Vector512<T>` (introduced .NET 8, runtime auto-detection): https://learn.microsoft.com/dotnet/api/system.runtime.intrinsics.vector512
- `SearchValues<T>`: https://learn.microsoft.com/dotnet/api/system.buffers.searchvalues
- Stephen Toub on SIMD/intrinsics improvements (annual posts): https://devblogs.microsoft.com/dotnet/author/toub/

---

## 14. What NOT to optimize

- **Cold paths**. Startup code that runs once. Admin endpoints. Configuration loaders. Don't `Span`-ify them; clarity wins.
- **Things the framework already does**. `string.Concat(a, b, c)` is optimized. `Enumerable.ToArray` on an `ICollection<T>` does a single `CopyTo`. `Dictionary<,>` indexing is already O(1) with low constant factors. Read the source before "optimizing" BCL calls.
- **Micro wins that don't move the macro needle**. A 10 ns savings × 1M calls/sec is 1% CPU. A 10 ns savings × 100 calls/sec is noise. Prove it on a flame graph.
- **DI resolution time** in normal apps. It's not your bottleneck; database round-trips and JSON are.
- **LINQ** — until profiling shows it. Modern LINQ is heavily optimized (.NET 9 rewrote a lot, .NET 10 more). The `foreach`+`List` rewrite of every LINQ chain is bikeshedding. **Hot-path rule, non-negotiable:** if a LINQ chain stays on a measured hot path, prove its allocation profile with `[MemoryDiagnoser]` (and ideally `[NativeMemoryProfiler]`) before signoff. Watch specifically for: closure captures (display-class allocs per call), iterator allocations (`Where`/`Select` chains), boxing of value-type predicates, interface enumeration (`IEnumerable<T>` instead of the concrete type). If the diagnoser shows non-zero `Allocated` per op on a thousands-of-calls-per-second path, rewrite.
- **`Task` → `ValueTask`** without measurement. Often a wash or a regression.

If you can't draw the flame graph that justifies the change, don't ship the change.

**Sources:**
- BenchmarkDotNet `MemoryDiagnoser`: https://benchmarkdotnet.org/articles/configs/diagnosers.html
- Stephen Toub on LINQ improvements (annual posts): https://devblogs.microsoft.com/dotnet/author/toub/

---

## 15. Benchmark methodology — BenchmarkDotNet recipe

Minimum viable benchmark:

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Diagnosers;
using BenchmarkDotNet.Jobs;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]                 // alloc/op, Gen0/1/2
[ThreadingDiagnoser]              // lock contention, completed work items
[DisassemblyDiagnoser(maxDepth: 2)] // emitted asm — for JIT-curious work
[SimpleJob(RuntimeMoniker.Net100, baseline: true)]
[SimpleJob(RuntimeMoniker.Net90)]
public class JsonBench
{
    private MyDto _dto = new(...);

    [Benchmark(Baseline = true)]
    public byte[] Reflection() => JsonSerializer.SerializeToUtf8Bytes(_dto);

    [Benchmark]
    public byte[] SourceGen()  => JsonSerializer.SerializeToUtf8Bytes(_dto, AppJsonContext.Default.MyDto);
}

public class Program { public static void Main(string[] a) => BenchmarkRunner.Run<JsonBench>(); }
```

**Diagnoser cheat sheet**

| Diagnoser | What it adds | Cost |
| --- | --- | --- |
| `[MemoryDiagnoser]` | Allocated, Gen0/1/2 | ~free |
| `[ThreadingDiagnoser]` | Lock contention, work items | low |
| `[DisassemblyDiagnoser]` | Emitted assembly | high |
| `[EventPipeProfiler]` (Linux/macOS/Win) | Sampled CPU, GC, exceptions | medium |
| `[NativeMemoryProfiler]` | Native allocations | medium |
| `[HardwareCounters(...)]` | CPU perf counters (cache misses, branch mispredicts) | medium, Win/Linux |

**Rules**
- **Always include a `[Benchmark(Baseline = true)]`** so BDN emits the **Ratio** column. Absolute ns are noise; ratios are signal.
- Use `[Params]` to sweep input sizes — perf is a function of N, never a single number.
- Use `[GlobalSetup]` for non-benchmarked initialization.
- On Linux, BDN uses **EventPipe / LTTng** for tracing; on Windows, **ETW**. Both feed `dotnet-trace` / PerfView.
- Compare across runtimes with multiple `[SimpleJob(RuntimeMoniker.NetXX)]` — that's how Microsoft writes the annual Stephen Toub posts.
- Publish your benchmark project as `Release`, run from the command line (`dotnet run -c Release --project Bench -- --filter '*'`).

**Cold-start vs steady-state — separate harness, separate report**

BDN measures *steady-state*. Startup wins from R2R, NativeAOT, and source generators do not show up — and can be hidden — in BDN numbers. Run a second harness:

- Publish the binary as you'd ship it (R2R, AOT, trimmed — whatever the deployment is).
- Launch the **process** N times (≥30) from a clean state, no warmup, one request per launch.
- Measure: process exit time for a CLI; first-request latency / "Application started" log for a service; `GC.GetTotalAllocatedBytes()` and working set at first-request for memory.
- Aggregate as a histogram (median + p95), not a mean. Cold-start distributions are long-tailed.
- Report **alongside** the BDN steady-state table. A change that wins one and loses the other is a trade-off, not a win.

**Sources:**
- BenchmarkDotNet docs: https://benchmarkdotnet.org/articles/overview.html
- BDN how-it-works (warmup, pilot, statistics): https://benchmarkdotnet.org/articles/guides/how-it-works.html
- BenchmarkDotNet v0.14.0 release notes (dotMemory diagnoser, ScottPlot exporter, .NET 8 `UseArtifactsOutput` fix): https://github.com/dotnet/BenchmarkDotNet/releases/tag/v0.14.0
- Akinshin, Andrey. *Pro .NET Benchmarking: The Art of Performance Measurement*. Apress, 2019. ISBN 978-1-4842-4940-6 — https://aakinshin.net/prodotnetbenchmarking/
- dotnet/performance benchmark suite: https://github.com/dotnet/performance

---

## Sources

### Authoritative
- **Stephen Toub — "Performance Improvements in .NET 10"** (the canonical annual post). https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/
- Prior years (still the best mental model for the runtime): "Performance Improvements in .NET 9", ".NET 8", ".NET 7" on https://devblogs.microsoft.com/dotnet/author/toub/
- **Announcing .NET 10**. https://devblogs.microsoft.com/dotnet/announcing-dotnet-10/
- **What's new in .NET 10** (learn.microsoft.com). https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10/overview
- **.NET performance docs**. https://learn.microsoft.com/dotnet/standard/performance/
- **GC fundamentals & tuning**. https://learn.microsoft.com/dotnet/standard/garbage-collection/
- **Native AOT deployment**. https://learn.microsoft.com/dotnet/core/deploying/native-aot/
- **Trimming**. https://learn.microsoft.com/dotnet/core/deploying/trimming/
- **`System.Text.Json` source generation**. https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation
- **`LoggerMessage` source generator**. https://learn.microsoft.com/dotnet/core/extensions/logger-message-generator
- **Regex source generator**. https://learn.microsoft.com/dotnet/standard/base-types/regular-expression-source-generators
- **`IHttpClientFactory` & `SocketsHttpHandler`**. https://learn.microsoft.com/aspnet/core/fundamentals/http-requests / https://learn.microsoft.com/dotnet/api/system.net.http.socketshttphandler
- **`System.IO.Pipelines`**. https://learn.microsoft.com/dotnet/standard/io/pipelines
- **ASP.NET Core performance best practices**. https://learn.microsoft.com/aspnet/core/performance/performance-best-practices
- **BenchmarkDotNet docs**. https://benchmarkdotnet.org/articles/overview.html
- **dotnet/runtime performance docs & wiki**. https://github.com/dotnet/runtime/tree/main/docs/performance
- **dotnet/performance benchmark suite**. https://github.com/dotnet/performance
- **Diagnostic CLI tools** (`dotnet-counters`, `dotnet-trace`, `dotnet-dump`, `dotnet-gcdump`). https://learn.microsoft.com/dotnet/core/diagnostics/
- **PerfView**. https://github.com/microsoft/perfview

### Community / people to follow
- **Stephen Toub** (.NET libraries lead) — devblogs posts above; the source of truth.
- **Adam Sitnik** (BenchmarkDotNet maintainer, .NET perf team) — https://adamsitnik.com/ — pooling, BDN, micro-opt patterns.
- **Andrey Akinshin** (BenchmarkDotNet lead maintainer) — https://aakinshin.net/ — author of *Pro .NET Benchmarking: The Art of Performance Measurement* (Apress, 2019, ISBN 978-1-4842-4940-6); the reference on BDN methodology, statistical analysis, and benchmark pitfalls.
- **Ben Adams** (Illyriad Games / .NET community) — https://github.com/benaadams — real-world high-throughput .NET service perf, Kestrel/ASP.NET Core internals, allocation hunting.
- **Maoni Stephens** (.NET GC architect) — https://maoni0.medium.com/ — anything about GC internals.
- **Stephen Cleary** — https://blog.stephencleary.com/ — async / `Task` semantics, `ValueTask`, threading. Author of *Concurrency in C# Cookbook*.
- **Andrew Lock** — https://andrewlock.net/ — ASP.NET Core internals, source generators, deployment.
- **David Fowler** (.NET architect) — Twitter/X and dotnet/aspnetcore — Pipelines, channels, ASP.NET Core perf patterns.
- **Nick Chapsas** — YouTube *KeepCoding* — accessible perf walkthroughs and BenchmarkDotNet demos.
- **Konrad Kokosa — *Pro .NET Memory Management*** (Apress, 2018, still the definitive book on the runtime memory model and GC). Companion site: https://prodotnetmemory.com/
- **Sasha Goldshtein**, **Ben Watson — *Writing High-Performance .NET Code*** (2nd ed.) — practical perf engineering reference.
- **Marc Gravell** — protobuf-net, Pipelines, Dapper perf.

> If a tip in this doc disagrees with the latest Stephen Toub post, **the post wins**. Re-read it every November.
