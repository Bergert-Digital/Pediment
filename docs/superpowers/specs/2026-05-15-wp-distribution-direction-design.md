# WP Starter Ecosystem — Distribution Direction

**Date:** 2026-05-15
**Status:** Approved design, pending implementation plan
**Repos touched:** `wp-starter-ai`, `wp-starter-theme`, `wp-client-template`, new `wp-starter-child-theme`

## Background

The WP starter ecosystem has three code units: `wp-starter-ai` (plugin),
`wp-starter-theme` (parent block theme), and `wp-client-template` (a
Bedrock/Composer/wp-env scaffold that composed all of them into one site).

`wp-client-template`'s Bedrock layout served an agency dev/deploy workflow but
is not installable on a standard WordPress host. The product value lives in the
plugin and the parent theme. Decision (2026-05-14, confirmed 2026-05-15): move
to a distribution model where any WP admin can install the pieces via zip
upload, and agencies start from a dedicated forkable child theme.

The fork-child-theme + install-zips model **fully replaces** the
Bedrock+Composer assembly model for site setup. `wp-client-template` has no
residual internal role and is retired.

## Goal

1. A new `wp-starter-child-theme` repo as the agency starting point.
2. `wp-client-template` decisively retired (deprecated + archived).
3. Reproducible installable zips for plugin, parent theme, and child theme.

Workstream A (a `wp-starter-ai` `Bootstrap.php` copy-paste bug) was an
independent prerequisite and is **already fixed** — commit `b7904ad` on
`wp-starter-ai/development`, suite 95/95 green. It is out of scope for this
document and recorded here only for traceability.

## Out of scope

- WP.org theme-directory submission or a custom update server. The zip pipeline
  is the near-term distribution mechanism; the hosted-update path is a future,
  separately-specced effort.
- Render-time hooks on the parent theme (`wp_head`, asset enqueue, Brand
  getters) flagged in the prior audit — a future gap, not addressed here.
- Stripping `client-theme/` out of `wp-client-template` — explicitly declined;
  it stays in place in the archived repo.

---

## Workstream B — `wp-starter-child-theme` repo

### Identity

| Field | Value |
|---|---|
| GitHub repo | `github.com/Bergert-Digital/wp-starter-child-theme` |
| Theme Name | `Starter Child Theme` |
| Template (parent) | `wp-starter-theme` |
| Text Domain | `starter-child` |
| npm package name | `wp-starter-child-theme` |
| Version | `0.1.0` |
| Branches | `main` (released) + `development` (integration), matching siblings |

### Content seed

Fresh git history, single initial commit. Seeded from a **snapshot** of
`wp-client-template/web/app/themes/client-theme/`:

- `style.css` — rewritten header with the identity above (`Template:
  wp-starter-theme`, `Text Domain: starter-child`).
- `functions.php` — child bootstrap: block auto-registration from
  `build/blocks/`, child stylesheet enqueue. Comments/`@package` updated to
  `StarterChild`.
- `theme.json`, `tsconfig.json`, `assets/fonts/.gitkeep`.
- `package.json` — name `wp-starter-child-theme`, existing `wp-scripts`
  build/start scripts retained.
- `src/blocks/promo-banner/` — **kept** as the worked block example
  (`block.json`, `edit.tsx`, `index.tsx`, `render.php`, `style.scss`).

### Test harness (from `wp-starter-theme`)

Bring over and adapt to the child theme's namespace/paths:

- `phpcs.xml.dist` — PHP linting.
- `phpunit.xml.dist` + `tests/` bootstrap structure — PHP unit tests.
- `playwright.config.ts` + E2E scaffolding.
- A dev-only `composer.json` (require-dev: phpcs/wpcs, phpunit, polyfills) with
  autoload paths matching the child theme.

Rationale: agencies are TDD-ready on fork and get linting parity with the
parent. This is heavier than a bare starter, chosen deliberately.

### Dev environment

`.wp-env.json` maps the parent theme and plugin from **sibling local
checkouts** (`../wp-starter-theme`, `../wp-starter-ai`) — the standard
side-by-side clone layout. Fast iteration against unreleased parent changes; no
version pinning. (To develop the child you must have the parent present and
activated; the sibling-mapping makes that automatic for the maintainer.)

### README

- What it is: the agency starting point; fork or download-as-zip, customize
  freely, push to your own git for per-client install.
- Install order on a fresh WP (no automatic theme-dependency resolution in WP):
  **parent theme zip → child theme zip → activate child**. Plugin zip any time.
- First-fork rename checklist: grep-replace `wp-starter-child-theme`,
  `Starter Child Theme`, `starter-child`, `StarterChild` with client values.
- "Replace or delete `promo-banner` before shipping to a client."

---

## Workstream C — Retire `wp-client-template`

1. One final deprecation commit on `wp-client-template`'s `development` branch
   prepending a banner to `README.md`, then fast-forward `main` to it so the
   archived default branch shows the banner.
   - Banner states the repo is retired, links the three product repos
     (`wp-starter-ai`, `wp-starter-theme`, `wp-starter-child-theme`), and gives
     the parent→child→plugin install order.
   - `client-theme/` is **left in place** (declined to strip).
2. Archive on GitHub via `gh repo archive Bergert-Digital/wp-client-template`
   — read-only, history preserved, reversible.

No code deletion. The Bedrock scaffold remains cloneable for historical
reference only.

---

## Workstream D — Zip pipeline (all three repos)

One GitHub Actions workflow per repo, triggered on `v*` tag push, producing a
GitHub Release with an installable/forkable zip attached.

### Common workflow shape

```
on: push: tags: ['v*']
job:
  - checkout
  - setup-node, setup-php
  - npm ci && npm run build
  - (wp-starter-ai only) composer install --no-dev --optimize-autoloader
  - assemble clean zip honoring a committed .distignore
  - softprops/action-gh-release with the zip asset
```

### Per-repo zip contents

| Repo | Zip contains | Excludes |
|---|---|---|
| `wp-starter-ai` (plugin) | `plugin.php`, `src/`, `build/`, runtime `vendor/`, `uninstall.php`, `wp-cli/` | `node_modules`, `tests`, `editor/` TS sources, dev composer deps, `.git`, CI, `*-report` |
| `wp-starter-theme` (parent) | `style.css`, `theme.json`, `functions.php`, `inc/`, `parts/`, `patterns/`, `templates/`, `build/`, `assets/` | `node_modules`, `tests`, `src/` TS, `tools/`, dev deps, `.git`, CI, `*-report` |
| `wp-starter-child-theme` | **Forkable**: built assets **+** `src/` + `theme.json` + config (mirrors the repo) | `node_modules`, `.git`, `*-report`, test-results |

Each repo gets a committed `.distignore` so the exclude list is reviewable and
versioned, not buried in YAML.

The child theme's zip is intentionally a near-mirror of the repo:
"download-as-zip" must equal "git clone" for agencies who don't use git, since
the child is a starter to be forked, not a black-box install.

---

## Remote-action protocol

Creating the GitHub repo and archiving `wp-client-template` are shared-state
actions. The implementer prepares everything locally, then runs `gh repo
create`, the initial `git push`, and `gh repo archive` itself — but **pauses
and shows the exact command for explicit go-ahead immediately before each
remote-affecting call**. No silent remote mutations.

## Execution order

1. **B** — build `wp-starter-child-theme` locally, create remote (paused
   confirm), push.
2. **C** — deprecation commit on `wp-client-template`; archive (paused
   confirm). After B so the banner links a live repo.
3. **D** — add the release workflow + `.distignore` to all three product repos
   (`wp-starter-ai`, `wp-starter-theme`, `wp-starter-child-theme`).

## Testing / verification

- Child theme: `composer lint`, phpunit, and a build (`npm run build`) must
  pass in the new repo before first push.
- Zip pipeline: validate by pushing a throwaway pre-release tag (e.g.
  `v0.0.0-test`) on one repo, confirming the Release zip installs cleanly into
  a fresh WP via Appearance/Plugins → Add New → Upload, then deleting the test
  release/tag. Repeat per repo.
- Retirement: confirm the archived `wp-client-template` README banner renders
  and links resolve before archiving (archive is reversible if not).
