# Frontend: templates, layout, JS, themes, i18n (Magento 2.4.x)

Sections up to "Translations" cover the Luma/Blank stack (RequireJS, Knockout, LESS); a Hyvä storefront follows
[Hyvä](#hyvä-storefronts) at the end, a headless one its own frontend's docs. Full Page Cache rules (`cacheable="false"`, `sections.xml` with `*`,
`$_isScopePrivate`, `createBlock()` in templates, view-model cache tags) are in [magento-perf-check](https://github.com/SilinMykola/magento-perf-check/blob/main/docs/SPEC.md#4-rule-catalog), not here.
Escaping and CSP: [security.md](security.md). Admin ACL for controllers, `system.xml` and UI components:
[security.md](security.md#admin-acl-in-code).

## Templates and view models

- Put template logic in a view model, not in a custom Block subclass or a helper ("the use of helpers in templates
  is discouraged"). A view model implements `Magento\Framework\View\Element\Block\ArgumentInterface`, is attached with
  `<argument name="view_model" xsi:type="object">Vendor\Module\ViewModel\X</argument>` on `<block>` or
  `<referenceBlock>`, and read with `$block->getViewModel()`. Every `xsi:type="object"` layout argument must implement
  `ArgumentInterface`; otherwise the interpreter throws `UnexpectedValueException` ("Instance of ... is expected").
- Do not create objects in templates (Technical Guidelines 6.2.6); pass them from the view model.

## Layout XML

- Extend, never copy: a file with the same handle name holding only your changes
  (`view/<area>/layout/<handle>.xml` in a module, `<theme>/<Vendor_Module>/layout/<handle>.xml` in a theme) is merged
  with the original. Override (`layout/override/base/`, `layout/override/theme/<Vendor>/<parent>/`) only for large
  changes, and never rename blocks or aliases or change the parent handle in an override.
- Change layout before overriding a `.phtml`. `remove="true"` deletes the element (a container's children go too);
  `display="false"` only skips rendering, the block objects are still created. `<remove src>` works only for `<head>`
  assets. `<move>` runs before removal and is silently skipped when the element is missing. Name every block;
  names are unique per page.

## Customer private content (correctness)

- Per-customer data reaches a cached page through a customer-data section: a class implementing
  `Magento\Customer\CustomerData\SectionSourceInterface`, registered in `etc/frontend/di.xml` on
  `Magento\Customer\CustomerData\SectionPoolInterface` → `sectionSourceMap`. The server renders a placeholder; JS fills
  it from `customerData.get('<section>')` (a Knockout observable, kept in the visitor's localStorage).
- `etc/frontend/sections.xml` invalidation fires only for jQuery AJAX and form submits with method POST, PUT or DELETE
  (`ajaxComplete` and `submit` listeners in `customer-data.js`). `fetch()`, native XHR and GET never invalidate: call
  `customerData.invalidate([...])` or `customerData.reload([...], true)` yourself. Same on 2.4.6.

## JavaScript

- Never copy or edit a core JS file. Change methods with a mixin under `config.mixins` in the area's
  `requirejs-config.js`: a jQuery widget mixin returns the widget under its parent alias, a UI component mixin returns
  `target.extend(mixin)`, a function mixin returns a wrapper. Replace a whole module with `map: {'*': {...}}` only
  when a mixin cannot do the job. To replace another module's mixin, set it to `false`, yours to `true`, and add that
  module to your `<sequence>`. `deps` load on every page.
- Initialize components declaratively, never with inline `<script>`: `data-mage-init` for one element,
  `<script type="text/x-magento-init">` for a selector or `*`.
- Checkout components change through `jsLayout` (the `jsLayout` argument in `checkout_index_index.xml`).
  `componentDisabled: true` loads the component but does not render it; to remove one, `unset` it in a
  `Magento\Checkout\Block\Checkout\LayoutProcessorInterface` class added to the `layoutProcessors` argument of
  `Magento\Checkout\Block\Onepage` (frontend `di.xml`). When moving a component, disable or remove the original.
- After changing `requirejs-config.js`: clean the cache and regenerate static files.

## Themes and LESS

- Never modify Luma or Blank: create a theme with `<parent>` in `theme.xml`.
- Add styles in `_extend.less`. A child theme's `_theme.less` replaces the parent's file completely (copy all its
  variables, then keep them in sync by hand). Module styles go in `view/frontend/web/css/source/_module.less`; Blank-based
  themes include them through `//@magento_import`.

## Translations (i18n)

- Translate literal strings with placeholders: `__('Hello %1', $name)`. Never concatenate or pass a variable as the
  phrase; `i18n:collect-phrases` cannot read it. JS: `$t()` or `$.mage.__()` (dependency `mage/translate`), then
  `.replace('%1', v)`; Knockout templates: `i18n:` binding or `translate` attribute; Underscore templates: `_.i18n()`;
  email templates: `{{trans}}`; UI component XML: `translate="true"`.
- Dictionaries: `i18n/<locale>.csv` in the module or theme; the file name matches the locale exactly, case included.
  Generate with `bin/magento i18n:collect-phrases -o <csv> <dir>` (`-m` only for language packs). A translation keeps
  every placeholder and adds none.

## Hyvä storefronts

Hyvä Theme is OSL-3.0/AFL-3.0 since 2025-11-10 (Hyvä Checkout, Enterprise and Hyvä UI stay paid).
- Detect it: the theme's parent chain ends in `Hyva/default` or `Hyva/default-csp`, or Composer has
  `hyva-themes/magento2-theme-module`. Then RequireJS, Knockout, LESS, mixins, `data-mage-init`/`x-magento-init`,
  `customer-data.js` and `jsLayout` above do not apply; layout XML, view models, `.phtml`, i18n and the server side of
  private content (`SectionSourceInterface` + `sectionSourceMap`; Hyvä loads sections from the same endpoint) do.
- JS is inline Alpine.js in the `.phtml`; register components with `Alpine.data()` inside `alpine:init`. Styles are
  Tailwind classes.
- Private content: listen for `private-content-loaded` on `window` and read `$event.detail.data`; force a reload with
  `window.dispatchEvent(new Event('reload-customer-section-data'))`. Never read customer data from localStorage.
- View models in templates: `$viewModels->require(Foo::class)`, no layout argument needed.
- CSP: call `$hyvaCsp->registerInlineScript()` right after every inline `</script>` (hash on cacheable pages, nonce
  elsewhere). The strict-CSP theme `Hyva/default-csp` uses the Alpine CSP build: no inline expressions, assignments,
  calls with arguments, `x-model` or inline `on*` handlers — use component methods and `data-*` attributes.
- A third-party Luma module needs a compatibility module, registered on
  `Hyva\CompatModuleFallback\Model\CompatModuleRegistry` (argument `compatModules`, items `original_module` /
  `compat_module`); templates at the same path then override the original.
- Tailwind builds only classes it finds in scanned files: modules are listed in `app/etc/hyva-themes.json`
  (`bin/magento hyva:config:generate` before the CSS build); a module's own sources go in
  `view/frontend/tailwind/tailwind-source.css` (Tailwind v4, default theme 1.4.0+) or `tailwind.config.js` (v3).
- Without Hyvä Checkout, `hyva-themes/magento2-luma-checkout` renders checkout with Luma: there RequireJS, Knockout,
  mixins and `jsLayout` apply, Alpine and Tailwind do not. A checkout extension needs Luma code for that fallback, or
  separate Hyvä Checkout code.

Hyvä sources (opened November 2026): [open-source announcement](https://hyva.io/blog/news/hyva-is-free-open-source.html),
[technical vision](https://docs.hyva.io/welcome/technical-vision.html),
[migrating JS](https://docs.hyva.io/hyva-themes/compatibility-modules/from-luma-to-hyva/migrating-js-and-templates.html),
[section data](https://docs.hyva.io/hyva-themes/writing-code/working-with-sectiondata.html),
[view models](https://docs.hyva.io/hyva-themes/writing-code/working-with-view-models/index.html),
[CSP](https://docs.hyva.io/hyva-themes/writing-code/csp/csp-compatibility.html),
[Alpine CSP](https://docs.hyva.io/hyva-themes/writing-code/csp/alpine-csp.html),
[compatibility modules](https://docs.hyva.io/hyva-themes/compatibility-modules/technical-deep-dive.html),
[Tailwind sources](https://docs.hyva.io/hyva-themes/working-with-tailwindcss/tailwind-purging-settings.html),
[Luma checkout fallback](https://docs.hyva.io/hyva-themes/building-your-theme/luma-theme-fallback.html);
code in GitHub `hyva-themes/magento2-theme-module` 1.5.3 (`src/Model/ViewModelRegistry.php`,
`src/ViewModel/HyvaCsp.php`, `src/view/frontend/templates/page/js/private-content.phtml`),
`hyva-themes/magento2-compat-module-fallback`, `hyva-themes/magento2-default-theme` (`composer.json`, CHANGELOG).

## Sources

Adobe docs (opened November 2026): [view models](https://developer.adobe.com/commerce/php/development/components/view-models),
[Technical Guidelines](https://developer.adobe.com/commerce/php/coding-standards/technical-guidelines),
[extend layouts](https://developer.adobe.com/commerce/frontend-core/guide/layouts/extend),
[override layouts](https://developer.adobe.com/commerce/frontend-core/guide/layouts/override),
[layout instructions](https://developer.adobe.com/commerce/frontend-core/guide/layouts/xml-instructions),
[private content](https://developer.adobe.com/commerce/php/development/cache/page/private-content),
[JS mixins](https://developer.adobe.com/commerce/frontend-core/javascript/mixins),
[JS init](https://developer.adobe.com/commerce/frontend-core/javascript/init),
[checkout view](https://developer.adobe.com/commerce/php/tutorials/frontend/custom-checkout/customize-view),
[create a theme](https://developer.adobe.com/commerce/frontend-core/guide/themes/create-storefront),
[customize styles](https://developer.adobe.com/commerce/frontend-core/guide/css/quickstart/customize-styles),
[translations](https://developer.adobe.com/commerce/frontend-core/guide/translations/theory),
[localization CLI](https://experienceleague.adobe.com/en/docs/commerce-operations/configuration-guide/cli/localization).
Core code, magento/magento2 2.4.9: `app/etc/di.xml` (`layoutObjectArgumentInterpreter`),
`lib/internal/Magento/Framework/View/Layout/Argument/Interpreter/DataObject.php`,
`app/code/Magento/Customer/view/frontend/web/js/customer-data.js`, `app/code/Magento/Checkout/etc/frontend/di.xml`.
