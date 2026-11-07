# Payment methods (Magento 2.4.x)

Code paths are in magento/magento2 at tag 2.4.9; the logic is the same on 2.4.6–2.4.8 unless a line says otherwise.
Money-path principles: SKILL.md non-negotiable 9. CSP/SRI on payment pages and encrypted config: [security.md](security.md).

## Gateway structure

- Build the method on a `Magento\Payment\Model\Method\Adapter` virtualType (arguments `code`, `formBlockType`,
  `infoBlockType`, `valueHandlerPool` with a `default` handler, `validatorPool`, `commandPool`). Never extend
  `Magento\Payment\Model\Method\AbstractMethod`: `@deprecated 100.0.6`, `@see Adapter`.
- Every operation needs a command in the pool AND `can_<operation>` = 1 in the method's `config.xml` (`can_authorize`,
  `can_capture`, `can_refund`, `can_void`, `can_cancel`, `can_sale`, `can_accept_payment`, `can_deny_payment`, ...).
  Without the flag `Adapter` returns null and the operation is skipped silently. Order cancel voids online only with
  `can_void` (and `can_cancel`); the admin Accept/Deny buttons need `can_review_payment`.
- Give every `GatewayCommand` a response validator: the handler runs only when the validator passes, and with no
  validator it also runs on declined responses. Map gateway error codes through `errorMessageMapper`, otherwise the
  customer sees "Transaction has been declined. Please try again later."
- Secrets read through `Magento\Payment\Gateway\Config\Config` come back decrypted only when the `config.xml` node
  declares the `Encrypted` backend model (rules in [backend-components.md](backend-components.md#configuration)).

## Payment action and order state

- `payment_action` `authorize` calls `authorize` with the base total due and records an auth transaction;
  `authorize_capture` calls `sale` when `can_sale` is set, otherwise `capture`, and core creates the invoice. Never
  create an invoice in your handler or an observer as well.
- Never set the order state from gateway code; `Payment::place()` and the state commands decide it. To hold an order
  for review, `setIsTransactionPending(true)` (order goes to `payment_review`). Redirect/hosted methods use
  `can_initialize` = 1 and an `initialize` command that fills `stateObject` (`state`, `status`, `is_notified`).
- In the response handler, `setTransactionId()` with the gateway's own ID and set `setIsTransactionClosed()` /
  `setShouldCloseParentTransaction()` deliberately. Without an ID Magento invents `<parent>-capture`; online refund
  needs the invoice's capture transaction ID, otherwise it fails with "If the invoice was created offline, try
  creating an offline credit memo."
- The `amount` argument is in base currency (base total due, invoice or credit memo `base_grand_total`). A partial
  invoice captures only its amount. Online refund runs only with `can_refund` and a credit memo that has
  `do_transaction` and an invoice.

## Logging

- Pass the method's config to `Magento\Payment\Model\Method\Logger` through a virtualType (`config` argument);
  without it `debug` and `debugReplaceKeys` are never read, so nothing is logged or masked. Masking walks array keys;
  a JSON string body is not masked, so log the request as an array before encoding it. Output: `var/log/payment.log`.

## Vault

- Store only the gateway token: build it in a response handler (`setGatewayToken`, `setExpiresAt`, `setTokenDetails`
  as JSON with `type`, `maskedCC`, `expirationDate`) and attach it with `setVaultPaymentToken`. Core
  `AfterPaymentSaveObserver` fills `customer_id`, `is_active`, `public_hash`, `is_visible` and (2.4.7+) `website_id`.
  Never store the card number or CVV. Token deletion is soft (`is_active` = 0, `is_visible` = 0).
- Accept only `public_hash` from the client and resolve it with `PaymentTokenManagement::getByPublicHash($hash,
  $customerId)`; never accept a gateway token or `entity_id`.

## Headless and notifications

- GraphQL: core `PaymentMethodInput` has only `code` and `purchase_order_number`. Add a field named after your method
  code, register a provider under that code in `Magento\QuoteGraphQl\Model\Cart\Payment\AdditionalDataProviderPool`
  (`dataProviders`, `etc/graphql/di.xml`), and in the `payment_method_assign_data_<code>` observer copy only known keys
  into `additional_information`.
- Webhook/IPN controllers verify the notification before loading the order (core PayPal IPN posts back first and
  accepts POST only) and record captures with `setTransactionId(<gateway id>)` + `registerCaptureNotification($amount)`
  instead of building invoices by hand; core finds the existing invoice by transaction ID, so a repeated notification
  does not create a second one. CSRF exemption for the controller: [security.md](security.md).

## Sources

Adobe docs (opened November 2026), under `https://developer.adobe.com/commerce/php/development/payments-integrations/`:
[payment-gateway](https://developer.adobe.com/commerce/php/development/payments-integrations/payment-gateway),
[gateway-command](https://developer.adobe.com/commerce/php/development/payments-integrations/payment-gateway/gateway-command),
[response-handler](https://developer.adobe.com/commerce/php/development/payments-integrations/payment-gateway/response-handler),
[response-validator](https://developer.adobe.com/commerce/php/development/payments-integrations/payment-gateway/response-validator),
[facade-configuration](https://developer.adobe.com/commerce/php/development/payments-integrations/base-integration/facade-configuration),
[payment-option-config](https://developer.adobe.com/commerce/php/development/payments-integrations/base-integration/payment-option-config),
[vault](https://developer.adobe.com/commerce/php/development/payments-integrations/vault),
[payment-token](https://developer.adobe.com/commerce/php/development/payments-integrations/vault/payment-token).
Core code: `app/code/Magento/Payment/Model/Method/{Adapter,AbstractMethod,Logger}.php`,
`app/code/Magento/Payment/Gateway/Command/GatewayCommand.php`, `app/code/Magento/Payment/Gateway/Config/Config.php`,
`lib/internal/Magento/Framework/App/Config/MetadataProcessor.php`,
`app/code/Magento/Sales/Model/Order/Payment.php`, `app/code/Magento/Sales/Model/Order/Payment/State/AuthorizeCommand.php`,
`app/code/Magento/Sales/Model/Order/Payment/Transaction/Manager.php`,
`app/code/Magento/Vault/Observer/{AfterPaymentSaveObserver,PaymentTokenAssigner}.php`,
`app/code/Magento/Vault/Model/PaymentTokenRepository.php`,
`app/code/Magento/QuoteGraphQl/Model/Cart/Payment/AdditionalDataProviderPool.php`,
`app/code/Magento/Paypal/Controller/Ipn/Index.php`, `app/code/Magento/Paypal/Model/Ipn.php`,
`app/code/Magento/Sales/Model/Order/Payment/Operations/RegisterCaptureNotificationOperation.php`.
