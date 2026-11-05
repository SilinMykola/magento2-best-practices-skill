# AGENTS.md

Instructions for anyone changing this repository, people and coding agents alike.

- Fork of o0mohd0o/magento2-best-practices-skill; README lists what differs from upstream. Keep that list current.
- Rules are version-sensitive: say which Magento/PHP/PHPUnit version a rule applies to, and check it against Adobe
  release notes or core code (`composer.lock` of the target version) before writing it.
- `references/quality-testing.md` is the single home of test design knowledge for the related repositories
  (magento2-ddev and magento2-core-contrib link to it).
- The skill stays environment-neutral; DDEV specifics belong in magento2-ddev.
