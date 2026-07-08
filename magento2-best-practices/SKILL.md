---
name: magento2-best-practices
description: Magento 2.4.x (target 2.4.9) development best practices. Use whenever writing, reviewing, or planning Magento/Adobe Commerce code — modules, plugins, observers, GraphQL resolvers, REST endpoints, db_schema changes, admin UI, cron/queues. Triggers on "build a Magento module/feature", "add GraphQL mutation/query", "fix Magento bug", "review this Magento code", "upgrade Magento".
---

# Magento 2 Best Practices (2.4.x, target 2.4.9)

You are working on Magento 2 code. Follow these rules. When a rule here is
version-sensitive, check [release-matrix.md](references/release-matrix.md).

## Non-negotiables

1. **Never edit core** (`vendor/`, `lib/internal`). Extend via a module in
   `app/code/Vendor/Module` (or composer package). Never edit vendored
   third-party modules except surgical, documented fixes.
2. **Narrowest extension mechanism wins** (Adobe playbook order): di.xml
   dependency/argument swap > observer (react to an existing event) > plugin
   (one public method) > preference (last resort — two modules preferring one
   class conflict, and preferences block core upgrades). Avoid `around`
   plugins unless you must short-circuit — they cost performance and hide
   bugs. Never plugin: final/private/static methods, constructors, virtual
   types, `NoninterceptableInterface` implementors.
3. **Constructor DI only.** Never call `ObjectManager::getInstance()` in
   product code (allowed only in factories/proxies/generated code, CLI
   bootstrap, and legacy test helpers). Use generated `Factory` classes for
   non-injectable (stateful/entity) objects, `Proxy` in di.xml for expensive
   constructor dependencies of hot classes.
4. **Service contracts**: depend on `Api/` interfaces (repositories, data
   interfaces), not concrete Models/ResourceModels/Collections of other
   modules. Mark your own stable surface `@api`. Return data interfaces, not
   arrays, from public services.
5. **Declarative schema only**: `etc/db_schema.xml` + generated
   `db_schema_whitelist.json` for tables/columns/indexes. Data changes =
   `Setup/Patch/Data` classes (schema patches only when declarative can't
   express it). Never InstallSchema/UpgradeSchema (removed pattern), never raw
   DDL in patches.
6. **No raw SQL against core tables** from modules. Use repositories or, at
   the lowest level, `ResourceConnection` with bound parameters — never string
   interpolation into queries.
7. **Escape all output** in templates via `$escaper->escapeHtml/escapeHtmlAttr/
   escapeUrl/escapeJs` — pick the context-correct method. `// @noEscape` only
   for values already escaped/known-safe, with a comment saying why.
8. **All state-changing controllers/mutations validate**: CSRF (form key) for
   frontend/admin POST, ACL (`_isAllowed`/`etc/acl.xml`) for adminhtml,
   authorization checks in GraphQL resolvers/webapi (never trust client IDs —
   derive customer/seller from context/token).
9. **Money paths are sacred**: single-writer services, idempotency keys,
   never float arithmetic on currency (use `Magento\Framework\Pricing` /
   bcmath / minor units), and never duplicate a posting an event observer
   already performs.
10. **Cache correctness**: anything rendering entity data declares identities
    (`IdentityInterface`); custom GraphQL queries declare cache identity or
    are explicitly uncacheable; never cache per-customer data in FPC.
11. **Every observer/plugin must be crash-safe**: no assumptions about admin
    session, request params, or area; wrap non-essential side-work in
    `try/catch (\Throwable)` + log. A broken observer on a core event takes
    the whole flow down.
12. **Secrets never in git**: `auth.json`, `env.php` values, API keys, crypt
    keys. Config that differs per environment goes in `env.php`/environment
    variables, not `config.php`.

## Decision guides

**Plugin vs Observer vs Preference**
- Change/extend behavior of ONE public method → plugin (prefer `after`).
- React to a domain event (order placed, creditmemo created) → observer on
  the event, side-effect only, never mutate the event's main flow implicitly.
- Replace an entire class implementation → preference (document why; check
  no other module preferences it: `bin/magento dev:di:info "Class\Name"`).

**GraphQL vs REST (webapi)**
- Storefront/customer-facing → GraphQL. Admin/integration/server-to-server →
  REST webapi.xml or message queue.
- GraphQL schema lives in the `*GraphQl` companion module; resolvers are THIN
  — validate input, check auth from `$context`, delegate to the base module's
  service contracts, map to output array. No business logic in resolvers.
- Lists that resolve per-item relations → batch resolver
  (`BatchResolverInterface`/`BatchServiceContractResolverInterface`) to avoid
  N+1.

**Sync vs Async**
- Anything slow/external (emails, exports, ERP sync, bulk ops) → message
  queue (`etc/communication.xml` + `etc/queue_*.xml`), consumer run by cron
  `consumers_runner` or supervisor. Never block a checkout/save on an
  external HTTP call.

## Standard workflows

**New module checklist**
`registration.php` + `etc/module.xml` (with `<sequence>` for load-order deps)
→ `composer.json` (if packaged) → code → `bin/magento setup:upgrade` →
verify `setup:db:status`. Frontend/admin changes need
`static-content:deploy` + `cache:flush` only in production mode.

**Schema change checklist**
Edit `etc/db_schema.xml` → regenerate whitelist
(`bin/magento setup:db-declaration:generate-whitelist --module-name=X`) →
`setup:upgrade` → confirm with `setup:db:status`. Destructive ops (drop
column/table) need explicit awareness that declarative schema WILL drop
removed elements.

**Before considering any change done**
- `vendor/bin/phpcs --standard=Magento2 <paths>` clean (or existing-issues-only)
- `bin/magento setup:upgrade && bin/magento cache:flush` runs clean
- Exercise the real flow (GraphQL call / page / admin action), then check
  `var/log/exception.log` and `var/log/system.log` — Magento often swallows
  the real error behind a generic `CouldNotSaveException`
- New/changed public service surface has at least a happy-path +
  failure-path test (see [quality-testing.md](references/quality-testing.md))

## Reference files (read on demand)

- [release-matrix.md](references/release-matrix.md) — 2.4.9/2.4.8 platform
  matrix (PHP/DB/search/cache), deprecations, upgrade path from 2.4.6
- [module-development.md](references/module-development.md) — DI details,
  plugins/observers depth, declarative schema & patches, module packaging
- [graphql-api.md](references/graphql-api.md) — resolver patterns, batch
  resolvers, caching, auth, input limits, REST/webapi, async APIs
- [performance-ops.md](references/performance-ops.md) — modes, deploy
  pipeline, cron/queues, indexers, Redis/Valkey, Varnish, collection
  anti-patterns
- [security.md](references/security.md) — patching cadence, admin hardening,
  CSP, code-level security rules, known attack classes
- [quality-testing.md](references/quality-testing.md) — unit/integration/
  api-functional tests, fixtures, PHPCS/PHPStan, CI, semver
