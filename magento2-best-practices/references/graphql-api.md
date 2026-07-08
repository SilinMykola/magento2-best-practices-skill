# GraphQL & Web API (Magento 2.4.x)

## Resolvers

- **Thin resolvers**: validate input → check auth from `$context` → delegate to the base module's service contracts → map to output. No business logic in the GraphQl module.
- **Batch to kill N+1**: prefer `BatchResolverInterface` (all requests for a field arrive at once — load children in ONE query, group by parent) or `BatchServiceContractResolverInterface` (converts requests to a batch service contract's DTOs; preserve result ordering). Core reference: `AbstractLinkedProducts`.
- Plain `ResolverInterface` that must aggregate → return `Resolver\Value` (deferred; callable invoked at the latest possible moment, letting you buffer IDs across invocations).
- **Errors**: throw the specific `GraphQl*` exceptions (`GraphQlInputException`, `GraphQlAuthorizationException`, `GraphQlAuthenticationException`, `GraphQlNoSuchEntityException`, `GraphQlAlreadyExistsException`). Only `ClientAware` exceptions surface messages to clients; everything else logs to exception.log and returns a generic internal error in production. Don't leak internals via ClientAware.
- **Auth**: `$context->getExtensionAttributes()->getIsCustomer()` / `$context->getUserId()`; throw `GraphQlAuthorizationException` when absent. GraphQL has NO admin-ACL layer — admin-privileged ops belong in REST with ACL resources. Never trust IDs from the client where the authenticated identity should be used.

## GraphQL caching

- Every cacheable custom query needs an Identity class (`Resolver\IdentityInterface::getIdentities()` returning per-entity tags + umbrella prefix tag) wired via `@cache(cacheIdentity: "FQCN")` in schema.graphqls. Session/customer-specific queries → `@cache(cacheable: false)`.
- Only **GET** GraphQL requests are FPC/Varnish-cached; POST never. **One uncacheable query in a request poisons cacheability for the whole request** — separate cacheable and uncacheable queries into different requests. Carry context via `Store`/`Content-Currency` headers, not cookies. `X-Magento-Cache-Id` (2.4.4+) enables logged-in caching on Varnish/Fastly only.
- 2.4.7+ has `Magento_GraphQlResolverCache` (server-side resolver-result caching, works for POST). Not on 2.4.6.
- Invalidation: entity saves (via identity tags), config changes, cache flush. Design Identity classes so saving entity X flushes exactly the queries embedding X.

## Schema design (schema.graphqls)

- Types PascalCase; fields/args snake_case; mutation inputs as dedicated `XxxInput` types; entity outputs include `id: ID`; `@resolver(class:)` + `@doc(description:)` on everything.
- Schemas merge across modules — you can add fields to other modules' types and values to shared enums. GraphQL schema lives in the `*GraphQl` companion module.
- **Never remove/rename fields**: additive evolution only; mark obsolete fields `@deprecated(reason: "Use X instead")` and keep them working (the Next.js frontend consumes these).

## Hardening & limits

- Query depth limit 20 / complexity 300 by default (all 2.4.x) — override via di.xml if genuinely needed, never disable.
- Disable introspection in production: `env.php` → `'graphql' => ['disable_introspection' => 1]`.
- Web API input limiting exists since 2.4.3 but is **OFF by default** — enable `webapi/validation/input_limit_enabled` + `graphql/validation/input_limit_enabled` (defaults once on: 20 items/array sync, 5000 async, page size 20/max 300; per-route override via `input-array-size-limit` in webapi.xml).
- 2.4.7+ native rate limiting ("backpressure", Redis-backed, off by default) for payment/order REST+GraphQL endpoints — carding defense. Not on 2.4.6: rely on WAF/CDN rate limits there.
- Prefer stateless Bearer-token GraphQL; sessions can be disabled (`graphql/session/disable 1` — believed 2.4.7+, verify before using on 2.4.6). Customer tokens default 1h lifetime; admin tokens 4h (REST-only generation).
- Since 2.4.4 integration tokens are rejected as standalone Bearer tokens — use full OAuth 1.0a; do NOT re-enable `oauth/consumer/enable_integration_as_bearer`.
- 2.4.9: CAPTCHA enforced on REST/GraphQL customer creation (headless FE must send tokens or admin-exempt the endpoint).

## REST (webapi.xml)

- Routes start `/V1`, params as `:param`; every route declares `<resources>`: a real ACL id, `self` (authenticated user), or `anonymous` (minimize!). `secure="true"` forces HTTPS.
- Identity substitution: `<data><parameter name="customerId" force="true">%customer_id%</parameter></data>` so clients can't spoof the authenticated user's ID.
- Serialization is driven by docblocks: every service method needs `@param`/`@return` with FULLY-QUALIFIED class names (even with `use` imports), arrays as `Type[]`. Data-interface constructor params must not be services; avoid a param literally named `item` (SOAP conflict).
- Heavy writes → async (`/rest/<store>/async/V1/...`) or bulk (`/async/bulk/V1/...`, path params become `bySku` style) — returns `bulk_uuid`, needs RabbitMQ + `async.operations.all` consumer; GET unsupported; check status via `/V1/bulk/:uuid/status`.
