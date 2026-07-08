# Security (Magento 2.4.x)

## Patching

- Apply security-only patches (`2.4.x-pN`, tied to APSB bulletins at helpx.adobe.com/security/products/magento.html) **within days** of release — attackers reverse-engineer patches in days-to-weeks (CosmicSting precedent). Latest wave 2026-05-12 (APSB26-49): 2.4.8-p5 / 2.4.7-p10 / 2.4.6-p15, fixes incl. stored-XSS→code-exec (CVE-2026-34686) and path-traversal write (CVE-2026-34653).
- Individual quality fixes: **Quality Patches Tool** (`composer require magento/quality-patches`; `vendor/bin/magento-patches status|apply ACSD-XXXXX|revert`) — never hand-edit vendor/.
- Run Adobe's free **Commerce Security Scan** (account.magento.com — works for Open Source) as continuous external monitoring.

## Admin hardening

- Custom admin front name (project: `admin_15cfht` ✓), Add Secret Key to URLs ON, case-sensitive passwords, session lifetime + ~90-day password lifetime, login lockout, IP-allowlist/VPN the admin, WAF in front, audit docroot for exposed .git/dumps/logs.
- Least-privilege ACL roles. **Since 2.4.8 new ACL resources from custom modules are NOT auto-granted to existing roles** — grant explicitly.
- 2FA (`Magento_TwoFactorAuth`) is mandatory since 2.4.0 for admin UI + admin web-API; never disable it in production. 2.4.9: admins configure only ONE enabled provider (was: all). Locked out → `bin/magento security:tfa:reset <user> <provider>`.
- reCAPTCHA on admin login + storefront forms. 2.4.9 also enforces CAPTCHA on REST/GraphQL account creation (pre-2.4.9 the APIs bypassed form CAPTCHA — bots registered via API).

## CSP & SRI (payment-page compliance, PCI DSS 4.0)

- Since 2.4.7: **checkout/payment pages enforce CSP** (no `unsafe-inline` in script-src); other pages report-only. 2.4.6 is report-only everywhere — code written today must already be CSP-safe or it breaks on upgrade.
- Whitelist sources per module in `etc/csp_whitelist.xml` (`<policy id="script-src"><value id="x" type="host">https://cdn...</value></policy>`).
- Inline scripts: `SecureHtmlRenderer` (`$secureRenderer->renderTag()/renderEventListenerAsTag()`), or 2.4.7+ `CspNonceProvider::generateNonce()`, or `type="hash"` SHA-256 whitelist entries. NEVER re-add `unsafe-inline` to payment pages.
- SRI: integrity hashes for local JS since 2.4.7 (backported 2.4.6-p6+), enforced on payment pages; 2.4.9 covers minified/bundled JS, per-area/theme/locale `sri-hashes.json`.
- Mode config: `csp/mode/{storefront,admin}/report_only` (+ `report_uri`). Upgrade playbook: run report-only, collect violations, fix, then enforce.

## Code-level rules

- **SQLi**: prepared statements/bound params ONLY (`$connection->select()->where('field = ?', $v)`, adapter `insert()/update()/delete()`); prefer repositories/collections; never interpolate input into SQL strings.
- **XSS**: escape ALL template output with the injected `$escaper` (`escapeHtml/escapeHtmlAttr/escapeUrl/escapeJs/escapeCss` — context-correct). `AbstractBlock::escape*` are deprecated. Only `*Html`-suffixed getters may output raw; anything else intentionally raw needs `/* @noEscape */` with justification.
- **CSRF**: state changes are POST with form_key; mark controllers `HttpPostActionInterface`; `CsrfAwareActionInterface` only for genuine exceptions (webhooks). Never disable form-key validation globally.
- **Prohibited functions**: `eval`, `serialize`/`unserialize` (object injection — use `SerializerInterface`/Json), `md5`/`sha1` for security, `srand`/`mt_srand`, `passthru`/`system`/`shell_exec`; never pass user input to include/require/fopen/preg_replace. PHPCS Magento2 ruleset enforces — keep it in CI.
- **Crypto**: `EncryptorInterface` for secrets (`encrypt/decrypt`) and passwords (`getHash/isValidHash`); system-config secrets get `backend_model` `Config\Backend\Encrypted`. Never roll your own.
- **Secrets & git**: `env.php` (crypt key! DB creds) and `auth.json` NEVER committed; `config.php` IS committed. Per-env values via `config:sensitive:set` / `CONFIG__DEFAULT__*` env vars; lock critical values with `config:set --lock-env`.

## Attack classes to design against

- **CosmicSting (CVE-2024-34102, CVSS 9.8)**: unauthenticated XXE via nested deserialization in REST/GraphQL → read env.php → stolen crypt key → forged admin JWTs → CMS-block skimmer injection (RCE when chained with CVE-2024-2961). Lessons: any API-reachable deserialization of nested structures is attack surface; XML parsers must not resolve external entities; env.php readability = total compromise.
- **Key rotation is incomplete by default**: rotating the crypt key did NOT invalidate old keys (Sansec: stores stayed compromised post-patch; Adobe AC-12485 hotfix added invalidation). Post-incident: rotate AND invalidate old keys, re-encrypt (e.g. gene/module-encryption-key-manager), rotate admin passwords + ALL integration/API tokens, and **diff all CMS blocks/pages for injected scripts**.
- Carding/DoS: keep input limits + pagination caps on (see graphql-api.md); 2.4.7+ payment-endpoint backpressure; on 2.4.6 rely on WAF rate limits.
