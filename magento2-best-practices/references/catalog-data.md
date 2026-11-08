# Catalog data: store scope, link field, URL rewrites, search attributes (Magento 2.4.x)

Code paths are in magento/magento2 at tag 2.4.9 (`vendor/magento/*` of a 2.4.9 install); behaviour on 2.4.6–2.4.8
was not checked unless a line says so. EAV attribute patches: [backend-components.md](backend-components.md#eav-attributes-in-data-patches).
Indexer modes: [performance-ops.md](performance-ops.md#indexers).

## Saving products in the right scope

- Unless code sets the store, the current store is the default store view, whatever the area: CLI commands, cron,
  queue consumers, `/V1/` REST calls and admin controllers that do not set it themselves (the product and category
  admin controllers do). `ProductRepository::get($sku)` then loads the product in that store view and `save()` writes
  there. Measured on a 2.4.9 install with sample data and one store view (November 2026, rolled back): loading with `get($sku)` from CLI and saving it unchanged created 8 store-view value
  rows (`image`, `small_image`, `thumbnail`, `tax_class_id`, `options_container`, `msrp_display_actual_price_type`,
  `news_to_date`, `custom_design_to`), so those attributes lost "Use Default Value"; `$product->save()` created 13.
  The repository's guard against copying unchanged values runs only for a store other than 0 when there are several
  store views or one website, never resets the image roles of a loaded product (they are in its data on entry), and
  compares the rest with `===`.
- To write default values, load and save in store 0: `$productRepository->get($sku, false, 0)` or
  `$storeManager->setCurrentStore(0)` first (both gave 0 store-view rows in the same test). A full save in a store view
  copies unchanged attributes there as above; to change one store-view value, use
  `Product\Action::updateAttributes([$id], [$code => $value], $storeId)` (limits below).
- Setting an attribute to `null` and saving in a store view deletes its store-view row, which restores "Use Default
  Value" (core admin does the same).
- REST scope comes from the URL: `/V1/...` is the default store view, `/all/V1/...` the admin store (default values),
  `/<store_code>/V1/...` that store view. Async bulk endpoints pass the request's store to their consumer; a custom
  consumer runs in the default store view and sets the store it needs itself.
- `ProductRepository::save()` of a new product without website IDs assigns it to the current store's website, or to
  every website when the current store is admin (so `setCurrentStore(0)` changes this too); `$product->save()` assigns
  none. Pass website IDs explicitly. New products and categories write their EAV values at store 0 regardless of the
  current store.
- `Magento\Catalog\Model\Product\Action::updateAttributes($ids, $data, $storeId)` writes raw values: no
  `catalog_product_save_*` events, no backend models, no validation. It dispatches only
  `catalog_product_attribute_update_before`, reindexes EAV (indexable attributes) and category-product (when not
  scheduled), and plugins add fulltext and stock; price and flat indexes are not reindexed (core's own mass-update
  consumer reindexes them itself), and full page cache is flushed only by an `adminhtml` plugin. On an install with one
  store view (`StoreManager::hasSingleStore()`, regardless of the single-store-mode setting) it ignores `$storeId`,
  writes store 0 and deletes the attribute's store-view rows.

## Link field (Adobe Commerce staging)

- Never hard-code `entity_id` in joins to EAV value tables (`catalog_product_entity_*`, `catalog_category_entity_*`)
  or in queries on staged entities. Get the column from
  `MetadataPool::getMetadata(ProductInterface::class)->getLinkField()`: it is read from the entity table's primary
  key, `entity_id` in Open Source and `row_id` in Adobe Commerce, where staged entities keep one row per scheduled
  version keyed by `created_in` / `updated_in` (Adobe's KB lists products, categories, catalog and cart price rules and
  CMS pages; not checkable in Open Source code). Core
  resource models do this throughout; code that joins on `entity_id` passes on Open Source and returns wrong rows on
  Commerce.

## URL rewrites

- Product rewrites are regenerated on `catalog_product_save_after` (when `url_key`, categories with category paths on,
  websites or `visibility` changed, or rewrites are missing; only for visible products), on product import, on
  website assignment and, through `catalog_product_attribute_update_before`, only when `visibility` changes. Category
  rewrites: `catalog_category_save_after` and `catalog_category_move_after`. Changing `url_key` through
  `Product\Action::updateAttributes()`, the EAV resource or SQL leaves the old rewrites; save through the repository.
- Core has no CLI command that regenerates URL rewrites.
- "URL key for specified store already exists" comes from the duplicate check of the URL rewrite storage on
  (`request_path`, `store_id`) when rewrites are saved (backed by a unique key on `url_rewrite`), not from a check on
  the `url_key` attribute. Only the admin product form and product import check URL keys before saving.
  Products that are "Not Visible Individually" get no rewrites, so a duplicate key surfaces only when visibility changes.
- `catalog/seo/generate_category_product_rewrites` defaults to 0 (also on 2.4.6 and 2.4.8). Saving it as 0 deletes the
  autogenerated category/product rewrite rows.
- On 2.4.9 product rewrite history follows the store config `catalog/seo/save_rewrites_history` (default 1) and
  overrides a `save_rewrites_history` value set on the product; on 2.4.8 only the product flag counted. A category save
  uses the `url_key_create_redirect` request flag; a category move uses the store config.

## Search and layered navigation attributes

- Saving an attribute through its resource model invalidates `catalogsearch_fulltext` only when `is_searchable`,
  `is_filterable` or `is_visible_in_advanced_search` changes. `is_filterable_in_search` and `used_for_sort_by` also put
  the attribute into the index but invalidate nothing: reindex after changing them.
- A `search_weight` change needs no reindex: it is a query-time boost, and the save resets the search request config.
- `EavSetup::addAttribute()` / `updateAttribute()` in data patches write table rows directly and bypass those plugins.
  A patch that changes these flags invalidates the index itself:
  `$indexerRegistry->get('catalogsearch_fulltext')->invalidate()`.
- A new attribute gets its OpenSearch field mapping on save only when the fulltext indexer is "Update on Save"; on
  "Update by Schedule" it waits for a full reindex.
- Layered navigation: `is_filterable` > 0 for category pages (1 hides options with no results, 2 shows them with 0),
  `is_filterable_in_search` for search results. Filter options come from the attribute's source model, so a text
  attribute without options shows nothing; the admin form allows filterable only for boolean, select, multiselect and
  price, data patches are not limited. Create a multiselect as the admin form does, `backend_type` `text` with
  `Magento\Eav\Model\Entity\Attribute\Backend\ArrayBackend`: the `catalog_product_index_eav` indexer reads
  multiselect values only from the text table.
- Saving an attribute with `is_searchable` = 0 also sets `is_visible_in_advanced_search` to 0.

## Sources

Adobe docs (opened November 2026):
[bulk endpoints and store scope](https://developer.adobe.com/commerce/webapi/rest/use-rest/bulk-endpoints),
[staging data structure (KB)](https://experienceleague.adobe.com/en/docs/experience-cloud-kcs/kbarticles/ka-27726).
Issue: magento/magento2#7879 (saving through the repository changes untouched attributes, closed).
Checked by running `ProductRepository::save()` and `Product::save()` from CLI on a 2.4.9 install inside a rolled-back
transaction (November 2026).
Core code: `module-store/Model/{StoreResolver,StoreScopeProvider}.php`, `module-catalog/Model/{Product,ProductRepository}.php`,
`module-catalog/Controller/Adminhtml/Product/Save.php`, `module-webapi/Controller/PathProcessor.php`,
`module-webapi-async/Model/OperationRepository.php`, `module-store/Model/StoreManager.php`,
`module-catalog/Controller/Adminhtml/Category/Save.php`, `module-url-rewrite/Model/Storage/DbStorage.php`,
`module-catalog-import-export/Model/Import/Product.php`, `module-catalog-url-rewrite/Observer/CategoryProcessUrlRewriteMovingObserver.php`, `module-eav/Model/ResourceModel/{ReadSnapshot,UpdateHandler}.php`,
`module-catalog/Controller/Adminhtml/Product/Initialization/Helper/AttributeFilter.php`, `module-catalog/etc/di.xml`,
`module-catalog/Model/Product/Action.php`, `module-catalog/Model/ResourceModel/Product/Action.php`,
`module-catalog/Model/Attribute/Backend/Consumer.php`, `framework/EntityManager/EntityMetadata.php`,
`module-catalog-url-rewrite/etc/{events,config}.xml`,
`module-catalog-url-rewrite/Observer/{ProductProcessUrlRewriteSavingObserver,ProcessUrlRewriteOnChangeProductVisibilityObserver}.php`,
`module-catalog-url-rewrite/Model/Product/{Validator,CurrentUrlRewritesRegenerator}.php`,
`module-catalog-url-rewrite/Model/TableCleaner.php`, `module-url-rewrite/etc/db_schema.xml`,
`module-catalog-search/Model/Indexer/Fulltext/Plugin/Attribute.php`, `module-catalog-search/Model/Attribute/SearchWeight.php`,
`module-catalog-search/Model/Search/RequestGenerator.php`,
`module-catalog-search/Setup/Patch/Data/SetInitialSearchWeightForAttributes.php`, `module-eav/Setup/EavSetup.php`,
`module-elasticsearch/Model/Indexer/Fulltext/Plugin/Category/Product/Attribute.php`,
`module-catalog/Model/ResourceModel/Product/Attribute/Collection.php`, `module-catalog/Model/Layer/Filter/AbstractFilter.php`,
`module-catalog-search/Model/Layer/Filter/Attribute.php`, `module-catalog/Model/ResourceModel/Product/Indexer/Eav/Source.php`,
`module-catalog/Model/ResourceModel/Eav/Attribute.php`.
