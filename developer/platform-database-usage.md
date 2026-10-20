Created: 2026-10-16
Last updated: 2026-10-16
Version: 2 — supersedes v1: `$PlatformDB`'s write surface is now gated by `atomicityScope`, not left to application convention — see [`atomicityScope` and the read-only/full split](#atomicityscope-and-the-read-onlyfull-split).

# Platform database usage (`$PlatformDB`)

When [`platform-database-service`](../configuration/platform-database-service.md) is configured, scripts receive the `$PlatformDB` binding, constructed around a different, tenant-independent `IDatabaseService` than `$DB`. It exists for reference/master data shared by every tenant (a country list, a currency table) that would be wasteful and error-prone to duplicate per tenant. This page covers usage from JavaScript; provider configuration is in [configuration/platform-database-service.md](../configuration/platform-database-service.md), and the reasoning behind a separate database connection (rather than a per-query flag on `$DB`) is in [architecture/multi-tenancy.md](../architecture/multi-tenancy.md#platform-scoped-database-platformdb).

## `atomicityScope` and the read-only/full split

Which binding your endpoint actually gets depends on its `atomicityScope` field in `mappings.json` (default `"tenant"` when the field is absent) — see [architecture: atomicity across `$DB`, `$Audit`, and `$Queue`](../architecture/atomicity-db-audit-queue-v10-2026-10-20.md#21-which-transaction-carries-the-guarantee--atomicityscope) for the full reasoning:

| `atomicityScope` | `$PlatformDB` | `$DB` |
| --- | --- | --- |
| `"tenant"` (default) | **Read-only**: `get`, `getFresh`, `findAll`, `find`, `list`, `unique`, `first`, `newParams` only — `ReadOnlyDatabaseBinding`, no `save`/`update`/`delete`/`deleteById`/`executeUpdate`, no `markRollbackOnly` | Full read/write, as always |
| `"platform"` | Full read/write — the identical method surface as [`$DB`](data-access.md): `get`, `getFresh`, `save`, `update`, `delete`, `deleteById`, `findAll`, `list`, `unique`, `first`, `find`, `executeUpdate`, `newParams`, `markRollbackOnly` | **Not bound at all** — `typeof $DB === 'undefined'` |

```json
{
  "mappings": {
    "api": {
      "/reference-data/refresh": {
        "POST": { "handler": "scripts/refresh-reference-data.js", "atomicityScope": "platform" }
      }
    }
  }
}
```

**This is a breaking change if you already write to `$PlatformDB` today.** Before this field existed, every endpoint got a full read/write `$PlatformDB` unconditionally, and which ones were "allowed" to write was an application convention, never enforced by the platform. Now, an endpoint that writes to `$PlatformDB` without declaring `atomicityScope: "platform"` gets `Unknown identifier: save` (or `update`/`delete`/`deleteById`/`executeUpdate`) the moment it tries — audit your existing endpoints for `$PlatformDB.save`/`update`/`delete`/`deleteById`/`executeUpdate` calls and add the field before relying on this in production. Conversely, an endpoint declared `atomicityScope: "platform"` that also uses `$DB` loses `$DB` entirely (`$DB is not defined`) — it has no declared reason left to touch the tenant database.

Calling code that only ever **reads** `$PlatformDB` needs no change — reading stays available either way.

```javascript
function execute(request, controlData) {
    var countries = $PlatformDB.findAll("Country");   // works with either atomicityScope
    var account = $DB.get("Account", controlData.accountId);
    return { account: account, countryName: lookupCountryName(countries, account.countryCode) };
}
```

## `$DB` and `$PlatformDB` are independent transactions

Each gets its own request-scoped session and its own transaction, opened and committed/rolled back independently by the kernel (`ApiResourceHandler`), the same lifecycle `$DB` already uses — see [Transactions and request scope](data-access.md#transactions-and-request-scope). There is no cross-database atomicity between them: a request writes through only one of the two (whichever `atomicityScope` names), the other side stays read-only or absent — see the table above.

`$PlatformDB.markRollbackOnly()` exists and works exactly like `$DB`'s — only on an `atomicityScope: "platform"` endpoint, since that's the only case where `$PlatformDB` is the full `DatabaseBinding` rather than the read-only subset: it flags `$PlatformDB`'s own transaction as rollback-only, independently of `$DB`'s.

## Reading reference data

```javascript
var countries = $PlatformDB.findAll("Country");   // every row, no tenant filter
var country = $PlatformDB.get("Country", "MA");
```

## Writing reference data (`atomicityScope: "platform"` endpoints only)

```javascript
function execute(request, controlData) {
    $PlatformDB.save("Country", { code: controlData.code, name: controlData.name });
    return { ok: true };
}
```

This only works on an endpoint declared `atomicityScope: "platform"` — see the table above. On any other endpoint, `$PlatformDB` has no write methods at all; the call fails immediately with `Unknown identifier: save`, not a permissions error at runtime.

## What `$PlatformDB` does not do

Same restrictions as `$DB`: no `describeSecret`-equivalent metadata read, no `capabilities()` query. There is also no per-entity opt-out — every entity mapped into `platform-database-service`'s Hibernate mapping files is tenant-independent for every caller; there's no way to make some entities in that database tenant-scoped and others not. If you need both tenant-scoped and tenant-independent entities, map the tenant-scoped ones under `database-service` (reachable via `$DB`) and the tenant-independent ones under `platform-database-service` (reachable via `$PlatformDB`) — they are genuinely two different databases, not two views of the same one.

## If `$PlatformDB` is missing

`$PlatformDB` is injected only when `platform-database-service` is configured. If your script references it without a configured connection, the binding will be absent — see [configuration/platform-database-service.md](../configuration/platform-database-service.md).

## Next

- [Architecture: atomicity across `$DB`, `$Audit`, and `$Queue`](../architecture/atomicity-db-audit-queue-v10-2026-10-20.md) — the full `atomicityScope` mechanism.
- [Configuration: platform database service](../configuration/platform-database-service.md)
- [Data access (`$DB`)](data-access.md)
- [Architecture: multi-tenancy](../architecture/multi-tenancy.md)
