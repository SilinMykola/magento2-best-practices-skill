# Quality & testing (Magento 2.4.x)

## Test types (official)

| Suite | Location | Purpose |
|---|---|---|
| Unit | `dev/tests/unit` + module `Test/Unit` | isolated algorithms, NO app stack |
| Integration | `dev/tests/integration` + module `Test/Integration` | components together: real DI, real DB (transaction-rollback isolation) |
| API-functional | `dev/tests/api-functional` | REST/SOAP/GraphQL from the client side (`phpunit_rest/soap/graphql.xml` configs; `WebapiAbstract`/`GraphQlAbstract`) |
| Static | `dev/tests/static` | phpcs/phpmd/PHPStan/copy-paste wrappers |
| MFTF | module `Test/Mftf`, run from `dev/tests/acceptance` | browser functional tests (XML) |

- Magento's definition of done requires automated tests for code changes.
- **Unit vs integration split**: pure calculation/branching → unit; anything touching DI wiring, DB, config, events, repositories, GraphQL/REST endpoints → integration/api-functional. Over-mocked unit tests of glue code are an anti-pattern (DI errors only surface at integration level).
- For a headless build (PWA/Next.js/etc. on GraphQL), **api-functional GraphQL coverage of custom resolvers is the highest-value layer**.
- MFTF is NOT deprecated (2.4.9 pins ^6.0) but is costliest to maintain — prioritize unit/integration/api-functional for module logic.

## Choosing the layer

Pick the lowest layer where the behaviour is visible; a bug fix gets a test that fails without the fix.

| The behaviour lives in | Test with | Location (module / Magento core checkout) |
|---|---|---|
| Pure logic of one class (calculation, condition, mapping) | unit | `<Module>/Test/Unit/<same path as class>Test.php` |
| SQL, collections, indexers, repositories, DI wiring, plugins, observers, config | integration | `<Module>/Test/Integration/...` / `dev/tests/integration/testsuite/Magento/<Module>/...` |
| Admin or storefront controller behaviour | integration (`AbstractBackendController` / `AbstractController`) | same |
| GraphQL resolver output or errors | api-functional GraphQL | `dev/tests/api-functional/testsuite/...` |
| REST endpoint | api-functional REST | same |
| UI component or RequireJS module | Jasmine | `dev/tests/js/jasmine/tests/app/code/<Vendor>/<Module>/...` |
| Only visible in the browser across pages | MFTF (slowest to write and run) | `<Module>/Test/Mftf/Test/` |

A fix often gets two tests: a unit test for the branch in logic and an integration test for the visible effect.
Test class names end with `Test` and match the file name.

## PHPUnit version cliffs

- 2.4.6 → PHPUnit **9.5**; 2.4.8 → **10.5**; 2.4.9 → **12.0**.
- Metadata style is VERSION-DEPENDENT: attributes (`#[Test]`, `#[DataProvider]`) exist only in PHPUnit **10+** — on 2.4.6 (PHPUnit 9.5) they are silently inert (a `#[DataProvider]` test fails "too few arguments"; a non-`test*`-named `#[Test]` method never runs). Rule: tests that must run on PHPUnit 9.x use PHPUnit-9 doc-comment annotations and `test*` method names (optionally duplicated with attributes for forward-compat); attributes-only becomes the rule after the PHPUnit ≥10 bump. Doc-comment annotations are deprecated in 11, REMOVED in 12.
- `withConsecutive()` removed since PHPUnit 10 — use per-invocation expectations or `willReturnCallback`.
- **ObjectManagerHelper (`TestFramework\Unit\Helper\ObjectManager`) is @deprecated** — instantiate the class under test with `new` + explicit `$this->createMock()` dependencies. Never the real ObjectManager in unit tests.

## PHPUnit 12 (2.4.9 and 2.4-develop)

Old tests in core still show pre-10 styles; do not copy them into new tests.
- PHPUnit metadata only as attributes (`#[DataProvider('provider')]`, `#[Test]`, `#[CoversClass]`); providers are `public static`.
- `withConsecutive()` is gone: `willReturnCallback()` with a call counter, or separate expectations.
- Build the class under test with `new` and `$this->createMock()`.
- When touching an old test, update only what the change needs.

## Fixtures (integration + api-functional)

- Prefer attribute-based parameterized fixtures (`#[DataFixture]` since 2.4.5; its `scope` and `count` arguments since 2.4.6, per `dev/tests/integration/framework/Magento/TestFramework/Fixture/DataFixture.php` at those tags):
  `#[DataFixture(ProductFixture::class, ['sku' => 'x-%uniqid%'], as: 'p1', scope: 'store2', count: 3)]`
- Reference other fixtures: `['cart_id' => '$cart.id$']`; retrieve via `DataFixtureStorageManager::getStorage()->get('alias')`.
- Fixture classes: single responsibility, `%uniqid%` for unique SKUs/emails, no inter-fixture coupling (compose via service APIs), implement revert for anything DB-transactions don't roll back.
- Companion attributes: `#[Config]`, `#[AppArea]`, `#[AppIsolation]`, `#[DbIsolation]`, `#[Cache]`, `#[DataFixtureBeforeTransaction]`.
- Indexer behaviour is tested in "Update by Schedule" mode, the mode production runs: reviewers reject fixes that
  only work on save. `#[DataFixtureBeforeTransaction(ScheduleMode::class, ['indexer' => '<id>'])]`
  (`Magento\Indexer\Test\Fixture\ScheduleMode`) for every involved indexer, `#[AppIsolation(true)]` (cached mview
  objects otherwise carry a stale `mview_state.version_id` into the next test and the changelog rows are skipped),
  `#[DbIsolation(false)]`, then `Magento\Indexer\Model\Processor::updateMview()` twice (one view queues work for
  another) instead of `reindexAll()`. Fixes belong where the partial reindex happens (mview action or its plugins).
- A legacy fixture file under `_files/` must not `require` another one: `Resolver::getInstance()->requireDataFixture()`.
- Integration setup: dedicated EMPTY throwaway DB in `dev/tests/integration/etc/install-config-mysql.php`; `TESTS_CLEANUP=enabled` reinstalls per run (clean, slow) — if disabled, delete `dev/tests/integration/tmp/sandbox-*` after module changes. Third-party module suites need a testsuite entry in phpunit.xml.

## Application state tests (core, 2.4.7+)

Core integration tests catch mutable state in services (rules in module-development.md, "Stateless services"):
- `dev/tests/integration/testsuite/Magento/GraphQl/App/GraphQlStateTest.php` (with `GraphQlCheckoutMutationsStateTest.php` and `GraphQlCustomerMutationsTest.php`) runs each GraphQL operation twice and compares shared ObjectManager instances: request 2 against request 1 ("%d objects changed state during request") and against the constructed state ("%d objects changed state since constructed"); the messages come from `GraphQl/App/State/GraphQlStateDiff.php`.
- `dev/tests/integration/testsuite/Magento/Framework/ObjectManager/ResetAfterRequestTest.php` builds every `ResetAfterRequestInterface` implementor, calls `_resetState()` and requires the constructed state back; a reset to `null` where the constructor set `[]` fails it.
- Fix a failure in this order: remove the mutable state (arguments, or a stateful helper from a Factory); else implement `_resetState()`; only for framework or config caches that are safe to keep, add an entry with a reason comment to `dev/tests/integration/framework/Magento/TestFramework/ApplicationStateComparator/_files/state-skip-list.php` (whole classes) or `state-filter-list.php` (single properties). The path given on Adobe's app-server page is outdated; use this one.
- "Typed property ... must not be accessed before initialization" in these tests means temporal coupling: refactor.

Sources: those files on 2.4-develop; [Application Server, "How to fix test failures"](https://developer.adobe.com/commerce/php/development/components/app-server) (opened November 2026).

## Static quality gates

- PHPCS: `vendor/bin/phpcs --standard=Magento2 --extensions=php,phtml` (magento-coding-standard [v41](https://github.com/magento/magento-coding-standard/releases/tag/v41), 2026-08-18; only ESLint was updated since v40). Marketplace blocks only severity ≥ 10 errors — reproduce with `--error-severity=10 --ignore-annotations`. CI: severity-10 = hard fail, lower = advisory. `phpcbf` auto-fixes.
- PHPStan: core's static suite runs level 1 — a floor, not a target. Modules: level 5+ with `bitexpert/phpstan-magento` (understands factories/proxies/generated code).
- php-cs-fixer is complementary (core ships `.php-cs-fixer.dist.php`: @PSR2 + ordered_imports, no_unused_imports, short arrays) — formatting, while phpcs checks the standard.
- SemVer enforcement: `magento/magento-semver` (`php bin/svc compare <old> <new>`) reports the required PATCH/MINOR/MAJOR bump + each breaking change; CI should fail if the report exceeds the composer.json bump. Marketplace runs exactly this.
- @api BC policy: PATCH releases allow only patch-level changes; @api surface changes (signatures, arg types/defaults, thrown exceptions) are forbidden without explicit decision. New constructor deps: append at END as `?Foo $foo = null` + ObjectManager fallback.
- Upgrade Compatibility Tool is **Adobe-Commerce-only**. Open Source substitutes: magento-semver + PHPStan/PHPCompatibility + `ampersand/magento2-upgrade-patch-helper`.

## CI

- Ready-made GitHub Actions for Magento modules: `extdn/github-actions-m2` (PHPCS latest Magento standard, PHPStan, PHPMD, PHPCPD, integration tests with MySQL/OpenSearch services against chosen Magento versions).
- Recommended matrix: lint+static on every PR; integration tests against BOTH the running version and the planned upgrade target (Magento + PHP) to surface migration breaks early.
- Run suites directly: `vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist path/to/Test/Unit` (CI-friendly); `bin/magento dev:tests:run` is a convenience wrapper.
