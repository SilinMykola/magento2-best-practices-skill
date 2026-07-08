# Performance & operations (Magento 2.4.x)

## Modes & deployment

- Production runs in **production mode** only (static files served from cache only, precompiled DI, errors logged never displayed). `deploy:mode:set production` auto-runs di:compile + static-content:deploy. Mode lives in env.php (MAGE_MODE).
- **Pipeline (build-server) deployment**: build needs code only — no install, no DB. Steps: `composer install --no-dev` → `setup:di:compile` → `composer dump-autoload -o --apcu` → `setup:static-content:deploy` (config known at build time via committed `app/etc/config.php` from `app:config:dump`). `env.php` stays out of git; sensitive values via `config:sensitive:set`/env vars.
- Canonical low-downtime deploy on prod: `maintenance:enable` → pull code → stop consumers → `app:config:import` → (only if schema/data changed) `setup:upgrade --keep-generated` → `cache:flush` → `maintenance:disable`. No DB changes → skip maintenance entirely. `--keep-generated` preserves build artifacts; `static-content:deploy -f` only outside production mode.
- DI precompilation saves up to ~25% request time; undeployed static content massively slows responses. After deploy: reindex, cache clean, then WARM the cache (curl homepage/top categories/PDPs) before traffic.
- SCD strategies: `quick` (default), `standard` (everything), `compact` (disk-saving). Parallelize `--jobs N`; scope `--language/--theme/--area` to skip unused locales.

## Cron & queues

- Install cron as the filesystem owner: `bin/magento cron:install` — must fire every minute, never as root. Without it: scheduled indexers, queue consumers, sitemaps, emails silently stop. Built-in groups: default, index, consumers; heavy groups get `use_separate_process=1` / `cron:run --group=X`.
- Consumers via env.php `cron_consumers_runner`: `cron_run: true`, `max_messages` ~10000 (NEVER 0/infinite — memory), list consumers explicitly or empty=all. 2.4.4+: `multiple_processes` per queue. Alternative: Supervisor/systemd running `queue:consumers:start`.
- **RabbitMQ (amqp) over the MySQL queue** for anything beyond trivial volume — the db adapter polls and doesn't scale with concurrent consumers. (2.4.9 supports RabbitMQ 4.2; ActiveMQ Artemis flagged as long-term direction.)

## Indexers

- All indexers **Update by Schedule** in production (`indexer:set-mode schedule`) — mview changelog (`*_cl` tables) + partial reindex via the index cron group. EXCEPTION: **Customer Grid must stay Update on Save**.
- Switch modes during low traffic (adds/removes DB triggers). Monitor backlog with `indexer:status`; recover invalid indexes with `indexer:reset` + `indexer:reindex`.

## Caching

- Redis/Valkey: at minimum separate DB numbers (default cache 0, page cache 1, sessions 2); production-grade = separate instances for sessions vs cache; Unix sockets on single-node. Don't touch Lua defaults (`use_lua=0`, `use_lua_on_gc=1`) — corruption risk on 2.4.7/2.4.8.
- Multi-node → enable **L2 cache** (local `/dev/shm` copy + Redis version-hash validation; a standard instance otherwise ships ~300KB cache per request over the network). Optional `use_stale_cache` for block_html/layout.
- **Varnish for FPC** in production; built-in FPC is a dev fallback (`system/full_page_cache/caching_application = 2`, VCL via `varnish:vcl:generate`). Tag-based PURGE invalidation, high TTL.
- All cache types enabled in production. Prefer `cache:clean` (tagged entries only) over `cache:flush` (wipes the whole backend incl. co-tenants → cold-cache stampede). Warm after any flush.

## Code-level performance

- Collection anti-patterns (ban in review): `load()` in loops (multiplied uncached queries — filter ONE collection instead); `getFirstItem()` on unbounded collections (NO implicit LIMIT 1 — use `setPageSize(1)` or repository + SearchCriteria); `addAttributeToSelect('*')` (loads every EAV attribute — name what you use).
- Prefer repositories/service contracts (`ProductRepositoryInterface` + `SearchCriteriaBuilder`) over direct collections in module code. Raw SQL is a PHPCS violation (`Magento2.SQL.RawQuery`); when truly needed use ResourceConnection with bound params.
- Move slow work off the request: async sales emails (Sales Emails > General), async order-grid indexing (Developer > Grid Settings), `setup:config:set --config-async 1`, message queues for exports/ERP/external HTTP. Never block checkout on an external call.

## Server tuning (Adobe-documented values)

- PHP: `memory_limit=1G` (prod), `realpath_cache_size=10M`, `realpath_cache_ttl=7200`; opcache: `memory_consumption=512`, `max_accelerated_files=60000`, `validate_timestamps=0` (then MUST reload php-fpm each deploy), **`save_comments=1` mandatory** (code generation reads comments), `enable_cli=1`. APCu on + `dump-autoload -o --apcu`. Same settings for CLI and FPM.
- Frontend: minify CSS/JS/HTML = Yes; JS bundling only when first-paint isn't critical (bundle loads whole on first view). Serve `pub/static` + `pub/media` directly from nginx/CDN, not PHP.
- MySQL: `innodb_buffer_pool_instances=8` (≥1GB each), `max_connections` 300–1000, `innodb_thread_concurrency = 2*(CPUs+disks)`.
