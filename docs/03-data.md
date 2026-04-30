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

Sources:
- EF Core docs — overview: https://learn.microsoft.com/ef/core/
- Dapper: https://github.com/DapperLib/Dapper
- `Microsoft.Data.SqlClient`: https://learn.microsoft.com/sql/connect/ado-net/microsoft-ado-net-sql-server
- Npgsql: https://www.npgsql.org/
- Jon P Smith — *Entity Framework Core in Action*: https://www.thereformedprogrammer.net/

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
- **Cartesian explosion warning** is on by default; don't suppress it, fix the query.

#### Split vs single queries

| Use | When |
|---|---|
| **`AsSplitQuery()`** | `Include` of two or more *collection* navigations on the principal; row-count blowup on a single query plan would dominate I/O. |
| **`AsSingleQuery()`** | Single round-trip matters (latency-sensitive endpoint), point lookups, or you need a consistent snapshot — multiple statements can see different rows under read-committed. |

- If you set `UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)` globally, keep `AsSingleQuery()` as the per-query escape hatch.
- On EF Core 9 and earlier, split queries require a **stable, unique ordering** on the principal — add `OrderBy(x => x.Id)` if your query lacks one, or pagination becomes non-deterministic.
- EF Core 10 fixes split-query ordering consistency by automatically including the principal's key in the inner subquery `ORDER BY`, so `Skip`/`Take` + `Include` + `AsSplitQuery` no longer silently returns mismatched rows. Adding an explicit unique `OrderBy` is still the safe default.

### IQueryable boundaries

- **Do not leak `IQueryable<T>` out of the data layer.** It's a footgun: callers can compose expensive predicates, hold the context open, or force evaluation on a disposed context.
- Return `Task<T>`, `Task<IReadOnlyList<T>>`, or `IAsyncEnumerable<T>` from repositories / handlers.

### Compiled queries

```csharp
private static readonly Func<AppDb, Guid, CancellationToken, Task<User?>> GetUserById =
    EF.CompileAsyncQuery((AppDb db, Guid id, CancellationToken ct) =>
        db.Users.AsNoTracking().FirstOrDefault(u => u.Id == id));
```

- The lambda body uses the *synchronous* `FirstOrDefault` even though the compiled delegate is async — `EF.CompileAsyncQuery` translates the expression and returns a `Task<T>`. Don't write `FirstOrDefaultAsync` inside; it won't compile as an expression.
- The `CancellationToken` parameter is honored by the async pipeline, but the underlying expression doesn't see it — cancellation flows through ADO.NET when the delegate is invoked. Pass it; don't rely on inner LINQ to observe it.
- Use for ultra-hot, parameterized lookups. The modern EF query cache makes the delta smaller than it once was, but compiled queries still win on allocation.

### Raw SQL escape hatch

```csharp
var rows = await db.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE TenantId = {tenantId} AND Active = 1")
    .AsNoTracking()
    .ToListAsync(ct);
```

- Use `FromSqlInterpolated` / `ExecuteSqlInterpolated` — they parameterize. **Never** concatenate user input into `FromSqlRaw`.
- EF10: `ExecuteUpdateAsync` now accepts regular C# lambdas (not just expression trees) — use it for set-based updates without loading entities, especially when the setter chain is built dynamically (`Action<SetPropertyCalls<T>>` overload, no more `Expression.Lambda` gymnastics).
- EF10 also enables `ExecuteUpdate` over JSON-mapped **complex type** properties (e.g. `s.SetProperty(b => b.Details.Views, b => b.Details.Views + 1)`) — another reason to prefer complex types over owned entities for value objects.
- `SetProperty` cannot reference navigation properties (`s.SetProperty(b => b.Owner.Name, ...)` won't translate) — flatten or split the update.

> **Caveat — `ExecuteUpdateAsync` / `ExecuteDeleteAsync` bypass the change tracker, `SaveChanges` interceptors, the outbox, and any `ISaveChangesInterceptor` you rely on for auditing or domain events.** Tracked entities in the same context go stale silently. Rules:
>
> - Use them for **infrastructural** set-based ops (TTL purges, status flips, GDPR scrubs) where you intentionally want to skip materialization.
> - **Do not** use them on aggregates whose write path emits domain events / outbox rows from the change tracker — you'll publish nothing.
> - If the same context already loaded affected rows, call `ChangeTracker.Clear()` or reload entries (`entry.ReloadAsync()`) before further work.

### Modeling: complex types vs owned entities vs primary constructors

| Use | When |
|---|---|
| **Complex type** (EF 8+, `ComplexProperty`) | Value object with **no identity** and no separate lifecycle (`Address`, `Money`, `DateRange`). Stored inline; cannot be queried as a top-level set. Supports records and shared instances. |
| **Owned entity** (`OwnsOne`/`OwnsMany`) | Dependent has identity, ownership semantics, or you need a separate table / collection. Cosmos provider treats related types as **owned by default**. |
| **Aggregate root** | Has its own identity, lifecycle, and is a query/aggregation boundary. |

- Prefer **complex types** for new value-object modeling on relational providers — they replace the owned-entity-as-value-object hack and don't carry a hidden shadow key.
- On **EF Core 10** specifically, complex types now support **optional values, JSON mapping (`ComplexProperty(...).ToJson()`), .NET structs, and `ExecuteUpdate`** — owned entities should be reserved for cases with true identity / separate lifecycle.
- **Primary constructors** (C# 12+): fine on DTOs, records, value objects, and complex types. On **EF tracked entities**, only use them after you've verified materialization works for your provider, that lazy-loading proxies (if any) can still subclass, and that navigation properties remain settable. The safe default is a parameterless ctor (public or private) plus init-only properties.

### ChangeTracker hygiene

- Short-lived contexts. No long-lived tracked graphs.
- `ChangeTracker.AutoDetectChangesEnabled = false` inside tight bulk loops, then call `ChangeTracker.DetectChanges()` once before `SaveChangesAsync`.
- `ChangeTracker.Clear()` between logical batches in a worker.

### `SaveChangesAsync` semantics

- One implicit transaction per call (unless ambient). Batches inserts/updates/deletes into minimal round-trips on SQL Server and PostgreSQL.
- Returns affected row count — check it for optimistic concurrency on raw updates.
- Throws `DbUpdateConcurrencyException` / `DbUpdateException` — catch the *specific* ones.

Sources:
- EF Core 10 — what's new: https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew
- EF Core 10 — complex types: https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew#complex-types
- EF Core 10 — `ExecuteUpdateAsync` non-expression lambda: https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew#executeupdateasync-now-accepts-a-regular-non-expression-lambda
- EF Core 10 — `ExecuteUpdate` over JSON columns: https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew#executeupdate-support-for-relational-json-columns
- EF Core 10 — split-query ordering: https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew#more-consistent-ordering-for-split-queries
- EF Core 10 — named query filters: https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew#named-query-filters
- EF Core 9 — what's new: https://learn.microsoft.com/ef/core/what-is-new/ef-core-9.0/whatsnew
- EF Core performance: https://learn.microsoft.com/ef/core/performance/
- Single vs split queries: https://learn.microsoft.com/ef/core/querying/single-split-queries
- `EF.CompileAsyncQuery`: https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.ef.compileasyncquery
- `ExecuteUpdate` / `ExecuteDelete` (incl. navigation caveat): https://learn.microsoft.com/ef/core/saving/execute-insert-update-delete
- Complex types (EF 8 release notes): https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#value-objects-using-complex-types
- EF team blog: https://devblogs.microsoft.com/dotnet/category/entity-framework/
- Jon P Smith — entity construction & DDD with EF: https://www.thereformedprogrammer.net/
- Andrew Lock — EF Core deep-dives: https://andrewlock.net/

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
- Default pool size is **1024** (the `poolSize` arg is optional); once exceeded, EF falls back to non-pooled creation rather than blocking.
- **Trade-offs**: context must have a simple constructor, any instance state leaks across requests if you're not careful, and interceptors/loggers captured by the first instance stick. Prefer stateless contexts.
- **Multi-tenant gotcha**: pooled contexts are effectively reused across requests, so `OnConfiguring` runs **once per pooled instance** — read tenant/user from a scoped service injected at request time, not from `IHttpContextAccessor` inside `OnConfiguring`.
- `AddPooledDbContextFactory<T>` combines pooling + factory for Blazor/workers.

Sources:
- `DbContext` lifetime: https://learn.microsoft.com/ef/core/dbcontext-configuration/
- `DbContext` pooling: https://learn.microsoft.com/ef/core/performance/advanced-performance-topics#dbcontext-pooling
- Andrew Lock on `AddDbContextFactory`: https://andrewlock.net/

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

- Named query filters: multiple global filters (e.g. soft-delete + tenant) toggleable by name — use `IgnoreQueryFilters("SoftDelete")` selectively instead of all-or-nothing. ([docs](https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew#named-query-filters))
- Native `json` column type on SQL Server 2025 / Azure SQL: enabled when you use `UseAzureSql(...)` or `UseSqlServer(..., o => o.UseCompatibilityLevel(170))`. Existing `nvarchar` JSON columns auto-migrate to the `json` type on the next migration unless you opt out via `HasColumnType("nvarchar(max)")` per property.
- Vector search GA: `SqlVector<float>` mapping plus `EF.Functions.VectorDistance` on SQL Server 2025 / Azure SQL — usable for AI/RAG workloads without dropping to ADO.NET.

### Scripts vs bundles vs `Database.Migrate()`

| Artifact | When |
|---|---|
| `dotnet ef migrations script --idempotent -o migrate.sql` | DBAs need to inspect/approve raw SQL before apply; you want to diff schema in PRs. |
| `dotnet ef migrations bundle` (self-contained `efbundle` executable) | First-class production deployment. One artifact moves through CI/CD, no .NET SDK on the target, runs idempotently against the connection string you pass. Default for new pipelines. |
| `context.Database.MigrateAsync()` at startup | Single-instance dev/test only. Never in multi-instance prod. |

### Zero-downtime: expand → migrate → contract

Schema changes ship across **multiple releases**. The deploy that introduces the change must keep the *previous* app version working.

- **Expand** (release N): add new column/table as **nullable** (or with a safe default); add new index `CONCURRENTLY` (Postgres) / `WITH (ONLINE = ON)` (Azure SQL/Enterprise SQL Server); deploy schema *before* code.
- **Dual-write / backfill** (release N+1): app writes to **both** old and new shapes; backfill job populates new column for historical rows in batches with throttling; monitor lag.
- **Switch reads** (release N+2): app reads from new shape, still dual-writes; verify.
- **Contract** (release N+3): drop old column / constraint / table after the previous version is fully retired and rollback window has passed.

- Long backfills: chunk by primary key, commit per batch, log progress, make resumable. Never `UPDATE` a billion rows in one transaction.
- Renames are **not** zero-downtime — model them as add-new + backfill + drop-old.
- Adding a `NOT NULL` column with no default rewrites the whole table on most engines; ship it as nullable + backfill + tighten in a later release.

Sources:
- Applying migrations (scripts, bundles, runtime): https://learn.microsoft.com/ef/core/managing-schemas/migrations/applying
- Migration bundles (canonical docs): https://learn.microsoft.com/ef/core/managing-schemas/migrations/applying#bundles
- EF Core 10 — JSON type support: https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew#json-type-support
- EF Core 10 — vector search support: https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew#vector-search-support
- Andrew Lock — running EF migrations on startup safely: https://andrewlock.net/
- Postgres `CREATE INDEX CONCURRENTLY`: https://www.postgresql.org/docs/current/sql-createindex.html

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

Sources:
- Concurrency conflicts: https://learn.microsoft.com/ef/core/saving/concurrency
- Npgsql `xmin` concurrency token: https://www.npgsql.org/efcore/modeling/concurrency.html
- Jon P Smith on EF concurrency patterns: https://www.thereformedprogrammer.net/

---

## 6. Transactions & Unit of Work

- **`SaveChangesAsync` is the UoW.** Stop wrapping it in your own `IUnitOfWork` interface — it adds nothing.
- **`BeginTransactionAsync`** only when:
  - You need multiple `SaveChanges` calls in one atomic unit.
  - You span providers (e.g., EF + Dapper on the same connection — share the transaction via `db.Database.UseTransaction(txn)`).
  - You need a specific isolation level (`IsolationLevel.Snapshot`, `Serializable`).
- **Avoid `TransactionScope`** unless you need ambient enlistment for legacy code. Async + `TransactionScope` requires `TransactionScopeAsyncFlowOption.Enabled` — easy to forget.

### Distributed transactions: don't

> **Architectural rule:** do **not** rely on distributed (two-phase / MSDTC / promoted) transactions across heterogeneous resources — DB + Service Bus, DB + Redis, DB + HTTP, two databases — in modern cloud apps. The coordinator is a single point of failure, most managed services don't enlist, and Linux .NET has no DTC at all.

- Keep transactions **local to one database connection**.
- For "save row + emit event / call service" use the **transactional outbox + idempotent consumer** pattern. See [`06-cloud-native.md`](./06-cloud-native.md) for the outbox/idempotency guidance and the `OutboxRelay` background service.
- For multi-aggregate / multi-service workflows, use **sagas** (Wolverine, MassTransit) with compensating actions, not distributed commits.

### Cross-system consistency → Outbox

Never do `SaveChanges()` then `ServiceBus.SendAsync()`. Either:

1. **Transactional outbox**: write domain event rows in the same `SaveChanges`, a relay publishes and marks sent. Use MassTransit, Wolverine, or roll your own (it's ~200 lines).
2. **Inbox / idempotency key** on the consumer side: `(message_id)` unique index; dedupe before side-effects.

Idempotency keys on **HTTP writes** too: `Idempotency-Key` header → table with `(key, tenant) UNIQUE`, store the response for replay.

Sources:
- EF transactions: https://learn.microsoft.com/ef/core/saving/transactions
- David Fowler on `TransactionScope` + async pitfalls: https://gist.github.com/davidfowl
- Jimmy Bogard on the outbox pattern: https://www.jimmybogard.com/refactoring-towards-resilience-evaluating-coupling/
- Cross-link: [06-cloud-native.md](./06-cloud-native.md)

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

- Enable retry for transient failures (Azure SQL, cloud Postgres). On Azure SQL specifically prefer **`UseAzureSql(...)`** (EF 9+) — it auto-configures Azure-tuned resiliency (transient error list, retry policy) without you maintaining the magic-numbers list. Reserve `UseSqlServer(...EnableRetryOnFailure(...))` for on-prem / generic SQL Server.
- **EF 9+ escape hatch**: when `UseSqlServer` is called by code you don't control (scaffolded `DbContext`, third-party library), call `optionsBuilder.ConfigureSqlEngine(c => c.EnableRetryOnFailureByDefault())` to flip retries on without rewriting the provider call.
- **Caveat**: execution strategies disable user-initiated transactions. Wrap in `strategy.ExecuteAsync(async () => { using var txn = ...; })`.

> **.NET Aspire**: if you're composing services with Aspire, prefer the Aspire EF client integrations — `Aspire.Microsoft.EntityFrameworkCore.SqlServer` (`AddSqlServerDbContext` / `EnrichSqlServerDbContext`), `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL` (`AddNpgsqlDbContext` / `EnrichNpgsqlDbContext`), and `Aspire.Microsoft.EntityFrameworkCore.Cosmos` (`AddCosmosDbContext`). They auto-enable retries (`DisableRetry` defaults to `false`), health checks, OpenTelemetry, and structured logging — don't double-configure `EnableRetryOnFailure` on top. Use the `EnrichXxxDbContext` variants when you need to keep your own `AddDbContext` registration.

Sources:
- EF Core performance: https://learn.microsoft.com/ef/core/performance/
- Connection resiliency: https://learn.microsoft.com/ef/core/miscellaneous/connection-resiliency
- SQL Server provider — connection resiliency (`UseAzureSql`, `ConfigureSqlEngine`): https://learn.microsoft.com/ef/core/providers/sql-server/#connection-resiliency
- Aspire SQL Server EF client integration: https://aspire.dev/integrations/databases/efcore/sql-server/sql-server-client/
- Aspire PostgreSQL EF client integration: https://aspire.dev/integrations/databases/efcore/postgres/postgresql-client/
- Aspire Cosmos DB EF client integration: https://aspire.dev/integrations/cloud/azure/azure-cosmos-db/azure-cosmos-db-get-started/
- Ben Adams — high-perf .NET: https://blog.marcgravell.com/ and https://twitter.com/ben_a_adams
- Shay Rojansky — query plans, Npgsql perf: https://www.roji.org/

---

## 8. Provider notes

### SQL Server / Azure SQL
- Use `Microsoft.Data.SqlClient` (not `System.Data.SqlClient` — deprecated).
- **Azure SQL: prefer `UseAzureSql(connectionString)`** — it bundles the Azure-specific transient-error list and resiliency settings. Drop manual `EnableRetryOnFailure` tuning unless you have a measured reason.
- For generic SQL Server (on-prem, containers, IaaS): `UseSqlServer(cs, sql => sql.EnableRetryOnFailure(...))`. If the call site is owned by scaffolded/third-party code, use `optionsBuilder.ConfigureSqlEngine(c => c.EnableRetryOnFailureByDefault())` (EF 9+).
- Connection string essentials: `Encrypt=True`, `TrustServerCertificate=False`, prefer `Authentication=Active Directory Default` over passwords.
- EF10: native `json` column type triggered by `UseAzureSql(...)` or `UseSqlServer(..., o => o.UseCompatibilityLevel(170))`; existing `nvarchar` JSON columns auto-migrate on the next migration unless you opt out per property.
- EF10: `SqlVector<float>` mapping + `EF.Functions.VectorDistance` for SQL Server 2025 / Azure SQL vector search.

### PostgreSQL (Npgsql)
- `Npgsql.EntityFrameworkCore.PostgreSQL` — tracks EF major versions.
- `UseXminAsConcurrencyToken()` gives you optimistic concurrency for free.
- Array columns (`int[]`, `text[]`), `jsonb`, range types, `LTREE` — all mapped. Use them instead of inventing tables.
- Connection pooling is client-side; use PgBouncer in transaction mode for high fan-out. Disable prepared statements or configure `Max Auto Prepare` accordingly.

### Cosmos DB (EF Core Cosmos provider)

Cosmos is a **document database** wearing an EF face. Modeling rules from relational EF do not transfer.

**Use EF Cosmos when**
- Your aggregate maps cleanly to a single document (order + lines, user + profile).
- Your access pattern is **point reads** (`id` + partition key) and partition-scoped queries.
- You want LINQ + change tracking + migrations-style container provisioning over the SDK.

**Skip EF Cosmos and drop to the SDK when**
- You need fine-grained RU control, bulk executor, change feed processor, or hierarchical partition keys with custom routing.
- You're doing cross-partition analytics (use the analytical store / Synapse link / Fabric mirroring instead).

**Modeling rules**
- **Always model the partition key explicitly** (`HasPartitionKey`). Default to a high-cardinality, query-aligned key (`/tenantId`, `/customerId`). Bad partition keys are *unfixable* without a re-import.
- Related types are **owned by default** — `OwnsMany(o => o.Lines)` becomes an embedded array, not a separate container. Reach for separate root entities only when lifecycle truly differs.
- Every read should be a **point read** (`db.Orders.FindAsync(id, partitionKey)`) or a query *with* the partition key in the predicate. Cross-partition queries fan out and burn RUs.
- No joins. Denormalize. Accept duplication; storage is cheap, RUs are not.
- No transactions across partition keys. Use **transactional batches** within a single partition key, or sagas across them.
- Concurrency: ETag (`_etag`) — map via `IsETagConcurrency()`.
- Migrations don't exist; you ship schema in code and version documents (`schemaVersion` field + read-side upgrade).

**EF9 / EF10 deltas worth knowing**
- **EF9+**: partition-key predicates in `Where` are extracted automatically — `db.Orders.Where(o => o.TenantId == t && o.Id == id)` becomes a true point read (`ReadItem`), not a cross-partition query. Stop hand-rolling `WithPartitionKey(...)` for the common shape.
- **EF9+**: hierarchical partition keys via `HasPartitionKey(a, b, c)` — supports the multi-level partitioning Cosmos exposes natively.
- **EF10**: full-text search, hybrid (RRF) search, and vector search GA on the Cosmos provider; Cosmos operations also flow through the standard `ExecutionStrategy` for retry-aware transactional batches.

**Cost discipline**
- Watch RU charge per query in App Insights / diagnostics — it's the latency signal that matters.
- Bulk writes: `AllowBulkExecution = true` on the underlying `CosmosClient`, then fan out via `Task.WhenAll`.
- Prefer the SDK directly when an endpoint is RU-sensitive and you need to control consistency level, indexing policy, or partition key path per operation.

Sources:
- EF Core Cosmos provider: https://learn.microsoft.com/ef/core/providers/cosmos/
- EF Core 9 — Cosmos updates: https://learn.microsoft.com/ef/core/what-is-new/ef-core-9.0/whatsnew#azure-cosmos-db-for-nosql
- EF Core 10 — what's new (Cosmos full-text/hybrid/vector search): https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew
- Cosmos partitioning guidance: https://learn.microsoft.com/azure/cosmos-db/partitioning-overview
- Cosmos request units: https://learn.microsoft.com/azure/cosmos-db/request-units
- EF team blog — Cosmos updates: https://devblogs.microsoft.com/dotnet/category/entity-framework/

### SQLite
- **Tests only**, and only when you're not using provider-specific SQL. Dialect differences (no `FOR UPDATE`, different `json_` functions, case sensitivity) will bite.
- For real tests → Testcontainers.

Sources:
- SQL Server provider: https://learn.microsoft.com/ef/core/providers/sql-server/
- Npgsql / EFCore.PG: https://www.npgsql.org/efcore/
- Cosmos provider: https://learn.microsoft.com/ef/core/providers/cosmos/
- Shay Rojansky (Npgsql lead): https://www.roji.org/

---

## 9. Async discipline

- **Every** data-access call is `async` + accepts `CancellationToken`.
- Never `.Result`, `.Wait()`, `GetAwaiter().GetResult()` on a `DbContext` call. Deadlocks in sync contexts, thread-pool starvation in server code.
- Flow `HttpContext.RequestAborted` / `IHostApplicationLifetime.ApplicationStopping` into queries. Long reports become cancellable for free.
- `ConfigureAwait(false)` is irrelevant in ASP.NET Core (no sync context). Still required in libraries.
- `ConfigureAwaitOptions` (.NET 8+, e.g. `SuppressThrowing`, `ContinueOnCapturedContext`, `ForceYielding`) is a `Task`-level option for specific helpers, **not** a replacement for `.ConfigureAwait(false)` on library data-access calls.

Sources:
- Async guidance (David Fowler): https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md
- Stephen Toub — async deep-dives: https://devblogs.microsoft.com/dotnet/author/stephentoub/
- Ben Adams on async perf pitfalls: https://blog.marcgravell.com/

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

Sources:
- Dapper: https://github.com/DapperLib/Dapper
- Khalid Abuhakmeh — EF + Dapper interop: https://khalidabuhakmeh.com/
- Nick Chapsas — Dapper vs EF benchmarks: https://nickchapsas.com/

---

## 11. Repository / Specification

- **Default: no repository over EF.** `DbContext` + `DbSet<T>` is already a repository + UoW. Another layer just launders `IQueryable` into named methods and loses composability.
- **Thin repo when**:
  - You enforce tenant/row-level filters centrally (`.Where(x => x.TenantId == current)`).
  - You gate raw-SQL queries behind a method so they're audited.
  - You need to stub *one* method in a test without containerizing.
- **Specification pattern** (`ISpecification<T>` + evaluator): only pays off when specs are genuinely reused across queries and endpoints. Otherwise it's LINQ-with-extra-steps. Avoid the Ardalis-style giant base class in small apps.

Sources:
- Jimmy Bogard — repository critique: https://www.jimmybogard.com/should-you-use-the-repository-pattern/
- Jon P Smith on EF + DDD: https://www.thereformedprogrammer.net/

---

## 12. CQRS-lite

- **Reads**: project directly from `DbContext` in the endpoint/handler. `AsNoTracking` + `Select`. No command bus.
- **Writes**: one class per command. Takes `DbContext` (or factory), validates, mutates, `SaveChangesAsync`, publishes events (via outbox).
- **MediatR**: don't add it *just* for indirection. Costs: allocation per request, stack-trace noise, DI registration sprawl, licensing friction since v12. Use it only if you genuinely need pipeline behaviors (logging, validation, tx) applied uniformly — and even then, ASP.NET Core filters / endpoint filters often suffice.

Sources:
- Jimmy Bogard on MediatR licensing: https://www.jimmybogard.com/automapper-and-mediatr-going-commercial/
- Andrew Lock — endpoint filters as MediatR alternatives: https://andrewlock.net/

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
- **Never materialize before filtering or projecting.** `db.Users.ToListAsync().Result.Where(u => u.Active)` pulls the whole table into memory and filters in C#. Filter and project in the `IQueryable` first, *then* `ToListAsync`.
- **Never enable lazy-loading proxies** (`UseLazyLoadingProxies()`) in services or APIs. They turn property accesses into invisible round-trips, break `AsNoTracking`, and produce N+1 patterns that don't show up in code review. Use explicit `Include` or projection.

Sources:
- EF testing guidance (incl. "don't use InMemory"): https://learn.microsoft.com/ef/core/testing/
- Testcontainers for .NET: https://dotnet.testcontainers.org/
- Respawn: https://github.com/jbogard/Respawn
- Jon P Smith — testing EF Core: https://www.thereformedprogrammer.net/

---

## 14. Caching

### Pick the right cache for the job

| Cache | When |
|---|---|
| **`OutputCache`** (ASP.NET Core middleware) | HTTP **response** caching — varies by route/query/header, supports tag-based eviction. Use for read endpoints whose entire response is cacheable. |
| **`HybridCache`** (.NET 9 GA, .NET 10 hardened) | App **data** caching with L1 in-memory + optional L2 distributed (Redis), built-in **stampede protection**, and tags. The default for new app-data caching. |
| **`IDistributedCache`** (Redis / SQL Server) | Simple distributed K/V when you don't need L1 or stampede protection (sessions, short tokens, small shared blobs). Most new code should reach for `HybridCache` instead. |
| **`IMemoryCache`** | Single-process, tiny, per-instance state. Fine inside a singleton service; not a cluster cache. |
| **EF "second-level" cache libraries** | Avoid. There is **no official EF L2 cache**. Third-party interceptors (e.g. `EFCoreSecondLevelCacheInterceptor`) miss invalidation from raw SQL, `ExecuteUpdate`, triggers, and other contexts. Prefer explicit cache-aside at the query you want to cache. |

```csharp
// dotnet add package Microsoft.Extensions.Caching.Hybrid
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
- `ExecuteUpdate`/`ExecuteDelete` and raw SQL **do not** trigger your cache invalidation hooks — invalidate explicitly at those call sites.

Sources:
- HybridCache (and `Microsoft.Extensions.Caching.Hybrid` NuGet): https://learn.microsoft.com/aspnet/core/performance/caching/hybrid
- OutputCache middleware: https://learn.microsoft.com/aspnet/core/performance/caching/output
- Distributed caching: https://learn.microsoft.com/aspnet/core/performance/caching/distributed
- Andrew Lock on HybridCache: https://andrewlock.net/
- Steve Gordon — HybridCache internals: https://www.stevejgordon.co.uk/

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

Sources:
- Data Protection: https://learn.microsoft.com/aspnet/core/security/data-protection/
- Always Encrypted (SQL Server): https://learn.microsoft.com/sql/relational-databases/security/encryption/always-encrypted-database-engine
- Managed identity for Azure SQL: https://learn.microsoft.com/azure/azure-sql/database/authentication-aad-overview
- Meziantou on Data Protection & column encryption: https://www.meziantou.net/

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
- What's new in EF Core 9 — https://learn.microsoft.com/ef/core/what-is-new/ef-core-9.0/whatsnew
- EF Core performance docs — https://learn.microsoft.com/ef/core/performance/
- EF Core testing guidance (incl. "don't use InMemory") — https://learn.microsoft.com/ef/core/testing/
- `DbContext` lifetime — https://learn.microsoft.com/ef/core/dbcontext-configuration/
- Migrations in production — https://learn.microsoft.com/ef/core/managing-schemas/migrations/applying
- Concurrency conflicts — https://learn.microsoft.com/ef/core/saving/concurrency
- Connection resiliency — https://learn.microsoft.com/ef/core/miscellaneous/connection-resiliency
- SQL Server provider — connection resiliency — https://learn.microsoft.com/ef/core/providers/sql-server/#connection-resiliency
- Aspire EF client integrations (SQL Server / PostgreSQL / Cosmos) — https://aspire.dev/integrations/databases/efcore/sql-server/sql-server-client/
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
- Ben Adams — high-perf .NET, allocations, kestrel — https://blog.marcgravell.com/ and https://twitter.com/ben_a_adams
