# Magento 2.4.x release matrix & upgrade path (verified 2026-07-06)

> Dated snapshot. Anything time-sensitive here (latest -p release, support
> end dates, requirements) must be re-checked against Adobe's release notes
> and system-requirements pages before you state it as current.

## Current releases

- **2.4.9** — latest line. GA **2026-05-12** (release-notes page says May 15; mid-May 2026). ~580 fixes. Regular support to **May 2029**. No 2.4.9-p1 yet (next patch wave ~Aug 2026).
- Patch levels shipped 2026-05-12: **2.4.8-p5, 2.4.7-p10, 2.4.6-p15** (bulletin APSB26-49, fixes up to CVSS 8.7).
- **2.4.6 regular support ENDS 2026-08-11.** If you run 2.4.6, upgrade planning is urgent, not optional.
- New cadence since Jan 2026: monthly security patches, annual feature release each May, aggregated -p releases May/November.

## 2.4.9 platform requirements (Adobe system-requirements matrix)

| Component | 2.4.9 | 2.4.6 |
|---|---|---|
| PHP | **8.4 / 8.5** (8.3 upgrade-only, 8.2 DROPPED) — composer.json: `~8.3.0 \|\| ~8.4.0 \|\| ~8.5.0` | 8.1 / 8.2 |
| MySQL | **8.4 LTS** (8.0 not listed) | 8.0 |
| MariaDB | 11.8 / 12.x (12.3 tested) | 10.6-era |
| Search | **OpenSearch 3.x only — Elasticsearch REMOVED** (deprecated 2.4.8; ES 7.17 itself EOS 2026-01-15) | ES 7.17 (EOS!) / OpenSearch 2 |
| Cache | **Valkey 9** (Valkey 8 in 2.4.8; Redis-compatible drop-in, Redis no longer listed) | Redis 7 |
| RabbitMQ | 4.2 (must migrate classic mirrored → quorum queues first; ActiveMQ Artemis flagged as long-term direction) | 3.x |
| Composer | 2.10 (practically: latest 2.x) | 2.2+ |
| Varnish / nginx | 8 / 1.30 | 7.x / 1.24 |
| PHPUnit / MFTF | 12 / ^6.0 | 9.5 / ^4.2 |

## Upgrade path 2.4.6 → 2.4.9 (staging box runbook skeleton)

The PHP/MySQL bumps must INTERLEAVE with the Magento hops — 2.4.6 itself only
supports PHP 8.1/8.2 and MySQL 8.0, so bumping the platform first would take
the running store down.

1. **Genuinely in-place on 2.4.6**: stay on latest 2.4.6-pN, and migrate
   search ES 7.17 → OpenSearch 2.x. ⚠ If the store uses a third-party search
   module (e.g. Smile ElasticSuite, engine setting `elasticsuite`), do NOT flip
   the engine to `opensearch` — upgrade that module to an OpenSearch-compatible
   release and repoint its client config. Flipping the engine is only for
   vanilla installs; on such stores it silently kills search/layered nav.
2. Code readiness while still on 2.4.6: make custom code PHP 8.3+ clean (no
   dynamic properties etc.), CSP-safe, PHPUnit-forward-compatible.
3. Hop to 2.4.7/2.4.8 **together with** PHP 8.3 (supported by both sides of
   that hop). RabbitMQ (if adopted): 3.x → 4.2 with quorum-queue migration
   before/with the 2.4.8 hop.
4. Hop to 2.4.9 **together with** PHP 8.4/8.5 and MySQL 8.0 → 8.4 LTS (2.4.8
   supports MySQL 8.4, so the DB can also move during step 3's window). Note
   utf8mb4 is the default collation since 2.4.8 — check custom schema/raw SQL
   for utf8mb3 assumptions.
5. After 2.4.9 is stable: OpenSearch 2.x → 3.x.
6. OSS toolchain for impact analysis (UCT is Adobe-Commerce-only): `magento/magento-semver` vs new core, PHPStan + PHPCompatibility for the PHP jump, `ampersand/magento2-upgrade-patch-helper` for customizations vs core diffs.

## Behavior deltas that WILL bite on upgrade (2.4.6 → 2.4.7+/2.4.9)

- **CSP enforce mode on payment/checkout pages since 2.4.7** (report-only everywhere on 2.4.6). Unwhitelisted inline JS on checkout breaks. Write CSP-safe code NOW (see security.md).
- **SRI hashes** for local JS since 2.4.7 (backported 2.4.6-p6+), enforced on payment pages; 2.4.9 covers minified/bundled JS with per-area/theme/locale `sri-hashes.json`.
- **PHPUnit 9.5 → 12**: attributes only (`#[Test]`, `#[DataProvider]`) — doc-comment annotations REMOVED in PHPUnit 12; `withConsecutive()` removed since PHPUnit 10.
- **TinyMCE → HugeRTE** in 2.4.9 (TinyMCE 5 removed in 2.4.8): admin WYSIWYG customizations need porting.
- **ACL change 2.4.8**: new ACL resources from custom modules are NO longer auto-granted to existing admin roles.
- GraphQL: 2.4.7 added `createGuestCart`, `applyCouponsToCart`, `estimateTotals`, `cancelOrder`; deprecated `createEmptyCart`, `clearCustomerCart`, `customAttributeMetadata`. 2.4.9: `clearCart` now in Open Source, `exchangeExternalCustomerToken`, `clearWishlist`, `OrderTotal.grand_total_excl_tax`, **CAPTCHA enforced on REST/GraphQL customer creation** (headless frontends must send tokens or exempt in config).
- 2.4.9 internals: Symfony 7.4 LTS, JWT lib major bump, `carlos-mg89/oauth` replaced with native PHP OAuth.
- Features that exist on 2.4.9 but NOT on 2.4.6: GraphQL Resolver Cache (2.4.7+), payment-endpoint rate limiting/backpressure (2.4.7+).

When writing code for a store on an older 2.4.x, do not use APIs introduced after its version; when writing anything new, prefer patterns that survive the 2.4.9 jump (CSP-safe JS, OpenSearch-agnostic search code, non-deprecated GraphQL mutations). Tests are the exception: they must match the RUNNING PHPUnit (9.5 → doc-comment annotations; 10+ → attributes; see quality-testing.md).
