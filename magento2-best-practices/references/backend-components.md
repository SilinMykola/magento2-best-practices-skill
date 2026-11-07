# Backend components: attributes, CLI, cron, logging, config, queues, indexers, cache types, email (Magento 2.4.x)

Performance rules for these components (Proxy for CLI command dependencies, cron every minute, mview subscriptions,
`Recurring` setup, synchronous topics, cron groups) are in [magento-perf-check](https://github.com/SilinMykola/magento-perf-check/blob/main/docs/SPEC.md#4-rule-catalog) and
[performance-ops.md](performance-ops.md), not here. Code paths below are in magento/magento2
at tag 2.4.9 unless a line says otherwise.

## Extension attributes

- Declaring an attribute in `etc/extension_attributes.xml` persists nothing. Load it in `afterGet`/`afterGetList`
  plugins on the repository and write it in `afterSave` through your own repository; in `afterGetList` load all items
  in one query. `<join>` is read-only and works only where the repository's `getList()` calls
  `JoinProcessorInterface::process()` (Product, Order, Quote, Customer repositories among others).
- `getExtensionAttributes()` can return null on plain DTOs (their interfaces declare `|null`); models extending
  `AbstractExtensibleModel` create the object lazily. On null, create it with `ExtensionAttributesFactory`.

## EAV attributes in data patches

- `EavSetup::addAttribute()` with `user_defined => true` and no `group` assigns the attribute to no attribute set, so it
  never shows in admin forms. With `group` (or `user_defined` false) it goes into every existing set. Calling it again
  for an existing code updates the attribute.
- Customer attributes need two more steps: `addAttributeToSet(Customer::ENTITY, <default set id>, null, $code)`, then
  load the attribute with `getEavConfig()->getAttribute()`, `setData('used_in_forms', ['adminhtml_customer', ...])` and
  save it. `used_in_forms` is not an `addAttribute()` option.
- Patch exception handling: module-development.md, "Database schema & patches".

## CLI commands

- Register commands in the `commands` argument of `Magento\Framework\Console\CommandListInterface` (di.xml). Every
  registered command is constructed on every `bin/magento` run, and Symfony's constructor calls `configure()`: do no
  work in either. In developer mode an exception while commands load stops all of `bin/magento`.
- Declare `execute(InputInterface $input, OutputInterface $output): int` and return `Cli::RETURN_SUCCESS` /
  `Cli::RETURN_FAILURE`. 2.4.9 ships symfony/console 7.4, where the `: int` return type is required (2.4.6: ^5.4;
  2.4.7–2.4.8: ^6.4).
- The CLI starts with no area: `getAreaCode()` throws "Area code is not set", a second `setAreaCode()` throws "Area code
  is already set". Set it inside `execute()`, or use `State::emulateAreaCode($area, $callback)` for a scoped call (it
  does not switch the DI config scope). Queue consumers run in `global` unless started with `--area-code`.

## Cron

- `<config_path>` makes a schedule admin-configurable; it falls back to `<schedule>` when the value is empty.
- Core locks each job against overlapping runs of itself (`Schedule::tryLockJob()`). Let exceptions escape: core marks
  the run `error` in `cron_schedule`, logs it, and never retries it. So jobs must be idempotent and resumable; add your
  own lock only when the same work can also start from CLI or a consumer.

## Logging

- Inject `Psr\Log\LoggerInterface`. Custom file: a virtualType of `Magento\Framework\Logger\Handler\Base` with argument
  `fileName`, wrapped in a virtualType of `Magento\Framework\Logger\Monolog` with `handlers`, injected as `logger`.
  The minimum level is the protected `$loggerType` property, so changing it needs a subclass.
- 2.4.8+ ships Monolog 3 (2.4.9: 3.10, psr/log 3); 2.4.6–2.4.7 ship Monolog 2. A custom handler on Monolog 3 receives
  a `LogRecord` object (`write(LogRecord $record): void`); on Monolog 2 it receives an array.
- Core's `var/log/debug.log` handler drops `debug()` in production mode unless debug logging is enabled
  (`setup:config:set --enable-debug-logging=true`). A custom `Handler\Base` virtualType keeps its default
  `$loggerType = Logger::DEBUG` and writes `debug()` in every mode; subclass it to raise the level.

## Configuration

- Always pass scope and code: `$scopeConfig->getValue($path, ScopeInterface::SCOPE_STORE, $storeId)`. Without them the
  `default` scope is read and website/store values are ignored. `isSetFlag()` is `(bool) getValue()`.
- Secrets: `backend_model="Magento\Config\Model\Config\Backend\Encrypted"` is needed on both the `system.xml` field
  (otherwise admin saves plaintext) and the `config.xml` node (`getValue()` then returns the decrypted value; crypto
  rules in [security.md](security.md)). Also list the path in the `sensitive` argument of
  `Magento\Config\Model\Config\TypePool` (di.xml) so `app:config:dump` keeps it out of `config.php`.

## Store emulation

- Rendering or sending store content from cron, CLI, admin or a consumer:
  `$emulation->startEnvironmentEmulation($storeId, Area::AREA_FRONTEND, true)` with `stopEnvironmentEmulation()` in
  `finally`. Emulation is single-level: a second start while one is active does nothing, so a missed stop leaves later
  code on the wrong store and locale. It switches store, theme, locale and inline translation, not the area code.

## Message queues

- A handler exception is final: the consumer catches `\Exception` and calls `reject($message, false)`, nothing is
  requeued. On MySQL the message gets status error; on AMQP it is dropped unless the queue has a dead-letter exchange.
  `NotFoundException` acknowledges the message with only a warning. `\Error` is not caught and stops the consumer. The
  broker can also redeliver after a lost connection, so handlers are idempotent; to retry, catch and republish
  yourself. Same on 2.4.6.
- Omit `connection` in `queue_*.xml` so `env.php` decides: 2.4.9 picks `queue/default_connection`, then stomp, then
  amqp, then db (2.4.8 has no stomp).

## Custom indexers and mview

- `mview.xml` triggers write the subscribed table's `entity_column` into `<view_id>_cl` on INSERT, DELETE (the old
  value) and UPDATE of **any** column except the ignored ones (default: `updated_at`), not only the columns your index
  reads. `setup:upgrade` recreates the triggers of scheduled indexers. Since 2.4.8 a newly installed indexer starts in
  Update by Schedule.
- Implement `Magento\Framework\Indexer\ActionInterface` (`executeFull`, `executeList`, `executeRow`) and
  `Magento\Framework\Mview\ActionInterface::execute($ids)` delegating to `executeList`. The ids include deleted
  entities, so `executeList` also removes index rows whose source is gone; for the same ids it must produce exactly
  what a full reindex would.
- `execute()` is idempotent (delete-then-insert or `insertOnDuplicate`): `View::update()` saves `version_id` only after
  all batches succeed, so after an exception the whole range runs again on the next cron.
- Full reindex without downtime: fill a `<table>_replica` declared in `db_schema.xml`, then
  `Magento\Catalog\Model\ResourceModel\Indexer\ActiveTableSwitcher::switchTable()` (Magento_Catalog, not `@api`:
  pin the dependency and re-check it on upgrade); partial reindexes write the active table.
- When a config change makes the index stale, call `IndexerRegistry::get($id)->invalidate()` from the backend model's
  `afterSave` when the value changed, and let cron rebuild it (core `PriceScope`). A changed `indexer.xml` marks the
  indexer invalid on `setup:upgrade`.
- Product price index with dimension modes (`indexer:set-dimensions-mode catalog_product_price`): the core table
  resolver plugin maps `catalog_product_index_price` to the dimension table only in the `frontend`, `webapi_rest` and
  `webapi_soap` areas. In `graphql`, admin, cron or CLI resolve it with `PriceTableResolver::resolve()` (not `@api`:
  re-check on upgrade).

## Custom cache types

- `etc/cache.xml` `<type name instance>`; the class extends `Magento\Framework\Cache\Frontend\Decorator\TagScope` with
  `TYPE_IDENTIFIER` and `CACHE_TAG` and is built from `FrontendPool::get(TYPE_IDENTIFIER)`. `save()` always adds
  `CACHE_TAG`; `clean()` removes only this type's entries; `clean(CLEANING_MODE_MATCHING_TAG, [$tag])` removes entries
  carrying both tags.
- On an existing install a new type stays disabled until `bin/magento cache:enable <id>` (only `setup:install` enables
  all types). A disabled type returns false from `load()` and ignores `save()`, so code must work with it off.

## Transactional email

- `TransportBuilder` with `setTemplateOptions(['area' => ..., 'store' => $storeId])` (both keys, or "The design config
  needs an area and a store"), `setFromByScope($identity, $storeId)` (`setFrom()` is deprecated since framework 102.0.1
  and ignores the store), and the send wrapped in `$inlineTranslation->suspend()` / `resume()` in `finally`.
- The `email_templates.xml` template id equals the config path with `/` replaced by `_`; the admin dropdown takes its
  "(Default)" option from that.

## Sources

Adobe docs (opened November 2026): [extension attributes](https://developer.adobe.com/commerce/php/development/components/add-attributes),
[custom customer attribute](https://developer.adobe.com/commerce/php/tutorials/admin/custom-text-field-attribute),
[CLI commands](https://developer.adobe.com/commerce/php/development/cli-commands/custom),
[custom cron](https://experienceleague.adobe.com/en/docs/commerce-operations/configuration-guide/crons/custom-cron-reference),
[custom log files](https://experienceleague.adobe.com/en/docs/commerce-operations/configuration-guide/logs/custom-log-files),
[sensitive settings](https://developer.adobe.com/commerce/php/development/configuration/sensitive-environment-settings),
[queue configuration](https://developer.adobe.com/commerce/php/development/components/message-queues/configuration).
Core code: `lib/internal/Magento/Framework/Api/ExtensionAttribute/JoinProcessor.php`,
`lib/internal/Magento/Framework/Model/AbstractExtensibleModel.php`, `app/code/Magento/Eav/Setup/EavSetup.php`,
`app/code/Magento/Customer/Setup/Patch/Data/DefaultCustomerGroupsAndAttributes.php`,
`lib/internal/Magento/Framework/Console/Cli.php`,
`lib/internal/Magento/Framework/App/State.php`, `app/code/Magento/Cron/Observer/ProcessCronQueueObserver.php`,
`lib/internal/Magento/Framework/Logger/Handler/Base.php`, `app/code/Magento/Developer/Model/Logger/Handler/Debug.php`,
`lib/internal/Magento/Framework/App/Config/ScopeConfigInterface.php`, `app/code/Magento/Paypal/etc/di.xml`,
`app/code/Magento/Store/Model/App/Emulation.php`, `lib/internal/Magento/Framework/MessageQueue/Consumer.php`,
`app/code/Magento/MysqlMq/Model/Driver/Queue.php` (`reject()`),
`lib/internal/Magento/Framework/MessageQueue/DefaultValueProvider.php`,
`lib/internal/Magento/Framework/Mail/Template/TransportBuilder.php`, `app/code/Magento/Contact/Model/Mail.php`,
`lib/internal/Magento/Framework/Mview/{View,View/Subscription}.php`, `app/code/Magento/Indexer/etc/di.xml`
(`ignoredUpdateColumns`), `app/code/Magento/Indexer/Setup/Recurring.php` (2.4.6 vs 2.4.8),
`app/code/Magento/Catalog/Model/ResourceModel/Indexer/ActiveTableSwitcher.php`,
`app/code/Magento/Catalog/Model/Indexer/Product/Price/{PriceTableResolver,Plugin/TableResolver}.php`,
`app/code/Magento/Catalog/Model/Indexer/Product/Price/System/Config/PriceScope.php`,
`lib/internal/Magento/Framework/Cache/Frontend/Decorator/TagScope.php`, `setup/src/Magento/Setup/Model/Installer.php`
(`enableCaches`); [indexing](https://developer.adobe.com/commerce/php/development/components/indexing),
[custom indexer](https://developer.adobe.com/commerce/php/development/components/indexing/custom-indexer),
[indexer optimization](https://developer.adobe.com/commerce/php/development/components/indexing/optimization),
[cache type](https://developer.adobe.com/commerce/php/development/cache/partial/cache-type);
`composer.lock` of 2.4.9 and `composer.json` at tags 2.4.6–2.4.8 for symfony/console and monolog versions.
