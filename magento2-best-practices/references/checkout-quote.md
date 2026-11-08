# Checkout, quote and shipping (Magento 2.4.x)

Code paths are in magento/magento2 at tag 2.4.9 (`vendor/magento/*` of a 2.4.9 install); behaviour on 2.4.6–2.4.8
was not checked. Payment methods: [payments.md](payments.md). Extension attributes: [backend-components.md](backend-components.md#extension-attributes).

## Totals collectors (fees, surcharges)

- Declare the collector in `etc/sales.xml`, section `quote`, group `totals`. Items take `name`, `instance`,
  `sort_order` and optional `<renderer>` children (used by the admin order-create totals); there is no
  `before`/`after`. Core order: subtotal 100, tax_subtotal 200, weee 225, discount 300, shipping 350, tax_shipping 375,
  shipping_discount 400, tax 450, weee_tax 460, grand_total 550. A fee needs a `sort_order` lower than 550: Grand sums
  only the amounts set before it runs. The sorted list is kept in the config cache, so a changed `sort_order` needs a
  cache clean.
- A taxable fee also sorts before `tax` (450) and adds itself to the address `associated_taxables` (keys in the
  `KEY_ASSOCIATED_TAXABLE_*` constants of `Magento\Tax\Model\Sales\Total\Quote\CommonTaxCollector`, read by
  `Tax::mapQuoteExtraTaxables()`); Weee does the same per item. How tax is then split and displayed was not checked.
- `collect()` runs once per quote address, billing included, and the quote grand total sums all addresses. In a
  single-shipping quote only one address holds items (billing for a virtual quote, shipping otherwise). Return early
  when `$shippingAssignment->getItems()` is empty, as core Shipping, Tax and Weee do; otherwise the fee lands on two
  addresses and the quote total no longer matches the order built from one of them.
- Call `parent::collect()` first (it sets the address and total that the `_setAmount()`/`_addAmount()` helpers write
  to), then write with `$total->setTotalAmount($code, $amount)` and `setBaseTotalAmount()`, `$code` being the item
  `name`. Do not touch `grand_total`: the Grand collector sums all total amounts. `setTotalAmount('fee', ...)` also sets
  `fee_amount` on the total, which is copied onto the address object; it is stored only if `quote_address` has a
  `fee_amount` column.
- `fetch()` returns `['code' => ..., 'title' => __('...'), 'value' => ...]` (or null to hide the row). It becomes a
  `total_segments` entry of the cart totals API. Three traps:
  - the `$total` passed to `fetch()` is rebuilt from the address data, so `getTotalAmount('fee')` is 0 there; read
    `$total->getData('fee_amount')`;
  - for a non-virtual quote the cart totals API reads the shipping address without collecting totals, so the fee must
    be in a `quote_address` column or it is missing from `total_segments`;
  - the title is kept only when it is a `Phrase`; a plain string becomes `''`.
- GraphQL `CartPrices` has no generic fee field (`grand_total` already includes it); extend the type with your own
  resolver. Its resolver collects totals on every query regardless of the collected flag, so collectors must be cheap
  and give the same result when run again.

## Fee on the order, invoice and credit memo

- A quote-address field listed in `etc/fieldset.xml` `sales_convert_quote_address` / `to_order` does not reach the
  order unless `Magento\Sales\Model\Order` has a real setter for it: `ToOrder::convert()` fills the order through
  `DataObjectHelper::populateWithArray()`, which keeps only keys with a declared `set*` method and drops the rest
  silently (on 2.4.9 a `fee_amount` key is dropped, `subtotal_incl_tax` is kept). Copy the fee with an `afterConvert`
  plugin on `ToOrder::convert()`, which receives the address, into your own `sales_order` columns. Multishipping also
  converts each address through `ToOrder` but never dispatches `sales_model_service_quote_submit_before` (it dispatches
  `checkout_type_multishipping_create_orders_single`), and that event receives the quote, not the address. The
  quote-level fieldset `sales_convert_quote` does copy custom keys.
- Invoice and credit memo need their own collectors in `sales.xml` sections `order_invoice` / `order_creditmemo`,
  extending `Magento\Sales\Model\Order\Invoice\Total\AbstractTotal` / `...\Creditmemo\Total\AbstractTotal`, plus order
  columns for the invoiced and refunded fee. Each collector adds its amount to `grand_total` itself (the invoice Grand
  collector does nothing).
  - Invoice: skip the fee when an earlier non-cancelled invoice already carries it, as core `Invoice\Total\Shipping`
    does; otherwise partial invoices bill it twice.
  - Credit memo: cap the fee at the order fee minus the fee already refunded, as core `Creditmemo\Total\Shipping` does
    with `getShippingAmount() - getShippingRefunded()`; the invoice check copied here refunds the fee again on every
    credit memo.
- Order, invoice and credit memo pages in admin and customer account, and the sales emails, show the row through a
  block with `initTotals()` that calls `addTotal()` under `order_totals`, `invoice_totals` and `creditmemo_totals`,
  added in every layout handle that renders them (core Weee does it in 22 layout files).
- Cart and checkout summary need a JS component extending `Magento_Checkout/js/view/summary/abstract-total` that reads
  `totals.getSegment('<code>')`, registered in both `checkout_cart_index.xml` (`block-totals`) and
  `checkout_index_index.xml` (sidebar summary `totals`).

## Shipping carriers

- Extend `Magento\Shipping\Model\Carrier\AbstractCarrier` (or `AbstractCarrierOnline` for an external rate API) and
  implement `CarrierInterface`. `$_code` must equal the `carriers/<code>` config key: `getConfigData()` builds the path
  from it and returns false when it is empty. The carrier code must not contain `_`; the method code may (the shipping
  method string is split at the first underscore).
- Core checks `carriers/<code>/active` before it creates the carrier, then `sallowspecific` / `specificcountry` /
  `showmethod` / `specificerrmsg` and `processAdditionalValidation()`. `collectRates()` returning false or an empty
  `Result` shows nothing. When `showmethod` is 0, a `Result` that holds an `Error` is dropped whole, valid rates in it
  included, so never append an error to a result that also has rates; an `Error` returned on its own is always shown.
- Keys of `getAllowedMethods()` are the method codes used in the rates; the admin "all methods" source builds
  `carrier_method` options from them.
- Rates are re-requested only when the address flag `collect_shipping_rates` is set (a `quote_address` column). Code
  that changes items, weight or address outside core services calls `setCollectShippingRates(true)` before
  `collectTotals()`. When the selected method is not among the new rates, its method and amount are cleared.
  `AbstractCarrierOnline` caches quotes in a static array: one PHP process only, despite the "session" in its docblock.

## Quote handling in code

- `CartRepositoryInterface::get()` loads any quote, including inactive (already ordered) ones and those of other stores;
  `getActive()` / `getActiveForCustomer()` throw `NoSuchEntityException` for inactive quotes. Use them for anything the
  customer acts on. `getActive()` does not check ownership: with a numeric cart ID from a request, compare the quote's
  customer ID with the current customer (or use `getActiveForCustomer()`). The repository keeps one instance per ID per
  request, so its state is shared between services.
- `collectTotals()` does nothing once `totals_collected_flag` is set on that instance. After changing a collected quote:
  `setTotalsCollectedFlag(false)`, `collectTotals()`, then save through the repository.
- `MaskedQuoteIdToQuoteIdInterface::execute()` checks neither ownership nor active state. A custom endpoint taking a
  masked ID does what `Magento\QuoteGraphQl\Model\Cart\GetCartForUser` does: require an active cart, compare the
  customer ID, and allow guests only on carts with `customer_id` 0.

## Checkout fields

- Add fields through a `layoutProcessors` item ([frontend.md](frontend.md#javascript), the `jsLayout` rule), as core
  captcha, gift message, PayPal and in-store pickup do.
- Quote-address `custom_attributes` are dropped on the server in Magento Open Source: the address custom attribute list
  is empty and unknown codes are discarded silently. Copy the value into `extension_attributes` with a JS mixin
  (Adobe's tutorial uses one on `Magento_Checkout/js/action/set-shipping-information`), declare it in
  `extension_attributes.xml` on `Magento\Quote\Api\Data\AddressInterface`, and persist it yourself
  ([backend-components.md](backend-components.md#extension-attributes)); Adobe's tutorial does not store it either.

## Sources

Adobe docs (opened November 2026):
[add a custom checkout address field](https://developer.adobe.com/commerce/php/tutorials/frontend/custom-checkout/add-address-field),
[price adjustments](https://developer.adobe.com/commerce/php/development/components/price-adjustments).
Checked by running `DataObjectHelper::populateWithArray()` on an order and reading the quote-address custom attribute
list on a 2.4.9 install (November 2026).
Core code: `module-sales/etc/sales.xsd`, `module-sales/Model/Config/Ordered.php`,
`module-{quote,sales-rule,tax}/etc/sales.xml`, `module-quote/Model/Quote/TotalsCollector.php`,
`module-quote/Model/Quote/Address/Total.php`, `module-quote/Model/Quote/Address/Total/{AbstractTotal,Grand,Shipping}.php`,
`module-quote/Model/Quote/TotalsReader.php`, `module-quote/Model/Cart/{CartTotalRepository,TotalsConverter}.php`,
`module-quote-graph-ql/Model/Resolver/CartPrices.php`, `module-quote/Model/Quote/Address/ToOrder.php`,
`framework/Api/DataObjectHelper.php`, `framework/DataObject/Copy.php`, `module-quote/Model/QuoteManagement.php`,
`module-sales/Model/Order/Invoice/Total/{Grand,Shipping}.php`, `module-sales/Model/Order/Creditmemo/Total/Shipping.php`,
`module-tax/Model/Sales/Total/Quote/{Tax,CommonTaxCollector}.php`, `module-weee/etc/sales.xml`,
`module-multishipping/Model/Checkout/Type/Multishipping.php`, `module-shipping/Model/Rate/{Result,CarrierResult}.php`, `module-sales/Block/Order/Totals.php`,
`module-checkout/view/frontend/web/js/view/summary/abstract-total.js`,
`module-shipping/Model/Carrier/{AbstractCarrier,AbstractCarrierOnline}.php`, `module-shipping/Model/Shipping.php`,
`module-shipping/Model/Config/Source/Allmethods.php`, `module-quote/Model/Quote/Address.php`,
`module-quote/Model/Quote/ShippingAssignment/ShippingProcessor.php`, `module-sales/Model/Order.php`,
`module-quote/Model/{Quote,QuoteRepository,MaskedQuoteIdToQuoteId}.php`,
`module-quote-graph-ql/Model/Cart/GetCartForUser.php`, `module-checkout/etc/frontend/di.xml`,
`module-quote/Model/Quote/Address/CustomAttributeList.php`, `framework/Model/AbstractExtensibleModel.php`.
