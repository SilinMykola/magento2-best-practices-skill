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
- `ObjectManager` direct use is prohibited. Exceptions: magic methods (`__wakeup` etc.), constructor BC preservation (new optional arg at END, `?Foo $foo = null`, + ObjectManager fallback), integration-test fixtures/global scope, and classes whose job is object creation (factories/proxies). Never in unit tests.

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
- Never catch and log inside a data patch's `apply()`: each data patch runs in a transaction and is written to `patch_list` only after `apply()` returns, so a swallowed exception records a half-applied patch as done (`lib/internal/Magento/Framework/Setup/Patch/PatchApplier.php`, 2.4.9). `NonTransactionableInterface` only for work that cannot run in a transaction. EAV attributes in patches: backend-components.md.

## Module wiring & packaging

- `module.xml <sequence>` orders config/view-files/setup loading only — NOT PHP class loading. If you depend on another module, add it to BOTH composer.json `require` and `<sequence>`. Re-run `module:enable` after sequence changes (regenerates config.php order). Circular sequences abort install.
- Composer packaging: `"type": "magento2-module"`, `registration.php` in `autoload.files`, PSR-4 mapping. `registration.php` uses `ComponentRegistrar::register(MODULE, 'Vendor_Module', __DIR__)`.

## PHP-level rules (Adobe Technical Guidelines)

- New PHP files MUST start `declare(strict_types=1);`, declare explicit return types; scalar type hints SHOULD be used. (Don't mass-edit legacy files to comply.)
- Composition over inheritance; non-public members SHOULD be private; no setters outside DTOs.
- Exceptions: user-facing exceptions MUST extend `LocalizedException`; never throw bare `\Exception`; don't manage business flow with exceptions. In repositories/web APIs use framework types (`NoSuchEntityException`, `CouldNotSaveException`, ...) so REST/GraphQL map them properly. Remember: a generic `CouldNotSaveException` usually MASKS the real error — check `var/log/exception.log`.
- PHPCS: `vendor/bin/phpcs --standard=Magento2` (magento/magento-coding-standard, v41 as of 2026-08; auto-fix with `phpcbf`). Raw SQL is flagged (`Magento2.SQL.RawQuery`). Core's own PHPStan bar is level 1 — use level 5+ with `bitexpert/phpstan-magento` for module code.

## Stateless services (2.4.7+)

- Service classes (behaviour, not data) SHOULD NOT hold mutable state; plugins MUST be stateless; getters SHOULD NOT change state; temporal coupling MUST be avoided (Technical Guidelines 2.9, 2.12, 2.14, 4.4). Pass per-request data (customer, quote, store, request) as method arguments, never keep it in service properties. Only data objects and entities hold state.
- A property cache kept for speed implements `Magento\Framework\ObjectManager\ResetAfterRequestInterface` (2.4.7+): `_resetState(): void` puts every mutable property back to the value the constructor gave it (`[]` stays `[]`, not `null`). Core example: `Magento\Customer\Model\AddressRegistry`. 2.4.7 resets only implementors of the interface (plus core's own reset list); from 2.4.8 any object with a `_resetState()` method is reset, so implement the interface explicitly when code must run on 2.4.7.
- Never cache request data in static properties or read superglobals (`$_GET`, `$_POST`, `$_SESSION`): they are not reset between requests in a long-running process. Use the injected request and session objects.
- The GraphQL Application Server that reuses objects across requests is Adobe Commerce only, but the state tests that enforce these rules ship in Open Source core (see quality-testing.md), and third-party modules may run under the app server.

## PHP 8.4 / 8.5 (2.4.9: 8.4/8.5, 8.3 upgrade-only — release-matrix.md)

- Every PHP deprecation is a crash: `app/bootstrap.php` sets `error_reporting(E_ALL)` and `Magento\Framework\App\ErrorHandler` throws "Deprecated Functionality: ..." in every mode.
- Default-null parameters are explicitly nullable: `?Foo $x = null`, never `Foo $x = null` (PHP 8.4 deprecation; `php -l` on PHP 8.4+ reports it). This includes the BC constructor pattern above. Core removed all such signatures in 2.4.8.
- Never use `null` as an array key, in reads, writes, `isset()`, `??` or `array_key_exists()` (PHP 8.5). Typical source: the ID of an unsaved entity or a null scope code; guard or normalise it (`$scopeCode ?? ''`).
- Only canonical casts `(int)`, `(bool)`, `(float)`, `(string)` (PHP 8.5). Drop `curl_close()` and Reflection `setAccessible()`; both are no-ops.
- Declare every property (dynamic properties deprecated in PHP 8.2). `Magento\Framework\DataObject` carries `#[\AllowDynamicProperties]`, so models and blocks are exempt, but services, plugins and resolvers are not; declare the property instead of adding the attribute.

Sources: [Technical Guidelines](https://developer.adobe.com/commerce/php/coding-standards/technical-guidelines), [Application Server](https://developer.adobe.com/commerce/php/development/components/app-server), [PHP 8.4 deprecations](https://www.php.net/manual/en/migration84.deprecated.php), [PHP 8.5 deprecations](https://www.php.net/manual/en/migration85.deprecated.php), [PHP 8.2 deprecations](https://www.php.net/manual/en/migration82.deprecated.php) (opened November 2026). Core code, magento/magento2: `lib/internal/Magento/Framework/ObjectManager/ResetAfterRequestInterface.php` (since 2.4.7), `lib/internal/Magento/Framework/ObjectManager/Resetter/Resetter.php` (2.4.7 vs 2.4.8), `lib/internal/Magento/Framework/App/ErrorHandler.php`, `lib/internal/Magento/Framework/DataObject.php` (2.4.9); commits 02eae874acf (implicit nullable, 2.4.8) and 491c19acfa0 (casts, 2.4.9) on 2.4-develop.
