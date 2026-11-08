---
name: magento2-best-practices
description: Magento 2.4.x (target 2.4.9) development best practices. Use whenever writing, reviewing, or planning Magento/Adobe Commerce code — modules, plugins, observers, GraphQL resolvers, REST endpoints, db_schema changes, admin UI, layout XML/templates/RequireJS/Hyvä themes, payment methods, checkout totals/fees, shipping carriers, catalog data and URL rewrites, price rules and coupons, import/export, inventory/MSI, CLI commands, cron/queues/indexers. Triggers on "build a Magento module/feature", "add GraphQL mutation/query", "fix Magento bug", "review this Magento code", "upgrade Magento".
---

# Magento 2 Best Practices (2.4.x, target 2.4.9)

**First, establish the project's real versions**: Magento from `composer.lock`
(`magento/product-community-edition` / `magento/product-enterprise-edition`,
or `magento/framework`), PHP from `composer.json`/runtime, PHPUnit from
`composer.lock`. Apply version-specific rules for THAT version only. If the
project has its own conventions file (e.g. `CLAUDE.md`, `.claude/` notes),
it wins over this skill. On 2.3.x, the 2.4.x-specific references don't apply.

**Then establish where you are.** If the repository's `composer.json` has `"name": "magento/magento2ce"`,
it is a checkout of Magento Open Source itself (a fork for contributions): there core code in `app/code/Magento`
and `lib/internal` IS the code you change, non-negotiable 1 does not apply, and the contribution workflow of
the `magento-core-contrib` skill takes over (backward compatibility rules become the strictest constraint).
Otherwise it is a project or a module: core lives in `vendor/` and non-negotiable 1 holds. If neither fits
(a Mage-OS fork, a patched copy of core in `app/code/Magento`), ask the user before editing core paths.

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
   express it). Never write new InstallSchema/UpgradeSchema (legacy pattern), never raw
   DDL in patches.
6. **No raw SQL against core tables** from modules (MSI `inventory_*` tables
   included: [inventory-msi.md](references/inventory-msi.md)). Use repositories or, at
   the lowest level, `ResourceConnection` with bound parameters — never string
   interpolation into queries.
7. **Escape all output** in templates via `$escaper->escapeHtml/escapeHtmlAttr/
   escapeUrl/escapeJs` — pick the context-correct method. `// @noEscape` only
   for values already escaped/known-safe, with a comment saying why.
8. **All state-changing controllers/mutations validate**: CSRF (form key) for
   frontend/admin POST, ACL for adminhtml (own `ADMIN_RESOURCE` per controller,
   `<resource>` per system.xml section, `aclResource` per UI component),
   authorization checks in GraphQL resolvers/webapi (never trust client IDs —
   derive customer/seller from context/token).
9. **Money paths are sacred**: single-writer services, idempotency keys,
   never float arithmetic on currency (use `Magento\Framework\Pricing` /
   bcmath / minor units), and never duplicate a posting an event observer
   already performs. Payment methods: [payments.md](references/payments.md);
   stock and salability: [inventory-msi.md](references/inventory-msi.md).
10. **Cache correctness**: anything rendering entity data declares identities
    (`IdentityInterface`); custom GraphQL queries declare cache identity or
    are explicitly uncacheable; never cache per-customer data in FPC.
11. **Every observer/plugin must be crash-safe**: no assumptions about admin
    session, request params, or area; wrap non-essential side-work in
    `try/catch (\Throwable)` + log. A broken observer on a core event takes
    the whole flow down. This is for observers and plugins only: data patches
    and cron jobs let exceptions escape, queue handlers follow
    [backend-components.md](references/backend-components.md).
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
  failure-path test (see [quality-testing.md](references/quality-testing.md)),
  and every new test has been seen failing before it passes
- Changed PHP files pass `php -l` under the newest PHP the target supports
  with no `Deprecated:` line in the output (the exit code stays 0; 8.4+
  reports implicitly nullable parameters, and Magento turns every
  deprecation into an exception — see module-development.md)
- In a DDEV project with the magento2-ddev harness, run tests through its commands
  (`ddev magento-test-unit`, `ddev magento-test-integration`, ...; the `magento-ddev` skill); when
  magento-perf-check is available, run it on the changed code

## Reference files (read on demand)

- [release-matrix.md](references/release-matrix.md) — 2.4.9/2.4.8 platform
  matrix (PHP/DB/search/cache), deprecations, upgrade path from 2.4.6
- [module-development.md](references/module-development.md) — DI details,
  plugins/observers depth, declarative schema & patches, module packaging,
  stateless services (2.4.7+), PHP 8.4/8.5 deprecations
- [graphql-api.md](references/graphql-api.md) — resolver patterns, batch
  resolvers, caching, auth, input limits, REST/webapi, async APIs
- [performance-ops.md](references/performance-ops.md) — modes, deploy
  pipeline, cron/queues, indexers, Redis/Valkey, Varnish, collection
  anti-patterns
- [security.md](references/security.md) — patching cadence, admin hardening,
  admin ACL in code, uploads/paths/mass assignment/SSRF, CSP, code-level security rules, known attack classes
- [quality-testing.md](references/quality-testing.md) — unit/integration/
  api-functional tests, fixtures, application state tests, PHPCS/PHPStan, CI, semver
- [backend-components.md](references/backend-components.md) — extension
  attributes, EAV attribute patches, CLI commands, cron, logging, config scope,
  store emulation, queue consumers, custom indexers/mview, cache types, email
- [frontend.md](references/frontend.md) — view models, layout XML, customer-data
  sections, RequireJS mixins, jsLayout, themes/LESS, translations, Hyvä storefronts
- [admin-ui.md](references/admin-ui.md) — listing data providers, fulltext search,
  form data providers and modifiers, mass actions, inline edit
- [payments.md](references/payments.md) — payment gateway (Adapter, commands,
  can_* flags), payment actions and order state, vault, headless and webhooks
- [inventory-msi.md](references/inventory-msi.md) — salable qty, stock resolution,
  legacy CatalogInventory, reservations, ERP stock import, source selection
- [checkout-quote.md](references/checkout-quote.md) — totals collectors and fees
  (quote, order, invoice, credit memo), shipping carriers, quote loading, checkout fields
- [catalog-data.md](references/catalog-data.md) — product save scope in CLI/cron/REST,
  link field (`row_id` on Commerce), URL rewrite regeneration, search and filter attributes
- [promotions.md](references/promotions.md) — custom cart rule conditions, rule order,
  coupon usage and generation, catalog rule indexing
- [import-export.md](references/import-export.md) — custom import/export entities,
  behaviors, bunches, validation strategy, indexers after import
