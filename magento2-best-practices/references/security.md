# Security (Magento 2.4.x)

## Patching

- Apply security patches **within days** of release — attackers reverse-engineer patches in days-to-weeks (CosmicSting precedent). Bulletins: `https://www.adobe.com/trust/security/products/commerce/apsb26-NN.html`. Since July 2026 fixes ship as monthly **isolated patches** (`2.4.x-2026-<mon>`, ZIP files, not Composer packages) on top of the latest `-pN` of the line. Earlier monthly patches must already be applied in release order; within a month apply CE, then EE, then B2B files. Check with `php vendor/bin/patch-status` (ships in the July 2026 patch). Open Source can download them only for 2.4.7 and later ([KA-49323](https://experienceleague.adobe.com/en/docs/experience-cloud-kcs/kbarticles/ka-49323)). Re-check the current state in [release-matrix.md](release-matrix.md) before stating it.
- Bulletins currently open (out-of-band hotfix, lines without a fix): [release-matrix.md](release-matrix.md#current-releases).
- Individual quality fixes: **Quality Patches Tool** (`composer require magento/quality-patches`; `vendor/bin/magento-patches status|apply ACSD-XXXXX|revert`) — never hand-edit vendor/.
- Run Adobe's free **Commerce Security Scan** (account.magento.com — works for Open Source) as continuous external monitoring.

## Admin hardening

- Custom admin front name, Add Secret Key to URLs ON, case-sensitive passwords, session lifetime + ~90-day password lifetime, login lockout, IP-allowlist/VPN the admin, WAF in front, audit docroot for exposed .git/dumps/logs.
- Least-privilege ACL roles. **Since 2.4.8 new ACL resources from custom modules are NOT auto-granted to existing roles** — grant explicitly. Code-level ACL: see [Admin ACL in code](#admin-acl-in-code).
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

## Admin ACL in code

- Every adminhtml controller overrides `ADMIN_RESOURCE` with its own `etc/acl.xml` resource. The inherited default is `Magento_Backend::admin`, which every saved role receives ("Give basic admin permissions to any admin"), so a controller without its own constant is open to every admin user. A `menu.xml` item's `resource` does not protect the controller.
- Every `system.xml` `<section>` declares `<resource>`: without it `Section::isAllowed()` returns false and the section is hidden from all admins.
- Every admin UI listing and form declares `<aclResource>` inside its `<dataSource>`. `mui/index/render` renders any UI component by `namespace` and its own `_isAllowed()` returns true; `aclResource` is the only check on that route. Adobe's admin-grid tutorial omits it — do not copy it blindly. Same on 2.4.6.

Sources: core code, magento/magento2 2.4.9: `app/code/Magento/Backend/App/AbstractAction.php`, `app/code/Magento/Authorization/Model/ResourceModel/Rules.php` (`saveRel`), `app/code/Magento/Config/Model/Config/Structure/Element/Section.php`, `app/code/Magento/Ui/Controller/Adminhtml/AbstractAction.php`, `app/code/Magento/Ui/Controller/Adminhtml/Index/Render.php` (`validateAclResource`), example `app/code/Magento/Cms/view/adminhtml/ui_component/cms_page_listing.xml`; [ACL tutorial](https://developer.adobe.com/commerce/php/tutorials/backend/create-access-control-list-rule), [admin grid tutorial](https://developer.adobe.com/commerce/php/development/components/add-admin-grid) (opened November 2026).

## Files, paths, mass assignment, SSRF

- Uploads: `Magento\MediaStorage\Model\File\Uploader` with `setAllowedExtensions([...])`; its
  `checkAllowedExtension()` also blocks `general/file/protected_extensions` (php, phtml, phar, htaccess, svg, html,
  xml, ...). The framework `Magento\Framework\File\Uploader` allows every extension when no allow-list is set. Add
  `checkMimeType([...])` (reads the content, not the client header) and `setAllowRenameFiles(true)`.
- Image validation (`addValidateCallback(..., $imageAdapter, 'validateUploadFile')`) does not prove a file is safe: an
  image/PHP polyglot passes it, so the extension check is what stops it (seen exploited in 2026:
  [Sansec, PolyShell](https://sansec.io/research/magento-polyshell)).
- Non-public files go under `DirectoryList::VAR_DIR`, never `pub/`, with random names; record ownership and check it
  before reading or deleting.
- File I/O through `Filesystem::getDirectoryRead/Write(DirectoryList::X)`: every method runs `PathValidator`, which
  throws `ValidatorException` for a path resolving outside the directory or starting with `-` (it is a prefix check:
  a sibling directory sharing the name prefix is not caught). `Filesystem\Driver\File`, `Io\File` and raw
  `file_put_contents()` validate nothing.
- Never `setData()`/`addData()` request params onto a model: hydrate an operation-specific DTO
  (`DataObjectHelper::populateWithArray()`), take IDs from the user context (`force="true"` in webapi.xml), and expose
  privileged fields only on endpoints with their own ACL. Core examples: `CustomerExtractor` in
  `Customer/Controller/Account/CreatePost.php`; `AccountManagementApi::validateCustomerRequest()` rejects `group_id`
  without its ACL resource.
- Do not fetch user-supplied URLs. If unavoidable: http/https only, an allow-list of hosts, never echo the response.
  Core's guard for downloadable links and import image URLs is `Magento\Downloadable\Model\Url\DomainValidator`
  (`downloadable_domains` in env.php, `bin/magento downloadable:domains:add`).

Sources: core code, magento/magento2 2.4.9: `lib/internal/Magento/Framework/File/Uploader.php`,
`app/code/Magento/MediaStorage/Model/File/Uploader.php`, `app/code/Magento/Store/etc/config.xml`,
`app/code/Magento/Catalog/Model/ImageUploader.php`, `lib/internal/Magento/Framework/Filesystem/Directory/PathValidator.php`,
`app/code/Magento/Customer/Model/AccountManagementApi.php`, `app/code/Magento/Downloadable/Model/Url/DomainValidator.php`;
Adobe docs (opened November 2026): [file uploads](https://developer.adobe.com/commerce/php/development/security/file-uploads),
[mass assignment](https://developer.adobe.com/commerce/php/development/security/mass-assignment),
[SSRF](https://developer.adobe.com/commerce/php/development/security/server-side-request-forgery).

## Attack classes to design against

- **CosmicSting (CVE-2024-34102, CVSS 9.8)**: unauthenticated XXE via nested deserialization in REST/GraphQL → read env.php → stolen crypt key → forged admin JWTs → CMS-block skimmer injection (RCE when chained with CVE-2024-2961). Lessons: any API-reachable deserialization of nested structures is attack surface; XML parsers must not resolve external entities; env.php readability = total compromise.
- **Key rotation is incomplete by default**: rotating the crypt key did NOT invalidate old keys (Sansec: stores stayed compromised post-patch; Adobe AC-12485 hotfix added invalidation). Post-incident: rotate AND invalidate old keys, re-encrypt (e.g. gene/module-encryption-key-manager), rotate admin passwords + ALL integration/API tokens, and **diff all CMS blocks/pages for injected scripts**.
- Carding/DoS: keep input limits + pagination caps on (see graphql-api.md); 2.4.7+ payment-endpoint backpressure; on 2.4.6 rely on WAF rate limits.
