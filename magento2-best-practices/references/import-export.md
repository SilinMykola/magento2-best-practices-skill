# Custom import and export entities (Magento 2.4.x)

Code paths are in magento/magento2 at tag 2.4.9 (`vendor/magento/*` of a 2.4.9 install); behaviour on 2.4.6–2.4.8 was
not checked unless a line says so. Adobe's former custom-import-entity tutorial returned 404 in November 2026. Stock
import: [inventory-msi.md](inventory-msi.md#writing-stock-erp-import).

## Import entity

- Declare the entity in `etc/import.xml`; `getEntityTypeCode()` must return its `name`. Two base classes exist and are
  configured differently: `Magento\ImportExport\Model\Import\Entity\AbstractEntity` (used by products; bunch size from
  `general/file/bunch_size`) and `Magento\ImportExport\Model\Import\AbstractEntity` (used by customers; bunch size from
  `import/format_v2/bunch_size`).
- The second one groups rows by `$masterAttributeCode`. Without it no row starts a group and nothing is imported, with
  no error; one invalid row drops its whole group (same at 2.4.6).
- Match `behaviorModel` to the base class: `Behavior\Basic` (append = "Add/Update", replace, delete) for the first,
  `Behavior\Custom` (add_update, delete, custom) for the second, which turns a behavior outside `_availableBehaviors`
  into add_update silently. `_importData()` branches on `getBehavior()`, and append/add_update must update existing
  records, not only insert.
- In `_importData()` read bunches with `$this->_dataSourceModel->getNextUniqueBunch($this->getIds())`, as core entities
  do. `getNextBunch()` iterates every row of `importexport_importdata`, including other imports.
- Validation and import are separate requests in the admin (Validate, then Start), and only valid rows are stored in
  bunches. Keep no state from `validateRow()` for `_importData()`; call `validateRow()` again on each row inside your
  `_importData()`, as core product and customer entities do (the framework does not call it again).
- Column checks: headers must match `^[a-z][a-z0-9_]*$` unless listed in `_specialAttributes`; with `needColumnCheck`
  a column outside `validColumnNames` is an error; a missing `_permanentAttributes` column fails the file. The product
  base class skips the column-name checks for delete; the customer base class checks them for every behavior.

## Errors, indexers and leftovers

- Validation strategy as coded (the REST import page describes it differently): row errors never stop a
  `validation-skip-errors` import; `validation-stop-on-errors` stops when the error count is greater than
  `allowed_error_count`, not at the first error (same at 2.4.6). File-level errors (headers, a missing permanent column)
  and system errors stop the import under either strategy.
- After an admin import that was not terminated by errors, only indexers declared as `<relatedIndexer>` of the entity
  in `import.xml` are invalidated, and only those not on "Update by Schedule". Declare yours, or reindex your rows in
  `_importData()` as the product entity does.
- Product import generates URL rewrites and other side effects in observers on
  `catalog_product_import_bunch_save_after` (MSI cleans source items on `catalog_product_import_bunch_delete_after`).
  A custom entity that writes product data directly gets none of them; see [catalog-data.md](catalog-data.md#url-rewrites).
- `importData()` opens no transaction: a failure part-way leaves the rows written so far. Validated data older than a
  day is deleted at the next validation and by cron `import_data_cleanup`.

## Export entity and scheduling

- Declare the entity in `etc/export.xml` with `entityAttributeFilterType`; the model extends
  `Magento\ImportExport\Model\Export\AbstractEntity` or `...\Export\Entity\AbstractEntity`. Admin export is always
  asynchronous: it publishes `import_export.export`, the `exportProcessor` consumer writes the file (CLI context, no
  admin session) and reports a failure as an admin notification, catching only `LocalizedException` and
  `FileSystemException`.
- Magento Open Source has no import queue and no scheduled import/export; Adobe documents scheduled import/export as
  Adobe Commerce only.

## Sources

Adobe docs (opened November 2026): [REST import](https://developer.adobe.com/commerce/webapi/rest/modules/import/),
[scheduled import/export](https://experienceleague.adobe.com/en/docs/commerce-admin/systems/data-transfer/data-scheduled-import-export).
Core code: `module-import-export/Model/Import.php`, `module-import-export/Model/Import/AbstractEntity.php`,
`module-import-export/Model/Import/Entity/AbstractEntity.php`, `module-import-export/Model/ResourceModel/Import/Data.php`,
`module-import-export/Model/Source/Import/Behavior/{Basic,Custom}.php`,
`module-import-export/Model/Import/ErrorProcessing/ProcessingErrorAggregator.php`,
`module-import-export/Controller/Adminhtml/Import/Start.php`, `module-import-export/Helper/Data.php`,
`module-import-export/etc/{import,export}.xsd`, `module-import-export/Controller/Adminhtml/Export/Export.php`,
`module-import-export/Model/{Export,Export/Consumer}.php`, `module-catalog-import-export/Model/Import/Product.php`, `module-customer-import-export/Model/Import/Customer.php`,
`module-catalog-url-rewrite/etc/events.xml`, `module-inventory-import-export/etc/events.xml`.
Older tag compared on GitHub (magento/magento2, tag 2.4.6, November 2026).
