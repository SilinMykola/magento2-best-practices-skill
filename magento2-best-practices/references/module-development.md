# Module & backend development (Magento 2.4.x)

## Extension mechanism hierarchy (Adobe Implementation Playbook order)

Pick the NARROWEST mechanism that works:
1. **di.xml dependency/argument replacement** — swap a collaborator; can effectively change protected behavior without interception.
2. **Observer** — react to an existing event; less overhead than plugins.
3. **Plugin** — the common fallback; public methods only.
4. **Preference** — "greedy", only when nothing else works; two modules preferring one class conflict, and preferences block core updates on upgrade.
5. **Source patch** — last resort, brittle.

## Plugins

- Cannot intercept: final methods/classes, non-public methods, static methods, `__construct`/`__destruct`, virtual types, objects instantiated before `Magento\Framework\Interception` bootstraps, `NoninterceptableInterface` implementors. (A plugin itself MAY be a virtual type; its target may not.)
- Prefer `before` (modify args; return array or null) and `after` (must return result; receives result + original args). **Avoid `around`** — extra stack frames, hides bugs; only legitimate use is terminating further plugins + the original method.
- Plugins MUST be stateless; SHOULD NOT change the intercepted object's state; SHOULD NOT target your own module's classes (call your code directly); SHOULD NOT target data objects.
- Execution order: lowest→highest `sortOrder`; ties resolved by module load order (`module.xml` sequence).
- Check for competing preferences/plugins: `bin/magento dev:di:info "Class\Name"`.

## Observers

- Declare in **area-specific** `etc/<area>/events.xml` (adminhtml, frontend, crontab, graphql, webapi_rest) — global registration is NOT recommended.
- Observers contain NO business logic — delegate to injected services. Small, single-purpose.
- NEVER mutate values (incl. objects) passed with the event; no assumptions about other observers or ordering; no events fired in constructors; avoid cyclic event chains.
- Prefer disabling a core observer (`disabled="true"`) over overriding its class.
- Crash-safety: never assume an admin session/user or request params exist; wrap non-essential side-work in `try/catch (\Throwable)` + log.

## Dependency injection

- Constructor injection for stable services (injectables/singletons). **Method injection** for per-call, non-injectable inputs (entities, request data) — pass as parameters.
- Newables (entities needing an ID) come from generated **Factories**. Expensive optional deps → **Proxy** configured in di.xml. Never type-hint `\Foo\Proxy` or interceptors in constructors — di.xml only.
- **Virtual types**: named, differently-configured instances of a class, di.xml-only; cannot be type-hinted or plugin targets.
- Request the most generic type you need. Constructors do assignment + validation only.
- `ObjectManager` direct use is prohibited. Exceptions: magic methods (`__wakeup` etc.), constructor BC preservation (new optional arg at END with `null` default + ObjectManager fallback), integration-test fixtures/global scope, and classes whose job is object creation (factories/proxies). Never in unit tests.

## Service contracts & @api

- Service interfaces in `Api/`, data interfaces in `Api/Data/`. Repositories: `save/get/getList(SearchCriteriaInterface)/delete/deleteById` + per-entity `SearchResultsInterface`.
- Api types limited to: scalars, data interfaces, one-dimensional indexed arrays, nullables, void. Data interfaces contain NO business logic. Service interfaces SHOULD declare a single public method.
- Cross-module calls go through service contracts, never another module's concrete Models/ResourceModels/Collections.
- `@api` = BC promise across MINOR/PATCH. Depend only on other modules' `@api` surface; mark your own stable contracts `@api`. SemVer: MAJOR = breaking, MINOR = additions/changed customization points, PATCH = compatible fixes.

## Database schema & patches

- **Declarative schema first**: `etc/db_schema.xml` (runs BEFORE patches). Data patches (`Setup/Patch/Data/`, `DataPatchInterface`) for data; schema patches only for what declarative can't express. Don't write new InstallSchema/UpgradeSchema (pre-2.3 legacy; still executed for old modules).
- Regenerate whitelist on every schema change: `bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module`. Dropping a column requires it whitelisted; when dropping a TABLE keep it in the whitelist or it won't drop. Renames preserve data via `onCreate="migrateDataFrom(col)"`.
- Declarative schema WILL drop elements you remove from db_schema.xml — removals are destructive by design.
- Patches run once (recorded in `patch_list`), ordered via `getDependencies()`; `getAliases()` when renaming a patch class. Revert (`PatchRevertableInterface`) is data-patch-only and runs ONLY on full `module:uninstall` — you can't revert one patch. Write patches idempotent.

## Module wiring & packaging

- `module.xml <sequence>` orders config/view-files/setup loading only — NOT PHP class loading. If you depend on another module, add it to BOTH composer.json `require` and `<sequence>`. Re-run `module:enable` after sequence changes (regenerates config.php order). Circular sequences abort install.
- Composer packaging: `"type": "magento2-module"`, `registration.php` in `autoload.files`, PSR-4 mapping. `registration.php` uses `ComponentRegistrar::register(MODULE, 'Vendor_Module', __DIR__)`.

## PHP-level rules (Adobe Technical Guidelines)

- New PHP files MUST start `declare(strict_types=1);`, declare explicit return types; scalar type hints SHOULD be used. (Don't mass-edit legacy files to comply.)
- Composition over inheritance; non-public members SHOULD be private; no setters outside DTOs.
- Exceptions: user-facing exceptions MUST extend `LocalizedException`; never throw bare `\Exception`; don't manage business flow with exceptions. In repositories/web APIs use framework types (`NoSuchEntityException`, `CouldNotSaveException`, ...) so REST/GraphQL map them properly. Remember: a generic `CouldNotSaveException` usually MASKS the real error — check `var/log/exception.log`.
- PHPCS: `vendor/bin/phpcs --standard=Magento2` (magento/magento-coding-standard, v40 as of 2026-03; auto-fix with `phpcbf`). Raw SQL is flagged (`Magento2.SQL.RawQuery`). Core's own PHPStan bar is level 1 — use level 5+ with `bitexpert/phpstan-magento` for module code.
