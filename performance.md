# .NET 10 Performance — Opinionated Best Practices

Dense, opinionated, sourced. Targets **.NET 10 (LTS, Nov 2025)**. Assumes ASP.NET Core / library / service workloads. Read Stephen Toub's "Performance Improvements in .NET 10" before arguing with anything here.

> **The single rule:** measure, don't guess. Every "do" below has a "don't" because someone shipped the don't and regretted it.

---

## 1. Methodology — measure first, always

**Do**
- Start from a **macro metric** that matters: p50/p95/p99 latency, RPS, allocations/sec, CPU%, GC pause time, working set. If you can't tie a micro-win to a macro metric, you're polishing brass.
- For micro-benchmarks: **BenchmarkDotNet (BDN)**. Always. It handles warmup, pilot, JIT tier promotion, statistical analysis, and process isolation. Anything you write with `Stopwatch` in a `for` loop is wrong by default (cold JIT, no warmup, dead-code elimination, tier-0 code).
- For live processes: **`dotnet-counters`** (cheap, EventCounters/Meter), **`dotnet-trace`** (sampled CPU/events, cross-platform), **`dotnet-gcdump`**, **`dotnet-dump`**, **PerfView** (Windows ETW, deepest), **`perf`/LTTng** on Linux, **EventPipe** as the cross-platform substrate. **Visual Studio Profiler** and **JetBrains dotTrace/dotMemory** for GUI-driven analysis.
- Isolate variables. One change per benchmark run. Pin CPU governors (`cpupower frequency-set -g performance`), disable turbo or accept the noise, run on quiet hardware, kill antivirus.
- Understand **tiered JIT**: tier-0 = fast compile / slow code; tier-1 = optimized after N calls (default 30). Without warmup you measure tier-0. **Dynamic PGO** (on-by-default since .NET 8, sharper in .NET 9/10) reshuffles tier-1 based on tier-0 instrumentation — so the *shape* of warmup matters.
- Reproduce production: same TFM, same RID, same GC mode (Server vs Workstation), same container limits.

**Don't**
- Don't benchmark in `Debug`. Don't benchmark with a debugger attached. Don't benchmark on a laptop on battery.
- Don't trust a single run. BDN's confidence intervals exist for a reason.
- Don't optimize before you've profiled. ~80% of "obvious" wins move nothing because they're off the hot path.

---

## 2. Allocations — the silent latency tax

GC time is allocation time deferred. Every `new` on the hot path is a future Gen0 (cheap), Gen1 (less cheap), or Gen2 (expensive, blocking on Workstation GC) collection. Objects ≥ **85,000 bytes** go on the **Large Object Heap (LOH)** — Gen2-collected, not compacted by default (opt-in via `GCSettings.LargeObjectHeapCompactionMode`). LOH fragmentation is a classic memory leak that isn't a leak.

**Do**
- Use **`Span<T>` / `ReadOnlySpan<T>`** for stack-only slicing of arrays, strings, native memory. Zero-alloc, bounds-checked, JIT loves them.
- Use **`Memory<T>` / `ReadOnlyMemory<T>`** when the buffer must escape a frame (async, fields).
- **`stackalloc Span<T>`** for small, bounded buffers (≤ ~1 KB rule of thumb; never inside a loop without bounds; never escape it). Gate larger sizes with `if (size <= threshold) stackalloc … else ArrayPool`.
- **`ArrayPool<T>.Shared.Rent` / `Return`** for transient buffers. **Always** `Return` in `finally`. Pass `clearArray: true` only if it held secrets.
- **`MemoryPool<T>`** when the consumer wants `IMemoryOwner<T>`.
- **`ObjectPool<T>`** (`Microsoft.Extensions.ObjectPool`) for expensive-to-construct mutable objects (StringBuilders, parsers). Use `DefaultObjectPoolProvider` and write a `PooledObjectPolicy<T>` with a `Reset`.
- **`Microsoft.IO.RecyclableMemoryStream`** instead of `new MemoryStream()` for serialization/proxy/streaming. It pools blocks and avoids the LOH for large payloads.
- Prefer **`struct`** for small (≤ 16 bytes), short-lived value types. Mark `readonly struct` and `readonly` members to let the JIT skip defensive copies.

**Don't**
- Don't `ToArray()` / `ToList()` on hot paths just to "materialize." Iterate the source.
- Don't capture closures in hot lambdas — each capture allocates a display class.
- Don't pass `params object[]` (boxes value types). Use generic overloads or interpolated handlers (§3).
- Don't `stackalloc` unbounded sizes (StackOverflowException = process death; no catch).
- Don't pool things that aren't expensive. Pooling a 24-byte POCO loses to the GC every time.

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

---

## 7. Async perf

**Do**
- Default to `Task` / `Task<T>`. Switch to **`ValueTask` / `ValueTask<T>` only when measured**: methods that frequently complete synchronously and are called millions of times/sec (e.g. cache hits, pipe reads). **A `ValueTask` may only be awaited once** — that constraint is real and easy to violate.
- **`IAsyncEnumerable<T>`** for streaming results; supports `await foreach`, `WithCancellation`, `ConfigureAwait`. The C# compiler emits a state machine that can be `[AsyncMethodBuilder]`-customized — `System.Threading.Channels` and `System.Linq.Async` make this ergonomic.
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

---

## 8. JIT & AOT

**Tiered compilation** (default on): tier-0 compiles fast, runs unoptimized; hot methods promote to tier-1 (optimized) using **dynamic PGO** instrumentation. .NET 8 made dynamic PGO default; .NET 9 and **.NET 10** sharpened it (better devirtualization, profiled GDV, loop inversion, stack allocation of small objects, struct-arg improvements). You generally do nothing — but understand it when reading benchmarks.

**ReadyToRun (R2R / crossgen2)**: AOT-compiles framework + your assemblies to native stubs that the JIT can replace. Use for **startup-sensitive** services. `dotnet publish -p:PublishReadyToRun=true`. Pair with `PublishReadyToRunComposite=true` for one-big-image, smaller disk, faster cold start; but updates require re-publish.

**NativeAOT** (`<PublishAot>true</PublishAot>`): full ahead-of-time compilation, no JIT, no tiered comp, no runtime codegen. .NET 10 makes it production-ready for many workloads, including ASP.NET Core Minimal APIs, gRPC, and console tools.
- **Wins**: 60–90% faster cold start, ~50–70% smaller images, lower steady-state working set, no JIT memory cost, fewer attack surfaces.
- **Costs (give-ups)**: no `Reflection.Emit`, no `Assembly.LoadFrom` of arbitrary IL, restricted reflection (must be analyzable / `[DynamicallyAccessedMembers]`), no plug-in / dynamic loading of new managed code, some libraries unsupported (legacy WCF server, EF Core has caveats around expression compilation). Trimming is mandatory and aggressive.
- **Targets**: cloud-native APIs, CLI tools, Lambda/Functions, serverless/cold-start-critical, sidecars.
- **Don't** AOT a plug-in host or anything that loads user assemblies.

**Trimming** (`<PublishTrimmed>true</PublishTrimmed>`, modes `partial` / `full`): removes unreferenced IL. Required for AOT. Annotate with `[DynamicallyAccessedMembers]` / `[RequiresUnreferencedCode]`; treat trim warnings as errors (`<TrimmerSingleWarn>false</TrimmerSingleWarn>`, `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`).

---

## 9. GC

**Modes**:
- **Server GC** (`<ServerGarbageCollection>true</ServerGarbageCollection>`, default in ASP.NET Core): one heap and one GC thread per logical CPU, parallel collection, larger segments. Higher throughput, higher memory floor.
- **Workstation GC**: one heap, one GC thread, smaller footprint. Default for desktop / single-CPU. Lower throughput.
- **Concurrent / background GC** (`<ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>`, default): Gen2 collection runs concurrently with mutator, dramatically reducing pause times.

**Generations**: Gen0 (nursery, ~cheap), Gen1 (survivors), Gen2 (long-lived, blocking on Workstation, background on Server), **LOH** (≥ 85 KB, Gen2). .NET 10 includes write-barrier improvements that further reduce GC pause time on Arm64 (8–20% in Microsoft's numbers).

**Containers**:
- **Always set memory limits** at the container level; the runtime auto-tunes heap to ~75% of the cgroup limit.
- For tight limits set **`DOTNET_GCHeapHardLimit`** (bytes) or **`DOTNET_GCHeapHardLimitPercent`** (0–100). Without these you OOM-kill instead of GC-ing harder.
- **`DOTNET_gcServer=1`**, **`DOTNET_gcConcurrent=1`** are typically already on; verify in your image.
- **DATAS** (Dynamically Adapting To Application Sizes, on by default in .NET 8+ Server GC for containers) shrinks heap when idle. Disable only with reason.

**`GC.TryStartNoGCRegion`**: blocks GC for a critical section. Almost always wrong outside of trading systems / game frames. If you reach for it, you have an allocation problem to fix instead.

**Diagnose**: `dotnet-counters monitor System.Runtime` (gen sizes, % time in GC, alloc rate), `dotnet-gcdump`, PerfView GCStats, the GC ETW/EventPipe events.

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

---

## 11. Containers — what to set

`Dockerfile` essentials for a perf-tuned ASP.NET Core image:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled AS runtime
# or *-azurelinux for the Microsoft Linux base, *-alpine for musl
ENV DOTNET_TieredPGO=1 \
    DOTNET_ReadyToRun=1 \
    DOTNET_TC_QuickJitForLoops=1 \
    DOTNET_GCHeapHardLimitPercent=75 \
    DOTNET_gcServer=1 \
    DOTNET_gcConcurrent=1 \
    ASPNETCORE_URLS=http://+:8080
```

- **Base images**: prefer **chiseled Ubuntu** (`*-noble-chiseled`) — distroless, ~100 MB, no shell, smaller CVE surface. **Azure Linux** (formerly Mariner) for Microsoft-supported distro. **Alpine** (musl) for size, but watch for native interop quirks.
- **Build images on .NET SDK 10**, run on `aspnet:10.0-*` (or `runtime:10.0-*` for non-web).
- **Container publish**: `dotnet publish -t:PublishContainer -p:ContainerBaseImage=mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled -p:ContainerRepository=myapp` — no Dockerfile needed; uses the SDK's container support. Pair with `-p:PublishReadyToRun=true`.
- For NativeAOT: `dotnet publish -p:PublishAot=true -r linux-x64 -c Release` then COPY into a `runtime-deps:10.0-*-chiseled-extra` (no managed runtime) base.
- Set `ASPNETCORE_HTTP_PORTS=8080` (8080 since .NET 8) and run as non-root (chiseled images do this by default, UID 1654).

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

---

## 13. Common micro-optimizations (use surgically)

- **`CollectionsMarshal.AsSpan(list)`** and **`CollectionsMarshal.GetValueRefOrAddDefault(dict, key, out exists)`** — eliminate copies and double lookups.
- **`Span<T>.Sort`** — in-place, generic, zero-alloc.
- **`MemoryMarshal.Cast<TFrom,TTo>(span)`** — reinterpret without copy.
- **`Unsafe.As<TFrom,TTo>(ref x)`** — last-resort reinterpret. You are now responsible for correctness.
- **`ImmutableArray<T>`** in fields where you want value-semantics with zero indirection (it's a `struct` wrapper around `T[]`).
- **Pattern matching** (`is`, `switch` expressions) is generally as cheap as hand-written `if` chains; the compiler emits balanced dispatch. Prefer it for clarity.
- **`SearchValues<T>`** (.NET 8+) for "does this byte/char appear in this set" — replaces `IndexOfAny(charArray)` with a vectorized lookup.
- **SIMD**: `Vector<T>`, `Vector128/256/512<T>`, hardware-intrinsic namespaces (`System.Runtime.Intrinsics.X86/Arm`). .NET 10 added AVX10.2 and Arm64 SVE intrinsics. For numeric kernels only; the LINQ/Span methods already use SIMD internally.
- **`[SkipLocalsInit]`** on methods that `stackalloc` large buffers and immediately overwrite — skips `init`-zero of locals. Measure; sometimes negligible.

---

## 14. What NOT to optimize

- **Cold paths**. Startup code that runs once. Admin endpoints. Configuration loaders. Don't `Span`-ify them; clarity wins.
- **Things the framework already does**. `string.Concat(a, b, c)` is optimized. `Enumerable.ToArray` on an `ICollection<T>` does a single `CopyTo`. `Dictionary<,>` indexing is already O(1) with low constant factors. Read the source before "optimizing" BCL calls.
- **Micro wins that don't move the macro needle**. A 10 ns savings × 1M calls/sec is 1% CPU. A 10 ns savings × 100 calls/sec is noise. Prove it on a flame graph.
- **DI resolution time** in normal apps. It's not your bottleneck; database round-trips and JSON are.
- **LINQ** — until profiling shows it. Modern LINQ is heavily optimized (.NET 9 rewrote a lot, .NET 10 more). The `foreach`+`List` rewrite of every LINQ chain is bikeshedding.
- **`Task` → `ValueTask`** without measurement. Often a wash or a regression.

If you can't draw the flame graph that justifies the change, don't ship the change.

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
- **Maoni Stephens** (.NET GC architect) — https://maoni0.medium.com/ — anything about GC internals.
- **Stephen Cleary** — https://blog.stephencleary.com/ — async / `Task` semantics, `ValueTask`, threading. Author of *Concurrency in C# Cookbook*.
- **Andrew Lock** — https://andrewlock.net/ — ASP.NET Core internals, source generators, deployment.
- **David Fowler** (.NET architect) — Twitter/X and dotnet/aspnetcore — Pipelines, channels, ASP.NET Core perf patterns.
- **Nick Chapsas** — YouTube *KeepCoding* — accessible perf walkthroughs and BenchmarkDotNet demos.
- **Konrad Kokosa — *Pro .NET Memory Management*** (Apress, 2018, still the definitive book on the runtime memory model and GC). Companion site: https://prodotnetmemory.com/
- **Sasha Goldshtein**, **Ben Watson — *Writing High-Performance .NET Code*** (2nd ed.) — practical perf engineering reference.
- **Marc Gravell** — protobuf-net, Pipelines, Dapper perf.

> If a tip in this doc disagrees with the latest Stephen Toub post, **the post wins**. Re-read it every November.
