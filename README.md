# Magento 2 Best Practices — Claude Skill

> **Fork of [o0mohd0o/magento2-best-practices-skill](https://github.com/o0mohd0o/magento2-best-practices-skill).**
> Changes vs upstream: removed leftovers of the author's own project (admin
> path, ElasticSuite/2.4.6/PHPUnit 9.5/Next.js assumptions), so the rules are
> actually generic; `SKILL.md` now starts by detecting the project's real
> Magento/PHP/PHPUnit versions; `release-matrix.md` is marked as a dated
> snapshot to re-check against Adobe release notes.

A [Claude Code](https://claude.com/claude-code) **Skill** that teaches Claude how to write, review, and plan Magento 2 / Adobe Commerce code the right way — targeting **Magento 2.4.9** practices while staying safe on 2.4.6–2.4.8.

It bundles hard-won, version-verified rules for the areas people get wrong most: DI and extension mechanisms, GraphQL/REST APIs, declarative schema, security, performance/ops, and testing. When it's active, Claude follows these rules instead of guessing from stale training data.

## What's in it

```
magento2-best-practices/
├── SKILL.md                    # entry point: non-negotiables, decision guides, workflows
└── references/                 # loaded on demand, only when relevant
    ├── release-matrix.md       # 2.4.9/2.4.8/2.4.6 platform matrix, deprecations, upgrade path
    ├── module-development.md   # DI, plugins vs observers, declarative schema & patches, packaging
    ├── graphql-api.md          # resolver patterns, batch resolvers, caching, auth, REST/webapi, async
    ├── performance-ops.md      # modes, deploy pipeline, cron/queues, indexers, Redis/Varnish
    ├── security.md             # patch cadence, admin hardening, CSP, code-level rules
    └── quality-testing.md      # unit/integration/api-functional tests, PHPCS/PHPStan, CI, semver
```

`SKILL.md` is the always-loaded core. The `references/` files are pulled in progressively — Claude reads `graphql-api.md` only when you're actually touching GraphQL, keeping context lean.

## Install

Skills live under `~/.claude/skills/` (personal, all projects) or `<project>/.claude/skills/` (shared with a repo via git).

### Option A — clone into your personal skills

```bash
git clone https://github.com/o0mohd0o/magento2-best-practices-skill.git /tmp/mag-skill
mkdir -p ~/.claude/skills
cp -r /tmp/mag-skill/magento2-best-practices ~/.claude/skills/
rm -rf /tmp/mag-skill
```

### Option B — install into a specific project (shareable with your team)

```bash
git clone https://github.com/o0mohd0o/magento2-best-practices-skill.git /tmp/mag-skill
mkdir -p /path/to/your/project/.claude/skills
cp -r /tmp/mag-skill/magento2-best-practices /path/to/your/project/.claude/skills/
rm -rf /tmp/mag-skill
# commit .claude/skills/magento2-best-practices so the whole team gets it
```

The only requirement is that the folder containing `SKILL.md` sits directly under a `skills/` directory. Verify it's picked up by running `/skills` inside Claude Code — `magento2-best-practices` should appear in the list.

## How to use it

Once installed, **you don't invoke it manually** — Claude auto-activates it based on the `description` in `SKILL.md` whenever your request looks like Magento work. Triggers include:

- "Build a Magento module that …"
- "Add a GraphQL mutation / query for …"
- "Fix this Magento bug" / "Review this Magento code"
- "Change db_schema" / "add a plugin / observer"
- "Plan the upgrade from 2.4.6"

You can also force it explicitly:

```
Use the magento2-best-practices skill and review app/code/Vendor/Module for issues.
```

### Example prompts

```
Build a Magento 2 module that adds a "gift note" to the checkout and persists it on the order. Follow best practices.

Review this plugin for correctness and see if an observer would be more appropriate.

I'm on 2.4.6 — what breaks if I upgrade to 2.4.9, and what's the safe path?

Add a GraphQL query to return a customer's loyalty balance, with proper auth and caching.
```

When the skill is active you'll notice Claude reaching for the narrowest extension mechanism, refusing to edit `vendor/`, using declarative schema, escaping template output, and checking `var/log/exception.log` after exercising a flow — because those are the rules it's now following.

## Customizing for your own project

This is the **generic** version. The recommended pattern is to add a `references/project-conventions.md` of your own with your codebase's specifics (module namespace, frozen contracts, stack reality, hard-won regressions) and a line near the top of `SKILL.md` telling Claude to read it and treat it as authoritative:

```markdown
When working in `~/your-project`, ALSO read
[project-conventions.md](references/project-conventions.md) — its rules win.
```

Keep anything secret or client-confidential in a **private** copy — never commit credentials, real hostnames, or business internals to a public skill.

## Compatibility

- **Claude Code** (CLI, desktop, IDE extensions) — native Skills support.
- Targets **Magento 2.4.9** best practices; annotated for 2.4.6/2.4.7/2.4.8 differences in `release-matrix.md`.
- Facts verified as of **2026-07-06**. Magento's cadence is monthly security patches + an annual May feature release, so re-check `release-matrix.md` against Adobe's release notes over time.

## License

MIT — see [LICENSE](LICENSE).
