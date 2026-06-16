# Cache for pgdog

## Architecture

pgdog intercepts read queries in the frontend layer, before they reach the backend pool. On a cache hit the response is returned to the client directly — PostgreSQL is never contacted. On a miss the query executes normally and the response is stored in the cache backend, then forwarded to the client.

The storage layer is abstracted behind a `CacheStorage` trait, making the backend swappable. Redis is the current implementation.

**Policy resolution** is field-by-field and three-tiered. For each field (`mode`, `ttl`, `key`), the first non-absent value wins:

```
SQL comment hint  →  pgdog.cache session parameter  →  global config
   (highest)                                             (lowest)
```

**Cache key** is an XXH3 hash of the database name, normalized query text (comments stripped, whitespace collapsed, allocation-free), and any bind parameters. A `key=NAME` hint overrides this entirely with the hash of just the name string.

**Error responses are never cached.** Queries inside explicit transactions always bypass the cache. Storage errors degrade gracefully to a passthrough — a Redis outage does not affect query serving.

---

## Implementation

### Configuration (`pgdog-config/src/cache.rs`)

**`[general.cache]`** — global cache config:
- `enabled: bool` — is caching enabled (default `false`)
- `policy: CachePolicy` — caching policy (default `no_cache`)
- `ttl: u64` — default TTL in seconds (default `300`)
- `backend: CacheBackend` — cache backend (default `redis`)
- `max_result_size: usize` — max cached result bytes (default `0` = unlimited)

**`[general.cache.redis]`**:
- `url: String` — Redis connection URL (default `redis://localhost:6379`)
- `cache_key_prefix: String` — prefix prepended to every Redis key (default `pgdog:`)
- `operation_timeout: NonZeroU64` — timeout in milliseconds for individual Redis operations (GET/SET/ping) (default `2000`)

Example TOML:
```toml
[general.cache]
enabled = true
policy  = "cache"
ttl     = 300

[general.cache.redis]
url               = "redis://localhost:6379"
cache_key_prefix  = "pgdog:"
operation_timeout = 2000
```

### Module Map (`pgdog/src/frontend/cache/`)

| File | Responsibility |
|------|---------------|
| `mod.rs` | Global `Cache` singleton (`OnceCell<Arc<Cache>>`), `try_read_cache`, `save_response_in_cache` |
| `storage/redis.rs` | `RedisCacheStorage`: async Redis backend, background reconnect, per-call timeouts |
| `directive.rs` | Parse and merge `CacheDirective` from SQL comment and connection parameter |
| `context.rs` | `CacheContext`: per-query response buffer and error flag held in `QueryEngineContext` |
| `integration.rs` | `cache_check()`: route guard, directive resolution, Redis GET/SET dispatch |
| `hashing.rs` | `compute_cache_key_hash`: XXH3 of `database + normalized query + bind params` |
| `wire.rs` | Deserialize flat wire-byte blob back into `Vec<Message>` |

### Dependencies

```toml
fred = { version = "10", features = ["enable-rustls"] }
xxhash-rust = { version = "0.8", features = ["xxh3"] }
```

---

## Key Design Decisions

| Decision | Choice |
|----------|--------|
| Interception point | Between `route_query()` and `before_execution()` in `handle()` |
| Cache config scope | **Global** (`config.general.cache`) |
| Redis client | `fred` crate (async-native, tokio integration) |
| Cacheable queries | Only reads (`route.is_read()`) and those that are not in transaction |
| Cache policy resolution | Field-by-field merge: SQL comment → connection param → global config |
| Cache HIT flow | Deserialize wire bytes → `Vec<Message>` → replay each through `process_server_message()` |
| Cache MISS flow | Normal execute → capture response via `CacheContext` → store in cache → respond |
| Cache key | XXH3 hash of `database_name + normalized query + bind params`; or XXH3 of just `key=NAME` when the hint is present |
| Query normalization | On-the-fly in hasher: comments stripped, whitespace collapsed (except inside string literals), no `String` allocated |
| Wire format | Full PostgreSQL wire messages stored as raw bytes (one concatenated buffer) |
| Config hotswap | `is_actual()` reads live config internally; only those config parameters, that require rebuild, triggers it |

---

## How to Control Cache

### Priority Order

Cache directives are resolved field-by-field. For each field (`mode` and `ttl`), the first non-`None` value wins:

```
SQL comment  →  pgdog.cache parameter  →  global config
(highest)                                    (lowest)
```

**Example:** Comment specifies `ttl=60` but no mode; parameter specifies `cache` but no TTL.
- Final mode: `cache` (from parameter)
- Final TTL: `60` (from comment)

### SQL Comments

Add a `/* pgdog_cache: … */` comment anywhere in your query. Arguments can appear in any order:

```sql
-- Bypass cache for this query
/* pgdog_cache: no_cache */ SELECT * FROM users WHERE id = 1;

-- Cache with default TTL
/* pgdog_cache: cache */ SELECT * FROM products WHERE category = 'electronics';

-- Cache with custom TTL in seconds
/* pgdog_cache: cache ttl=300 */ SELECT * FROM orders;

-- Force-cache: always repopulate, skip cache lookup
/* pgdog_cache: force_cache ttl=300 */ SELECT * FROM orders;

-- Only specify TTL, inherit mode from connection parameter or global config
/* pgdog_cache: ttl=60 */ SELECT * FROM sessions;

-- Custom cache key (bypasses automatic key computation)
/* pgdog_cache: cache key=my_query ttl=120 */ SELECT * FROM analytics;

-- Cache directive alongside other pgdog directives
/* pgdog_cache: force_cache ttl=10 pgdog_role: replica */ SELECT * FROM analytics;
```

SQL comments are skipped on-the-fly while hashing, with no intermediate `String` allocation. Surrounding whitespace left by a stripped comment is also collapsed, so `"/* pgdog_cache: cache */ SELECT 1"` and `"SELECT 1"` produce exactly the same cache key. Spaces inside string literals (`WHERE name = 'hello world'`) are never affected.

### Connection Parameter

Set `pgdog.cache` at connection time (via DSN options) or with `SET` after connecting. Arguments can appear in any order. Lower priority than a SQL comment, higher than global config.

```sql
SET pgdog.cache = 'no_cache';             -- all queries in this connection bypass cache
SET pgdog.cache = 'cache';                -- cache all queries with default TTL
SET pgdog.cache = 'cache ttl=300';        -- cache all queries with 5-minute TTL
SET pgdog.cache = 'force_cache ttl=300';  -- force cache all queries with 5-minute TTL
SET pgdog.cache = 'ttl=120';              -- only specify TTL, inherit mode from global config
```

Or via DSN:

```sh
# Cache all queries with 5-minute TTL
psql postgresql://postgres:postgres@127.0.0.1:5432/postgres?options=-c%20pgdog.cache%3Dcache%20ttl%3D300
```

---

## Tests

### Running unit tests

```sh
cargo nextest run -p pgdog frontend::cache
```

### Integration tests (PostgreSQL + Redis + pgdog required)

```sh
bash integration/cache/run.sh
```

Or if you already have pgdog running on port 6432 with that config:

```sh
bash integration/cache/dev.sh
```
