# Platform database usage (`$PlatformDB`)

When [`platform-database-service`](../configuration/platform-database-service.md) is configured, scripts receive the `$PlatformDB` binding — the same `DatabaseBinding` class that wraps `$DB`, just constructed around a different, tenant-independent `IDatabaseService`. It exists for reference/master data shared by every tenant (a country list, a currency table) that would be wasteful and error-prone to duplicate per tenant. This page covers usage from JavaScript; provider configuration is in [configuration/platform-database-service.md](../configuration/platform-database-service.md), and the reasoning behind a separate database connection (rather than a per-query flag on `$DB`) is in [architecture/multi-tenancy.md](../architecture/multi-tenancy.md#platform-scoped-database-platformdb).

## It behaves exactly like `$DB`, minus the tenant

`$PlatformDB` exposes the identical method surface as [`$DB`](data-access.md) — `get`, `getFresh`, `save`, `update`, `delete`, `deleteById`, `findAll`, `list`, `unique`, `first`, `find`, `executeUpdate`, `newParams`, `markRollbackOnly` — because both are the same `DatabaseBinding` class. The one difference: no call on `$PlatformDB` ever reads or writes a `tenantId` column, regardless of the current request's tenant. A row saved via `$PlatformDB.save` has no `tenantId` set; `$PlatformDB.findAll`/`get`/etc. never filter by one.

```javascript
function execute(request, controlData) {
    var countries = $PlatformDB.findAll("Country");
    var account = $DB.get("Account", controlData.accountId);
    return { account: account, countryName: lookupCountryName(countries, account.countryCode) };
}
```

## `$DB` and `$PlatformDB` are independent transactions

Each gets its own request-scoped session and its own transaction, opened and committed/rolled back independently by the kernel (`ApiResourceHandler`), the same lifecycle `$DB` already uses — see [Transactions and request scope](data-access.md#transactions-and-request-scope). **A given request should write through only one of the two, never both** — reads from the other side are fine (e.g. reading `$PlatformDB` reference data while writing tenant data via `$DB`), but there is no cross-database atomicity: if a script wrote to both and one side's transaction failed to commit, the other side's write would not be rolled back with it. Which endpoints are allowed to write to `$PlatformDB` at all (as opposed to only ever reading from it) is a convention your application enforces, not something the platform gates today — see [What `$PlatformDB` does not do](#what-platformdb-does-not-do).

`$PlatformDB.markRollbackOnly()` exists and works exactly like `$DB`'s: it flags `$PlatformDB`'s own transaction as rollback-only, independently of `$DB`'s.

## Reading reference data

```javascript
var countries = $PlatformDB.findAll("Country");   // every row, no tenant filter
var country = $PlatformDB.get("Country", "MA");
```

## Writing reference data (from the endpoints authorized to do so)

```javascript
function execute(request, controlData) {
    $PlatformDB.save("Country", { code: controlData.code, name: controlData.name });
    return { ok: true };
}
```

There's nothing script-visible that distinguishes a "read-only" from a "read-write" caller — see the note above. Restrict which endpoints call `$PlatformDB.save`/`update`/`delete`/`deleteById` by convention, code review, or your own application-level authorization, the same way you'd restrict any other sensitive operation.

## What `$PlatformDB` does not do

Same restrictions as `$DB`: no `describeSecret`-equivalent metadata read, no `capabilities()` query. There is also no per-entity opt-out — every entity mapped into `platform-database-service`'s Hibernate mapping files is tenant-independent for every caller; there's no way to make some entities in that database tenant-scoped and others not. If you need both tenant-scoped and tenant-independent entities, map the tenant-scoped ones under `database-service` (reachable via `$DB`) and the tenant-independent ones under `platform-database-service` (reachable via `$PlatformDB`) — they are genuinely two different databases, not two views of the same one.

## If `$PlatformDB` is missing

`$PlatformDB` is injected only when `platform-database-service` is configured. If your script references it without a configured connection, the binding will be absent — see [configuration/platform-database-service.md](../configuration/platform-database-service.md).

## Next

- [Configuration: platform database service](../configuration/platform-database-service.md)
- [Data access (`$DB`)](data-access.md)
- [Architecture: multi-tenancy](../architecture/multi-tenancy.md)
