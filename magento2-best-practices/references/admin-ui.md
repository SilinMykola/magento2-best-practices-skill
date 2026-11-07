# Admin UI components (Magento 2.4.x)

ACL for controllers, `system.xml` and `aclResource`: [security.md](security.md#admin-acl-in-code). Code paths are in
magento/magento2 at tag 2.4.9; same on 2.4.6 unless a line says otherwise.

## Listings

- Map the listing's `<dataProvider name>` to a collection in the `collections` argument of
  `Magento\Framework\View\Element\UiComponent\DataProvider\CollectionFactory` (a `SearchResult` subclass or virtualType
  with `mainTable` and `resourceModel`). A missing mapping throws "Not registered handle".
- Keyword search (`<filterSearch name="fulltext"/>`) needs an `indexType="fulltext"` index on the main table in
  `db_schema.xml`; without one `FulltextFilter` ignores the keyword with no error. Joined columns are never searched.
  Adobe's admin-grid tutorial does not mention the index.

## Forms

- A form data provider's `getData()` returns `[$entityId => $fields]`, with key `''` for a new entity.
- After a failed save: `DataPersistorInterface::set($key, $postData)` and redirect to `*/*/edit` with the id; on success
  `clear($key)`. Your form data provider must read the persisted data, merge it over the loaded entity and clear it;
  do not copy core CMS providers here (the Page provider uses it only for a new page, so edits are lost on error). The
  persistor lives in the session, so data that is never cleared reappears on the next form open.
- Product form: extend `Magento\Catalog\Ui\DataProvider\Product\Form\Modifier\AbstractModifier` and add it to the
  `modifiers` argument of virtualType `Magento\Catalog\Ui\DataProvider\Product\Form\Modifier\Pool` with a non-zero
  `sortOrder` (`0` throws "The parameter "sortOrder" is missing"). Modifiers run in ascending `sortOrder` for both
  `modifyData` and `modifyMeta`; place yours after the one whose output you change (core `eav` is 10).
- Category form has no modifier pool for data: extend it with `category_form.xml` plus plugins on
  `Magento\Catalog\Model\Category\DataProvider` (`afterGetData`, `afterPrepareMeta`), as core does.

## Mass actions and inline edit

- Mass action controllers take items from `$this->filter->getCollection($this->collectionFactory->create())`
  (`Magento\Ui\Component\MassAction\Filter`); never read `selected` yourself — "Select All" posts `excluded=false` and
  `Filter` rebuilds the listing with its current filters. Implement `HttpPostActionInterface` and declare your own
  `ADMIN_RESOURCE`: `Filter` does not check the listing's `aclResource`, so the constant is the only check.
- Inline edit controllers are POST and AJAX only, have a save-level `ADMIN_RESOURCE`, read `items[<id>][<field>]`,
  merge each item over the stored entity, run the same validation as Save, and return JSON `{messages, error}`.

## Sources

Adobe docs (opened November 2026): [admin grid tutorial](https://developer.adobe.com/commerce/php/development/components/add-admin-grid),
[modifiers](https://developer.adobe.com/commerce/frontend-core/ui-components/concepts/modifier),
[mass actions](https://developer.adobe.com/commerce/frontend-core/ui-components/components/mass-actions).
Core code: `lib/internal/Magento/Framework/View/Element/UiComponent/DataProvider/{CollectionFactory,FulltextFilter}.php`,
`app/code/Magento/Ui/Component/Form.php`, `app/code/Magento/Ui/DataProvider/Modifier/Pool.php`,
`app/code/Magento/Catalog/etc/adminhtml/di.xml`, `app/code/Magento/Catalog/Model/Category/DataProvider.php`,
`app/code/Magento/Ui/Component/MassAction/Filter.php`, `lib/internal/Magento/Framework/App/Request/DataPersistor.php`,
`app/code/Magento/Cms/Controller/Adminhtml/Page/{Save,MassDelete,InlineEdit}.php`,
`app/code/Magento/Cms/Model/Page/DataProvider.php`, `app/code/Magento/Cms/etc/{di,db_schema}.xml`.
