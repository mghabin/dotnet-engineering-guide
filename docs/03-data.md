# Data Access in .NET 10 — Opinionated Best Practices

Audience: senior .NET engineers shipping services on **.NET 10 / EF Core 10** (LTS, Nov 2025). Lean, do/don't, no ceremony.

> **Working thesis**: EF Core is the default. Dapper is a scalpel, not a hammer. Raw `Microsoft.Data.SqlClient` / `Npgsql` is an escape hatch. Most teams that "replaced EF with Dapper for perf" actually replaced it with bad SQL — and lost the change tracker, migrations, and provider abstraction in the process.

---

## 1. Choosing the tool

| Use | When |
|---|---|
| **EF Core 10** | Default. Transactional write paths, aggregates with navigations, anything with migrations, multi-provider code. |
| **Dapper** | Hot read paths with hand-tuned SQL, reporting queries, dynamic SQL built at runtime, stored-proc-heavy shops, fan-out joins EF translates poorly. |
| **Raw `SqlClient` / `Npgsql`** | Bulk copy (`SqlBulkCopy`, `NpgsqlBinaryImporter`), `COPY`, streaming LOBs, advisory locks, `LISTEN/NOTIFY`, provider-specific features EF doesn't expose. |

**Do**
- Start with EF. Profile. Replace specific queries with Dapper/raw when a trace justifies it.
- Mix freely: EF `DbContext.Database.GetDbConnection()` hands you the same connection Dapper wants. No separate pool needed.
- Keep writes in EF (change tracker + concurrency + outbox), move reads to Dapper if needed (CQRS-lite).

**Don't**
- Don't "Dapper everything" — you're rewriting a change tracker, migrations, and identity map badly.
- Don't "EF everything" — translating a 7-table reporting query through LINQ is a code review tax on your team forever.
- Don't build your own micro-ORM. Dapper exists. It's 2 files.

---

## 2. EF Core core practices (EF10)

### Tracking

```csharp
// Read-only list endpoint — never track.
var users = await db.Users
    .AsNoTracking()
    .Where(u => u.TenantId == tenantId)
    .Select(u => new UserListItem(u.Id, u.Email, u.DisplayName))
    .ToListAsync(ct);
```

- **`AsNoTracking()`** on every query whose result you won't `SaveChanges` on. Non-negotiable.
- **`AsNoTrackingWithIdentityResolution()`** when the result graph contains the same entity multiple times via `Include` and you need reference equality — rare; usually project instead.
- Set `QueryTrackingBehavior.NoTrackingWithIdentityResolution` at the context level only if 90%+ of queries are reads. Opt *into* tracking for writes.

### Projection beats Include

```csharp
// DON'T — materializes whole Order + OrderLines graph you don't need.
var orders = await db.Orders.Include(o => o.Lines).ToListAsync();

// DO — SELECT exactly what the DTO needs. EF generates one tight SQL statement.
var orders = await db.Orders
    .Where(o => o.CustomerId == id)
    .Select(o => new OrderSummary(
        o.Id, o.PlacedAt, o.Lines.Sum(l => l.Qty * l.UnitPrice)))
    .ToListAsync(ct);
```

- **Project to DTOs** for reads. `Include` is for write-path aggregates you're about to mutate.
- **Split queries** (`AsSplitQuery()`) when including multiple collections — avoids Cartesian explosion. Configure globally via `UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)` if that's your default.
- **Cartesian explosion warning** is on by default; don't suppress it, fix the query.

### IQueryable boundaries

- **Do not leak `IQueryable<T>` out of the data layer.** It's a footgun: callers can compose expensive predicates, hold the context open, or force evaluation on a disposed context.
- Return `Task<T>`, `Task<IReadOnlyList<T>>`, or `IAsyncEnumerable<T>` from repositories / handlers.

### Compiled queries

```csharp
private static readonly Func<AppDb, Guid, CancellationToken, Task<User?>> GetUserById =
    EF.CompileAsyncQuery((AppDb db, Guid id, CancellationToken ct) =>
        db.Users.AsNoTracking().FirstOrDefault(u => u.Id == id));
```

Use for ultra-hot, parameterized lookups. Modern EF query cache makes the delta smaller than it once was, but compiled queries still win on allocation.

### Raw SQL escape hatch

```csharp
var rows = await db.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE TenantId = {tenantId} AND Active = 1")
    .AsNoTracking()
    .ToListAsync(ct);
```

- Use `FromSqlInterpolated` / `ExecuteSqlInterpolated` — they parameterize. **Never** concatenate user input into `FromSqlRaw`.
- EF10: `ExecuteUpdateAsync` now accepts regular C# lambdas (not just expression trees) — use it for set-based updates without loading entities.

### ChangeTracker hygiene

- Short-lived contexts. No long-lived tracked graphs.
- `ChangeTracker.AutoDetectChangesEnabled = false` inside tight bulk loops, then call `ChangeTracker.DetectChanges()` once before `SaveChangesAsync`.
- `ChangeTracker.Clear()` between logical batches in a worker.

### `SaveChangesAsync` semantics

- One implicit transaction per call (unless ambient). Batches inserts/updates/deletes into minimal round-trips on SQL Server and PostgreSQL.
- Returns affected row count — check it for optimistic concurrency on raw updates.
- Throws `DbUpdateConcurrencyException` / `DbUpdateException` — catch the *specific* ones.

---

## 3. DbContext lifetime

| Scenario | Lifetime |
|---|---|
| ASP.NET Core request | **Scoped** via `AddDbContext` |
| Blazor Server, Blazor WebAssembly hosted | **`IDbContextFactory<T>`** — scope ≠ request |
| Background workers, `IHostedService` | **`IDbContextFactory<T>`** per unit of work |
| Parallel `Task.WhenAll` over a shared context | **Never.** Factory per task. `DbContext` is not thread-safe. |
| Minimal APIs / gRPC | Scoped |

```csharp
builder.Services.AddDbContextFactory<AppDb>(o => o.UseNpgsql(cs));
// in handler:
await using var db = await factory.CreateDbContextAsync(ct);
```

### Pooling

```csharp
builder.Services.AddDbContextPool<AppDb>(o => o.UseSqlServer(cs), poolSize: 128);
```

- Pooling removes per-request context construction cost. Real but small (single-digit % typically).
- **Trade-offs**: context must have a simple constructor, any instance state leaks across requests if you're not careful, and interceptors/loggers captured by the first instance stick. Prefer stateless contexts.
- `AddPooledDbContextFactory<T>` combines pooling + factory for Blazor/workers.

---

## 4. Migrations

**Do**
- Check in `Migrations/` and the `ModelSnapshot`. Review them like code — they *are* code.
- For prod, generate **idempotent SQL**: `dotnet ef migrations script --idempotent -o migrate.sql`. Apply via your deploy pipeline, not `context.Database.Migrate()` at startup.
- Baseline legacy DBs with `migrations add Initial --from-migration 0` and a hand-edited initial.
- Use `HasData` for **reference data only** (lookup tables). Not for tenants, users, or anything environmental.

**Don't**
- Don't call `Database.Migrate()` at startup in multi-instance deployments — race conditions, partial failures on cold start, first pod wins. Fine for single-instance dev/test.
- Don't edit an applied migration. Add a new one.
- Don't let `dotnet ef` auto-apply in CI without a review gate. Schema changes are production incidents in waiting.

### EF10 notes

- Named query filters: multiple global filters (e.g. soft-delete + tenant) toggleable by name — use `IgnoreQueryFilters("SoftDelete")` selectively instead of all-or-nothing.
- Native JSON column type on SQL Server/Azure SQL: prefer the `json` type over `nvarchar(max)` for new JSON-mapped properties.

---

## 5. Concurrency

```csharp
public class Order
{
    public Guid Id { get; set; }
    [Timestamp] public byte[] RowVersion { get; set; } = default!;
}
```

- **Optimistic by default.** `[Timestamp]` / `IsRowVersion()` on SQL Server; `xmin` system column on PostgreSQL via `UseXminAsConcurrencyToken()` (Npgsql).
- Use `[ConcurrencyCheck]` on individual columns only when row-version isn't available.
- Pessimistic (`SELECT ... FOR UPDATE`, `UPDLOCK`) only for genuine contention hotspots — inventory decrements, seat reservations. Always with a timeout.

### Retry loop

```csharp
for (var attempt = 0; attempt < 3; attempt++)
{
    try { await db.SaveChangesAsync(ct); break; }
    catch (DbUpdateConcurrencyException ex)
    {
        foreach (var entry in ex.Entries)
        {
            var dbValues = await entry.GetDatabaseValuesAsync(ct);
            if (dbValues is null) throw; // deleted
            entry.OriginalValues.SetValues(dbValues);
            // reapply business rule on top of fresh values
        }
    }
}
```

Retry is a **business decision** — don't silently last-write-wins. Surface the conflict if the user edited the same fields.

---

## 6. Transactions & Unit of Work

- **`SaveChangesAsync` is the UoW.** Stop wrapping it in your own `IUnitOfWork` interface — it adds nothing.
- **`BeginTransactionAsync`** only when:
  - You need multiple `SaveChanges` calls in one atomic unit.
  - You span providers (e.g., EF + Dapper on the same connection — share the transaction via `db.Database.UseTransaction(txn)`).
  - You need a specific isolation level (`IsolationLevel.Snapshot`, `Serializable`).
- **Avoid `TransactionScope`** unless you need ambient enlistment for legacy code. Async + `TransactionScope` requires `TransactionScopeAsyncFlowOption.Enabled` — easy to forget.

### Cross-system consistency → Outbox

Never do `SaveChanges()` then `ServiceBus.SendAsync()`. Either:

1. **Transactional outbox**: write domain event rows in the same `SaveChanges`, a relay publishes and marks sent. Use MassTransit, Wolverine, or roll your own (it's ~200 lines).
2. **Inbox / idempotency key** on the consumer side: `(message_id)` unique index; dedupe before side-effects.

Idempotency keys on **HTTP writes** too: `Idempotency-Key` header → table with `(key, tenant) UNIQUE`, store the response for replay.

---

## 7. Performance

### N+1

- EF logs a warning for client evaluation and cartesian explosion. Elevate to **error** in dev:
  ```csharp
  o.ConfigureWarnings(w => w.Throw(RelationalEventId.MultipleCollectionIncludeWarning));
  ```
- Add a MiniProfiler / OpenTelemetry SQL span count assertion in integration tests for hot endpoints.

### Materialization

- `ToListAsync` buffers everything. Fine for bounded pages.
- `AsAsyncEnumerable()` / `await foreach` streams rows — use for exports, long scans, SSE/NDJSON endpoints. Holds the connection open; don't stream to slow clients without a timeout.
- `ToListAsync` + `Select` projection is almost always the right answer for APIs.

### Indexes

```csharp
modelBuilder.Entity<Order>()
    .HasIndex(o => new { o.TenantId, o.PlacedAt })
    .IncludeProperties(o => new { o.Status, o.Total })   // covering index
    .HasFilter("[Status] <> 'Deleted'");                  // filtered
```

- Design indexes from query plans, not intuition. `EXPLAIN ANALYZE` (Postgres) / `SET STATISTICS IO ON` (SQL Server).
- Composite index order = selectivity order. Tenant-scoped apps: `TenantId` first, always.

### Bulk

- EF's batched `SaveChanges` handles hundreds of rows fine. For tens of thousands:
  - **EFCore.BulkExtensions** (`BulkInsertAsync`, `BulkUpdateAsync`, `BulkMerge`) for SQL Server/PG.
  - **`linq2db.EntityFrameworkCore`** for set-based `Merge`/`InsertFromQuery`.
  - **`SqlBulkCopy` / `NpgsqlBinaryImporter`** for millions — bypass EF entirely.
- Prefer **`ExecuteUpdateAsync` / `ExecuteDeleteAsync`** over load-then-save for set-based ops. EF10 lets you use normal lambdas.

### Connection resiliency

```csharp
o.UseSqlServer(cs, sql => sql.EnableRetryOnFailure(
    maxRetryCount: 5,
    maxRetryDelay: TimeSpan.FromSeconds(10),
    errorNumbersToAdd: null));
```

- Enable retry for transient failures (Azure SQL, cloud Postgres).
- **Caveat**: execution strategies disable user-initiated transactions. Wrap in `strategy.ExecuteAsync(async () => { using var txn = ...; })`.

---

## 8. Provider notes

### SQL Server / Azure SQL
- Use `Microsoft.Data.SqlClient` (not `System.Data.SqlClient` — deprecated).
- Azure SQL: always `EnableRetryOnFailure`, `Encrypt=True`, `TrustServerCertificate=False`.
- EF10: native `json` type mapping — opt in for new models.

### PostgreSQL (Npgsql)
- `Npgsql.EntityFrameworkCore.PostgreSQL` — tracks EF major versions.
- `UseXminAsConcurrencyToken()` gives you optimistic concurrency for free.
- Array columns (`int[]`, `text[]`), `jsonb`, range types, `LTREE` — all mapped. Use them instead of inventing tables.
- Connection pooling is client-side; use PgBouncer in transaction mode for high fan-out. Disable prepared statements or configure `Max Auto Prepare` accordingly.

### SQLite
- **Tests only**, and only when you're not using provider-specific SQL. Dialect differences (no `FOR UPDATE`, different `json_` functions, case sensitivity) will bite.
- For real tests → Testcontainers.

---

## 9. Async discipline

- **Every** data-access call is `async` + accepts `CancellationToken`.
- Never `.Result`, `.Wait()`, `GetAwaiter().GetResult()` on a `DbContext` call. Deadlocks in sync contexts, thread-pool starvation in server code.
- Flow `HttpContext.RequestAborted` / `IHostApplicationLifetime.ApplicationStopping` into queries. Long reports become cancellable for free.
- `ConfigureAwait(false)` is irrelevant in ASP.NET Core (no sync context). Still required in libraries.

---

## 10. Dapper patterns

```csharp
public sealed class OrderReadRepo(IDbConnectionFactory f)
{
    public async Task<IReadOnlyList<OrderRow>> RecentAsync(Guid tenantId, CancellationToken ct)
    {
        await using var conn = await f.OpenAsync(ct);
        return (await conn.QueryAsync<OrderRow>(
            new CommandDefinition(
                "SELECT id, placed_at, total FROM orders WHERE tenant_id = @tenantId ORDER BY placed_at DESC LIMIT 100",
                new { tenantId }, cancellationToken: ct))).AsList();
    }
}
```

- **Always parameterize.** Dapper makes it easy; string concat is a CVE.
- **`IDbConnectionFactory`** abstraction: opens a connection from config, nothing more. Don't invent a repository layer just for Dapper — a static class of query methods is fine.
- **Multi-mapping** (`QueryAsync<A,B,TResult>`) for joined rows; `splitOn:` is the one knob to learn.
- **`QueryMultipleAsync`** for dashboards — one round trip, N result sets.
- Share the connection with EF when you need a cross-tool transaction: `conn = db.Database.GetDbConnection(); txn = db.Database.CurrentTransaction?.GetDbTransaction();`

---

## 11. Repository / Specification

- **Default: no repository over EF.** `DbContext` + `DbSet<T>` is already a repository + UoW. Another layer just launders `IQueryable` into named methods and loses composability.
- **Thin repo when**:
  - You enforce tenant/row-level filters centrally (`.Where(x => x.TenantId == current)`).
  - You gate raw-SQL queries behind a method so they're audited.
  - You need to stub *one* method in a test without containerizing.
- **Specification pattern** (`ISpecification<T>` + evaluator): only pays off when specs are genuinely reused across queries and endpoints. Otherwise it's LINQ-with-extra-steps. Avoid the Ardalis-style giant base class in small apps.

---

## 12. CQRS-lite

- **Reads**: project directly from `DbContext` in the endpoint/handler. `AsNoTracking` + `Select`. No command bus.
- **Writes**: one class per command. Takes `DbContext` (or factory), validates, mutates, `SaveChangesAsync`, publishes events (via outbox).
- **MediatR**: don't add it *just* for indirection. Costs: allocation per request, stack-trace noise, DI registration sprawl, licensing friction since v12. Use it only if you genuinely need pipeline behaviors (logging, validation, tx) applied uniformly — and even then, ASP.NET Core filters / endpoint filters often suffice.

---

## 13. Testing data access

**Do**
- **Testcontainers** (`Testcontainers.PostgreSql`, `Testcontainers.MsSql`) for integration tests. Real engine, real SQL, real migrations. Fast enough with image caching.
- `Respawn` to reset between tests instead of rebuilding containers per test.
- **Snapshot the generated SQL** for hot queries:
  ```csharp
  o.LogTo(Console.WriteLine, LogLevel.Information)
   .EnableSensitiveDataLogging()   // dev/test only
   .EnableDetailedErrors();
  ```
  Assert on the captured SQL string in a golden file; catches silent plan regressions.

**Don't**
- **`Microsoft.EntityFrameworkCore.InMemory`** — it lies. No relational semantics, no constraints, no transactions, no SQL. Tests pass that prod fails. Microsoft officially recommends against it.
- SQLite-as-Postgres-substitute. Different dialect, different bugs.
- Unit-test the DbContext. Test your *handlers* against a real DB.

---

## 14. Caching

- **`IMemoryCache`** for single-process, tiny, per-instance state.
- **`HybridCache`** (.NET 9 GA, improved in .NET 10): L1 in-memory + L2 distributed (Redis), with stampede protection and tag-based invalidation. Replace hand-rolled `IMemoryCache` + `IDistributedCache` sandwiches.
  ```csharp
  builder.Services.AddHybridCache();
  var dto = await cache.GetOrCreateAsync(
      $"user:{id}",
      async ct => await db.Users.AsNoTracking()
          .Where(u => u.Id == id)
          .Select(u => new UserDto(...)).FirstAsync(ct),
      tags: new[] { $"tenant:{tenantId}" },
      cancellationToken: ct);
  ```

**Invalidation**
- **Cache-aside** is the default. Write path: `SaveChangesAsync` → `cache.RemoveByTagAsync("tenant:{id}")`.
- Keep TTLs short (seconds–minutes) for anything mutable. Long TTL + manual invalidation = incidents.
- **Do not** put EF-tracked entities in cache. DTOs or primitives only. Tracked graphs across cache are identity-map bombs.

**Second-level cache libraries (EFCoreSecondLevelCacheInterceptor etc.)**
- Tempting, dangerous. Invalidation spans all code paths including raw SQL, `ExecuteUpdate`, triggers. Prefer explicit cache-aside at the query you actually want to cache.

---

## 15. Secrets, connection strings, at-rest encryption

- **Connection strings**: never in appsettings committed to source. Use User Secrets (dev), Key Vault / AWS Secrets Manager / environment (prod). `AddAzureKeyVault` or Workload Identity on AKS.
- Don't put passwords in DSNs when the platform supports Managed Identity (Azure SQL, Azure PG) — use `Authentication=Active Directory Default` / `Azure Identity` tokens.
- **ASP.NET Core Data Protection**: configure a persisted key ring (Key Vault / blob storage + Key Vault key) in multi-instance deployments. Default file-system keys on a container = tokens invalidated on pod restart.
- **Column-level encryption**:
  - SQL Server **Always Encrypted** for deterministic lookup on sensitive columns; column-master-key in Key Vault.
  - PostgreSQL: `pgcrypto` for app-layer, or encrypt in code via Data Protection `IDataProtector` and store `byte[]` (deterministic = searchable, randomized = safer).
  - For PII tokenization, keep keys in KMS/HSM, not in the DB.
- Log **parameterized SQL**, never values. `EnableSensitiveDataLogging` is a dev/test switch; gate it behind `IsDevelopment()`.

---

## TL;DR defaults

- EF Core 10, scoped context (web) / factory (workers+Blazor), `AsNoTracking` + projection by default.
- `EnableRetryOnFailure`, idempotent migration scripts applied by the pipeline, row-version concurrency.
- Outbox for events, idempotency keys for HTTP writes.
- Testcontainers for tests; InMemory is banned.
- HybridCache for shared caching; invalidate by tag on write.
- Dapper for the five hot queries that deserve it. Not for everything.

---

## Sources

**Authoritative**
- What's new in EF Core 10 — https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew
- EF Core performance docs — https://learn.microsoft.com/ef/core/performance/
- EF Core testing guidance (incl. "don't use InMemory") — https://learn.microsoft.com/ef/core/testing/
- `DbContext` lifetime — https://learn.microsoft.com/ef/core/dbcontext-configuration/
- Migrations in production — https://learn.microsoft.com/ef/core/managing-schemas/migrations/applying
- Concurrency conflicts — https://learn.microsoft.com/ef/core/saving/concurrency
- Connection resiliency — https://learn.microsoft.com/ef/core/miscellaneous/connection-resiliency
- HybridCache — https://learn.microsoft.com/aspnet/core/performance/caching/hybrid
- Data Protection — https://learn.microsoft.com/aspnet/core/security/data-protection/
- `Microsoft.Data.SqlClient` — https://learn.microsoft.com/sql/connect/ado-net/microsoft-ado-net-sql-server
- Npgsql / EFCore.PG docs — https://www.npgsql.org/efcore/
- EF team blog (Arthur Vickers, Shay Rojansky) — https://devblogs.microsoft.com/dotnet/category/entity-framework/
- .NET 10 / EF Core 10 release notes — https://github.com/dotnet/core/blob/main/release-notes/10.0/10.0.0/efcoreanddata.md
- Testcontainers for .NET — https://dotnet.testcontainers.org/
- Dapper — https://github.com/DapperLib/Dapper

**Community / practitioner**
- Shay Rojansky (Npgsql lead, EF team) — https://www.roji.org/
- Jon P Smith — *Entity Framework Core in Action* and https://www.thereformedprogrammer.net/
- Andrew Lock — https://andrewlock.net/ (config, startup, DbContext lifetime, HybridCache)
- Jimmy Bogard — https://www.jimmybogard.com/ (outbox, CQRS, repository critique)
- Khalid Abuhakmeh — https://khalidabuhakmeh.com/ (EF + Dapper interop)
- Nick Chapsas — https://nickchapsas.com/ and YouTube (perf, Dapper vs EF benchmarks)
- Meziantou — https://www.meziantou.net/ (async, Data Protection, column encryption)
- Steve Gordon — https://www.stevejgordon.co.uk/ (async, caching, HybridCache internals)
- David Fowler — gists on `TransactionScope`, async pitfalls — https://gist.github.com/davidfowl
