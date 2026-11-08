# Cart and catalog price rules (Magento 2.4.x)

Code paths are in magento/magento2 at tag 2.4.9 (`vendor/magento/*` of a 2.4.9 install); behaviour on 2.4.6–2.4.8 was
not checked unless a line says so. Totals collectors and fees: [checkout-quote.md](checkout-quote.md).

## Custom conditions

- Conditions of the rule root are validated against the quote address (`$rule->validate($address)`), action
  conditions against each item (`$rule->getActions()->validate($item)`). For the root tree, add a condition type in an
  observer on `salesrule_rule_condition_combine`: call `setConditions([...])` on the `additional` object; core merges
  it into its own list. The action tree (`Condition\Product\Combine`, Found, Subselect) dispatches no event: add to it
  with a plugin on its `getNewChildSelectOptions()`. A condition offered in both trees must handle both models.
- A product attribute is offered in rule conditions only with `is_used_for_promo_rules` = 1, `is_visible` = 1 and an
  allowed input type (boolean, date, datetime, multiselect, price, select, text, textarea, weight).
- Product attributes reach quote items only if they are registered. On rule save core extracts attribute codes from
  the serialized conditions with a regex that matches only the exact type `Magento\SalesRule\Model\Rule\Condition\Product`
  and stores them in `salesrule_product_attribute`; a plugin adds those to the `quote_item` product attributes. A
  condition class that extends `Condition\Product`, or a rule written by SQL, is not registered and validates against an
  empty value. Add such attributes to group `quote_item` in your `etc/catalog_attributes.xml`.
- New in 2.4.9 (absent at tags 2.4.6 and 2.4.8): when a rule has item-level action conditions and at least one item
  matches them, its address conditions (subtotal, total qty, weight) are validated against the totals of those items
  only; core swaps the address totals before `validate()` and restores them after. A custom address condition reads those reduced totals.

## Rule order and discounts

- Rules run in `sort_order` ascending with no tie-breaker: give rules that interact distinct sort orders. "Discard
  subsequent rules" removes only the items that rule applied to; other items still get later rules (same at 2.4.6 and
  2.4.8).
- For shipping, rules without "Apply to Shipping Amount" are skipped before their "Discard subsequent rules" flag is
  checked, so a later rule can still discount shipping after an earlier discard rule that does not apply to shipping.
- The `discounts` extension attribute (cart item, quote address) is filled only while totals are collected and is not
  stored: a freshly loaded quote has none until `collectTotals()`, and the order keeps only `applied_rule_ids`,
  `discount_description` and `coupon_rule_name`, without a per-rule breakdown. GraphQL exposes it as `Cart.prices.discounts`.
- Disabling or deleting a rule does not recollect quotes at once: the `sales.rule.quote.trigger.recollect` consumer
  sets `trigger_recollect` = 1 on active quotes that have the rule applied, and each is recollected when next loaded.

## Coupons

- Usage limits: on 2.4.9 coupon `times_used` and per-customer rule usage are updated synchronously when the order is
  placed (and reverted if placing fails); the rule's own `times_used` is updated by the `sales.rule.update.coupon.usage`
  consumer, so `$rule->getTimesUsed()` lags until it runs. On 2.4.6 all counters were updated by that consumer, so
  limits were not enforced while consumers were down.
- Coupon generation: `CouponManagementService::generate()` (REST) is synchronous and returns the codes; it requires
  auto-generation on the rule. The admin "Generate" button only publishes to `sales_rule.codegenerator`; no codes exist
  until the `codegeneratorProcessor` consumer runs.

## Catalog price rules

- There is no indexer for cart price rules. Catalog rule prices are indexed for yesterday, today and tomorrow only;
  cron `catalogrule_apply_all` (01:00) only marks the indexer invalid (when any active rule exists) and
  `indexer_reindex_all_invalid` reindexes it. Without cron, a rule with a future start date reaches prices only after a
  manual `catalogrule_rule` reindex or a rule save (same at 2.4.8).

## Sources

Core code: `module-sales-rule/Model/Rule/Condition/{Combine,Address,Product}.php`, `module-sales-rule/Model/Utility.php`,
`module-sales-rule/Model/{RulesApplier,Validator}.php`, `module-sales-rule/Model/Quote/Discount.php`,
`module-sales-rule/Model/ResourceModel/Rule.php`, `module-sales-rule/Model/ResourceModel/Rule/Collection.php`,
`module-sales-rule/Model/Plugin/QuoteConfigProductAttributes.php`, `module-catalog/Model/ResourceModel/Eav/Attribute.php`,
`module-sales-rule/Model/Coupon/Quote/UpdateCouponUsages.php`, `module-sales-rule/Plugin/CouponUsagesIncrement.php`,
`module-sales-rule/Model/CouponUsageConsumer.php`, `module-sales-rule/Model/Service/CouponManagementService.php`,
`module-sales-rule/Controller/Adminhtml/Promo/Quote/Generate.php`, `module-sales-rule/etc/{di,queue_consumer,extension_attributes}.xml`,
`module-sales-rule/Observer/RuleQuoteRecollectTotalsObserver.php`, `module-sales-rule/Model/Rule/RuleQuoteRecollectTotalsOnDemand.php`, `module-quote-graph-ql/etc/schema.graphqls`,
`module-catalog-rule/Cron/DailyCatalogUpdate.php`, `module-catalog-rule/Model/Indexer/ReindexRuleProductsPriceProcessor.php`,
`module-indexer/etc/crontab.xml`. Older tags compared on GitHub (magento/magento2, tags 2.4.6 and 2.4.8, November 2026).
