# Magento 2.4.x release matrix & upgrade path (verified 2026-07-06; releases, lifecycle and requirements re-checked 2026-11-07)

> Dated snapshot. Anything time-sensitive here (latest -p release, support
> end dates, requirements) must be re-checked against Adobe's release notes
> and system-requirements pages before you state it as current.

## Current releases

- **2.4.9** — latest line. GA **2026-05-12** (release-notes page says May 15; mid-May 2026). ~580 fixes. Regular support to **2029-05-31**. No 2.4.9-p1 as of 2026-11-07.
- Latest `-p` levels, shipped 2026-05-12: **2.4.8-p5, 2.4.7-p10, 2.4.6-p15** (bulletin APSB26-49, fixes up to CVSS 8.7). Since July 2026 security fixes ship as monthly **isolated patches** (`2.4.9-2026-jul`, `-aug`, `-sep`, `-oct`, ...) applied on top of the latest `-p`; bulletins APSB26-73 (Jul), APSB26-92 (Aug), APSB26-138 (Sep), APSB26-146 (Sep, out-of-band hotfix, exploited in the wild), APSB26-153 (Oct). How to apply: [security.md](security.md#patching).
- Seen as of 2026-11-07, needs action: [APSB26-146](https://www.adobe.com/trust/security/products/commerce/apsb26-146.html) (2026-09-07, out of band, priority 1, CVSS 10.0 code execution without auth, CVE-2026-75650, **exploited in the wild**) is fixed by a separate hotfix that the October isolated patch does not include; [APSB26-153](https://www.adobe.com/trust/security/products/commerce/apsb26-153.html) (2026-10-13) fixes another CVSS 10.0 code execution (CVE-2026-103849) and lists no fixed Open Source 2.4.6 version. A store without both is exposed; Open Source 2.4.6 has no fix and must upgrade.
- Lifecycle ([policy](https://experienceleague.adobe.com/en/docs/commerce-operations/release/planning/lifecycle-policy), updated 2026-09-18): 2.4.6 regular support ENDED 2026-08-11 (extended support to 2027-08-31 is described for Adobe Commerce customers; APSB26-153 lists no fixed Open Source 2.4.6 version); 2.4.7 to 2027-05-31; 2.4.8 to 2028-05-31; 2.4.9 to 2029-05-31. On Open Source 2.4.6, upgrading is now a security matter.
- Cadence ([schedule](https://experienceleague.adobe.com/en/docs/commerce-operations/release/planning/schedule), updated 2026-10-08): monthly security patches, one feature release and one full patch each May.
- **2.4.10-alpha1** tagged 2026-10-13, no GA date announced. Its `composer.json`: PHP `~8.3.0||~8.4.0||~8.5.0`, PHPUnit `^12.0`, MFTF `^7.0`; requirements page lists OpenSearch 3.7+, Varnish 9, RabbitMQ 4.3 ([2.4.10 release notes](https://experienceleague.adobe.com/en/docs/commerce-operations/release/notes/adobe-commerce/2-4-10)).

## 2.4.9 platform requirements (Adobe system-requirements matrix)

| Component | 2.4.9 | 2.4.6 |
|---|---|---|
| PHP | **8.4 / 8.5** (8.3 upgrade-only, 8.2 DROPPED) — composer.json: `~8.3.0 \|\| ~8.4.0 \|\| ~8.5.0` | 8.1 / 8.2 |
| MySQL | **8.4 LTS** (8.0 not listed) | 8.0 |
| MariaDB | 11.8 / 12.x (12.3 tested) | 10.6-era |
| Search | **OpenSearch 3.x only — Elasticsearch REMOVED** (deprecated 2.4.8; ES 7.17 itself EOS 2026-01-15) | ES 7.17 (EOS!) / OpenSearch 2 |
| Cache | **Valkey 9** (Valkey 8 in 2.4.8; Redis-compatible drop-in, Redis no longer listed) | Redis 7 |
| RabbitMQ | 4.2 at GA; requirements page shows 4.3 and ActiveMQ Artemis 2 as of 2026-11 (must migrate classic mirrored → quorum queues first; ActiveMQ Artemis flagged as long-term direction) | 3.x |
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
   supports MySQL 8.4, so the DB can also move once step 3 has reached 2.4.8;
   2.4.4–2.4.7 support no MySQL newer than 8.0, and MySQL 8.0 itself reached end
   of support 2026-04-30 per the system-requirements page, seen November 2026). Note
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
