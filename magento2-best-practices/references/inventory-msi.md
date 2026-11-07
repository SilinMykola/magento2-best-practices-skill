# Inventory: Multi-Source Inventory (Magento 2.4.x)

MSI ships in magento/inventory (2.4.9 locks `magento/inventory-metapackage` 1.2.9; 2.4.6 = 1.2.6, 2.4.7 = 1.2.7).
Paths below are in magento/inventory, `CatalogInventory` paths in magento/magento2. Behaviour is the same on
2.4.6–2.4.9 unless a line says otherwise.

## Is MSI there at all

- Never assume it: many stores disable the `Magento_Inventory*` modules or remove them with Composer `replace`. Code
  that needs MSI declares it in `composer.json` and `<sequence>`. Optional MSI code lives in a bridge module, or
  branches on `ModuleManagerInterface::isEnabled('Magento_InventorySales')`: check the implementation module, not
  `interface_exists()` (the preferences are in `Magento_InventorySales/etc/di.xml`; a disabled module leaves an
  interface that loads but cannot be instantiated). Without MSI the legacy `CatalogInventory` API is the only path.

## Reading: can this be sold

- Ask through `InventorySalesApi` with the stock resolved from the website: `AreProductsSalableInterface` for status,
  `AreProductsSalableForRequestedQtyInterface` for a quantity (also checks min/max sale qty, increments, decimals).
  `IsProductSalableInterface` and `IsProductSalableForRequestedQtyInterface` are `@deprecated in favor of bulk API`.
- `GetProductSalableQtyInterface` only for simple, virtual and downloadable products; for configurable, bundle and
  grouped it throws `InputException`. Its result is stock qty plus reservations minus the out-of-stock threshold.
- Resolve the stock with `StockResolverInterface::execute(SalesChannelInterface::TYPE_WEBSITE, $websiteCode)
  ->getStockId()` (website code, not ID); default stock via `DefaultStockProviderInterface::getId()`, never a literal
  `1`.
- Outside the storefront take the website from the order, quote or store, never from the current context: in admin
  the current website is `admin`, which `AdaptStockResolverToAdminWebsitePlugin` (on
  `GetStockBySalesChannelInterface`) maps to the default stock without error; in cron and CLI it falls back to the
  default store's website. Both are silently wrong with several stocks.
- A quantity shown on a cacheable page needs cache identities or private content: SKILL.md non-negotiable 10.

## Legacy CatalogInventory

- `StockRegistryInterface`, `StockStateInterface` and `StockItemRepositoryInterface` are
  `@deprecated 100.3.0 Replaced with Multi Source Inventory`. Plugins make `getStockStatus*()` return salable data, but
  `getStockItem*()->getQty()`, `cataloginventory_stock_item.qty` and `StockStateInterface` qty methods are the default
  source's physical quantity: no reservations, no other sources. With MSI, order placement does not decrement it
  (`SubtractQuoteInventoryObserver` is disabled).
- Never set stock through legacy stock-item saves (`updateStockItemBySku()` etc.): they write only the default source.

## Reservations

- Reservations (`inventory_reservation`) are append-only: never insert, update or delete rows and never call
  `AppendReservationsInterface` from business code. Core places them on order placement, cancel, shipment, invoice of
  virtual/downloadable items and credit memo; for a custom sales event use `PlaceReservationsForSalesEventInterface`.
  Fix drift with `bin/magento inventory:reservation:list-inconsistencies -r | bin/magento
  inventory:reservation:create-compensations`.
- Create orders through `CartManagementInterface::placeOrder()` or `OrderManagementInterface::place()`, never
  `OrderRepositoryInterface::save()`: the reservation comes from a plugin on `OrderManagementInterface::place()`.
- Keep the `inventory.reservations.updateSalabilityStatus` consumer running.

## Writing stock (ERP import)

- Write source quantities only through `SourceItemsSaveInterface` (REST `POST /V1/inventory/source-items`) with changed
  rows batched in one call; assignments and moves through `InventoryCatalogApi` bulk services
  (`BulkSourceAssignInterface`, `BulkSourceUnassignInterface`, `BulkInventoryTransferInterface`). Their plugins sync
  the legacy tables, composite parents and indexes; raw SQL on `inventory_source_item` skips all of that. Never write
  `inventory_stock_<id>` (for the default stock, `inventory_stock_1` is a view over `cataloginventory_stock_status`).
  Unassigning a source clears its quantity.
- "Update by Schedule" on the `inventory` indexer does not defer the reindex after `SourceItemsSave`: the save plugin
  reindexes in the same request. For large imports set `cataloginventory/indexer/strategy` to `async` and run the
  `inventory.indexer.sourceItem` and `inventory.indexer.stock` consumers.

## Shipments and source selection

- Always set `source_code` on the shipment's extension attributes when creating it in code (REST:
  `arguments.extension_attributes.source_code` on `POST /V1/order/{id}/ship`). Without it
  `AssignSourceCodeToShipmentPlugin` (on `ShipmentFactory::create()`) uses the stock's only source, or else the
  `default` source — with several sources stock is deducted from `default` silently.
- Pick sources with `SourceSelectionServiceInterface::execute($request, $algorithmCode)` (`priority` by default, or
  `distance`). Source selection only recommends; deduction happens on shipment save.

## Sources

Adobe docs (opened November 2026): [reservations](https://developer.adobe.com/commerce/php/development/framework/inventory-management/reservations),
[inventory web API](https://developer.adobe.com/commerce/php/development/components/web-api/inventory-management),
[manage source items](https://developer.adobe.com/commerce/webapi/rest/inventory/manage-source-items),
[create shipment](https://developer.adobe.com/commerce/webapi/rest/tutorials/inventory/create-shipment/),
[selection and reservations](https://experienceleague.adobe.com/en/docs/commerce-admin/inventory/basics/selection-reservations),
[inventory CLI](https://experienceleague.adobe.com/en/docs/commerce-admin/inventory/cli),
[install/disable MSI](https://experienceleague.adobe.com/en/docs/commerce-admin/inventory/get-started/install-update),
[inventory config](https://experienceleague.adobe.com/en/docs/commerce-admin/config/catalog/inventory).
Code: `InventorySalesApi/Api/*.php` (tags 1.2.6 and 1.2.9), `InventorySales/Model/GetProductSalableQty.php`,
`InventorySales/Model/GetStockBySalesChannel.php`, `InventorySales/etc/{di,events}.xml`,
`InventorySales/Plugin/Sales/OrderManagement/AppendReservationsAfterOrderPlacementPlugin.php`,
`InventoryCatalog/Plugin/InventorySalesApi/StockResolver/AdaptStockResolverToAdminWebsitePlugin.php`,
`InventoryCatalog/Plugin/CatalogInventory/UpdateSourceItemAtLegacyStockItemSavePlugin.php`,
`InventoryCatalog/Setup/Patch/Schema/CreateLegacyStockStatusView.php`,
`InventoryIndexer/Plugin/InventoryApi/ReindexAfterSourceItemsSavePlugin.php`,
`InventoryShipping/Observer/SourceDeductionProcessor.php`,
`InventoryShipping/Plugin/Sales/Shipment/AssignSourceCodeToShipmentPlugin.php`; magento/magento2 `app/code/Magento/CatalogInventory/Api/*.php`.
